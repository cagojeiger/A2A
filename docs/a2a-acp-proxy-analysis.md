# A2A ↔ ACP 프록시 — 개념 타당성 분석

## 1. 두 프로토콜 요약

| | ACP (Agent-Client Protocol) | A2A (Agent-to-Agent) |
|---|---|---|
| **만든 곳** | Zed Industries | Google → Linux Foundation |
| **역할** | 코드 에디터 ↔ AI 코딩 에이전트 | AI 에이전트 ↔ AI 에이전트 |
| **핵심** | 세션, 프롬프트, 파일/터미널 접근 | 태스크, 에이전트 발견, 스킬, 아티팩트 |
| **전송** | stdio (NDJSON) | HTTP (JSON-RPC / gRPC / REST) |
| **방향성** | 양방향 (에이전트가 클라이언트에 파일/터미널 요청) | 단방향 요청-응답 (클라이언트→서버) |

---

## 2. 프록시 개념

### 핵심 아이디어
```
┌────────────────┐         ┌──────────────────┐         ┌─────────────────┐
│  A2A 클라이언트  │  A2A    │   A2A↔ACP 프록시   │  ACP    │  ACP 코딩 에이전트 │
│  (오케스트레이터  │ ──────→ │                    │ ──────→ │  (Claude Code,   │
│   다른 에이전트)  │ HTTP    │  A2A Server 역할   │ stdio   │   Copilot 등)    │
│                │ ←────── │  ACP Client 역할   │ ←────── │                 │
└────────────────┘         └──────────────────┘         └─────────────────┘
```

**프록시는 두 개의 얼굴을 가짐:**
- **A2A 측**: HTTP 서버로서 `AgentCard`를 노출하고 `SendMessage`를 받아 처리
- **ACP 측**: stdio 클라이언트로서 코딩 에이전트 프로세스를 spawn하고 `session/prompt`로 작업 전달

### 이것이 가능하게 하는 것
> **어떤 A2A 에이전트든 ACP 호환 코딩 에이전트에게 코드 작업을 위임할 수 있다**

예: PM 에이전트가 "이 버그를 고쳐줘" → 프록시 → Claude Code가 실제로 코드 수정

---

## 3. 개념 타당성 분석

### 3.1 잘 맞는 부분 (타당한 이유)

#### (a) 메시지 모델 호환성 — 높음
```
A2A Part(text) ←→ ACP ContentBlock(text)     ✅ 직접 매핑
A2A Part(raw/url) ←→ ACP ContentBlock(image)  ✅ MIME 기반 매핑
A2A Part(data) ←→ ACP ContentBlock(resource)  ⚠️ 변환 필요하지만 가능
```
양쪽 모두 "텍스트 + 바이너리 + 구조화 데이터"라는 동일한 컨텐츠 패러다임을 사용.

#### (b) 작업 흐름 매핑 — 자연스러움
```
A2A SendMessage              → ACP session/prompt
A2A Task(WORKING)            ← ACP sessionUpdate (스트리밍 중)
A2A Task(COMPLETED)          ← ACP PromptResponse(end_turn)
A2A Task(FAILED)             ← ACP PromptResponse(refusal) 또는 에러
A2A Task(INPUT_REQUIRED)     ← ACP session/request_permission
A2A CancelTask               → ACP session/cancel
A2A TaskArtifactUpdateEvent  ← ACP sessionUpdate(tool_call: edit/write)
```
A2A의 Task 상태 머신이 ACP의 프롬프트 라이프사이클과 자연스럽게 대응됨.

#### (c) 스트리밍 호환 — 양쪽 모두 지원
```
A2A: SendStreamingMessage → SSE stream (TaskStatusUpdate, TaskArtifactUpdate)
ACP: session/prompt 후 sessionUpdate 노티피케이션 연속 수신
```
프록시가 ACP의 `sessionUpdate`를 받을 때마다 A2A의 `StreamResponse`로 변환하여 전송하면 됨.

#### (d) 에이전트 발견 — 새로운 가치
ACP 코딩 에이전트는 현재 발견 메커니즘이 없음. A2A `AgentCard`를 통해 코딩 에이전트의 능력을 노출하면:
```json
{
  "name": "Claude Code via ACP",
  "skills": [
    { "id": "code-edit", "name": "Code Editing", "tags": ["coding", "refactor", "debug"] },
    { "id": "code-review", "name": "Code Review", "tags": ["review", "quality"] },
    { "id": "terminal-exec", "name": "Command Execution", "tags": ["build", "test", "deploy"] }
  ],
  "default_input_modes": ["text/plain"],
  "default_output_modes": ["text/plain", "application/json"]
}
```
→ 다른 에이전트들이 "코딩 능력이 필요한 작업"을 자동으로 이 에이전트에 위임 가능.

### 3.2 구조적 불일치 (도전 과제)

#### (a) 전송 계층 격차 — **가장 큰 도전**
```
A2A: HTTP 기반 (클라이언트가 서버에 요청)
ACP: stdio 기반 (프로세스 간 양방향 파이프)
```
- 프록시가 ACP 에이전트를 **subprocess로 spawn**해야 함
- 에이전트 프로세스의 수명 관리가 필요 (풀링? 요청마다 새로?)
- 이것 자체는 어렵지 않으나, **프록시가 에이전트를 직접 실행해야 한다는 운영 부담**이 생김

#### (b) 파일시스템 접근 — **핵심 설계 결정**
ACP의 가장 독특한 특성: **에이전트가 클라이언트에게 파일 읽기/쓰기를 요청**함.
```
ACP Agent → "fs/read_text_file /src/main.rs 읽어줘" → ACP Client (프록시)
ACP Agent → "fs/write_text_file /src/main.rs 이렇게 바꿔줘" → ACP Client (프록시)
ACP Agent → "terminal/create 'cargo build' 실행해줘" → ACP Client (프록시)
```
**프록시가 실제 파일시스템과 터미널을 제공해야 함.**

→ 선택지:
1. **프록시가 로컬 파일시스템 직접 제공** (간단하지만 보안 위험)
2. **Git 레포 클론 + 샌드박스 환경** (안전하지만 복잡)
3. **컨테이너/VM 기반 격리** (가장 안전, 가장 복잡)

#### (c) 권한 모델 불일치
```
ACP: 에이전트 → "이 파일 수정해도 될까요?" → 사용자에게 묻기 (interactive)
A2A: 에이전트 간 소통에서는 사용자가 없음 (opaque execution)
```
→ 선택지:
1. **자동 승인** (위험하지만 빠름)
2. **A2A `INPUT_REQUIRED` 상태로 변환** → 원래 A2A 클라이언트에게 판단 위임
3. **허용 목록 기반 정책** (특정 경로/명령만 자동 허용)

#### (d) 세션 vs 태스크 수명
```
ACP: 세션 기반 — initialize → session/new → 여러 prompt → 세션 종료
A2A: 태스크 기반 — 각 SendMessage가 독립적 (context_id로 연결은 가능)
```
프록시가 A2A `context_id` ↔ ACP `sessionId` 매핑 테이블을 유지해야 함.
- 같은 context_id의 메시지 → 기존 ACP 세션에서 계속
- 새 context_id → 새 ACP 세션 생성
- 세션 타임아웃/정리 정책 필요

---

## 4. 이 프록시가 만들어지면 뭐가 달라지나

### 현재 상태
```
[PM 에이전트] → "코드 고쳐줘" → ??? (직접 코딩 에이전트 호출 방법 없음)
[테스트 에이전트] → "이 코드 테스트해줘" → ??? (터미널 접근 없음)
[DevOps 에이전트] → "배포 스크립트 만들어줘" → ??? (파일 생성 불가)
```

### 프록시가 있으면
```
[PM 에이전트] ──A2A──→ [프록시] ──ACP──→ [Claude Code] → 실제 코드 수정 + PR 생성
[테스트 에이전트] ──A2A──→ [프록시] ──ACP──→ [코딩 에이전트] → 테스트 실행 + 결과 반환
[DevOps 에이전트] ──A2A──→ [프록시] ──ACP──→ [코딩 에이전트] → 스크립트 생성 + 검증
```

### 구체적으로 열리는 시나리오

| 시나리오 | 설명 |
|---------|------|
| **멀티 에이전트 코드 리뷰** | QA 에이전트가 변경사항을 코딩 에이전트에게 보내서 리뷰 요청 |
| **자동 버그 수정 파이프라인** | 모니터링 에이전트가 에러 탐지 → 코딩 에이전트가 자동 패치 |
| **크로스 프레임워크 코딩** | LangGraph/CrewAI 기반 에이전트가 ACP 코딩 에이전트 활용 |
| **에이전트 팀 구성** | 프론트엔드 에이전트 + 백엔드 에이전트 + 테스트 에이전트 동시 협업 |
| **기업 코딩 표준 적용** | 정책 에이전트가 코딩 에이전트에게 "이 표준에 맞게 리팩터링해" |

---

## 5. 타당성 결론

### 개념 자체는 타당하다

| 평가 항목 | 점수 | 이유 |
|----------|------|------|
| 프로토콜 호환성 | ★★★★☆ | 메시지/스트리밍/상태 모델이 잘 매핑됨 |
| 기술적 실현성 | ★★★★☆ | 전송 계층 브릿지만 필요, 프로토콜 번역은 명확 |
| 실용적 가치 | ★★★★★ | 멀티 에이전트 코딩 워크플로우의 빈 공간을 채움 |
| 보안/운영 | ★★★☆☆ | 파일시스템 접근 + 터미널 실행의 보안 설계가 핵심 |
| 생태계 영향 | ★★★★★ | ACP 에이전트의 도달 범위를 A2A 전체로 확장 |

### 핵심 설계 결정이 필요한 3가지

1. **파일시스템 접근 정책**: 어디까지 자동으로 허용할 것인가?
2. **에이전트 프로세스 관리**: 요청마다 새로 spawn? 풀링? 컨테이너 격리?
3. **권한 에스컬레이션**: ACP의 permission request를 A2A 측에서 어떻게 처리할 것인가?

---

## 6. 역방향 프록시도 가능한가? (ACP → A2A)

반대 방향도 생각해볼 수 있음:
```
[코드 에디터] ──ACP──→ [코딩 에이전트] ──A2A──→ [외부 에이전트들]
```

예: 사용자가 Zed에서 "이 API의 문서를 작성해줘" → 코딩 에이전트가 A2A로 문서화 전문 에이전트에 위임

이것은 ACP 에이전트가 **A2A 클라이언트** 역할을 하는 것으로, 프록시보다는 에이전트 내부 기능에 가까움. MCP로 외부 도구에 접근하듯, A2A로 외부 에이전트에 접근하는 패턴.

---
---

# A2A ↔ ACP 프록시 — 구현 설계서

## Context

ACP 코딩 에이전트(Claude Code 등)는 현재 코드 에디터(Zed, VS Code)를 통해서만 접근 가능하다. A2A 멀티 에이전트 생태계에서 코딩 능력을 활용하려면, A2A ↔ ACP 간 프로토콜 변환 프록시가 필요하다. 이 프록시를 통해 어떤 A2A 에이전트든 표준 A2A 프로토콜로 ACP 코딩 에이전트에게 코딩 작업을 위임할 수 있게 된다.

---

## 7. 프로젝트 구조

```
a2a-acp-proxy/
├── pyproject.toml              # 프로젝트 메타데이터 + 의존성
├── README.md
├── config.yaml                 # 설정 파일
├── src/
│   └── a2a_acp_proxy/
│       ├── __init__.py
│       ├── __main__.py         # 엔트리포인트 (서버 시작)
│       ├── config.py           # 설정 모델 (Pydantic)
│       ├── server.py           # A2A 서버 설정 + AgentCard 정의
│       ├── executor.py         # AgentExecutor 구현 (A2A → ACP 브릿지 핵심)
│       ├── acp_client.py       # ACP 클라이언트 (subprocess + stdio JSON-RPC)
│       ├── translator.py       # 프로토콜 변환 (A2A ↔ ACP 메시지 매핑)
│       ├── session_manager.py  # A2A context_id ↔ ACP sessionId 매핑
│       ├── fs_provider.py      # ACP의 fs/read_text_file, fs/write_text_file 처리
│       ├── terminal_provider.py # ACP의 terminal/* 처리
│       └── permission_handler.py # ACP의 session/request_permission 처리
└── tests/
    ├── test_translator.py
    ├── test_acp_client.py
    └── test_executor.py
```

---

## 8. 기술 스택 및 의존성

```toml
[project]
name = "a2a-acp-proxy"
requires-python = ">=3.11"
dependencies = [
    "a2a-sdk[all]",     # A2A Python SDK (Starlette, Uvicorn 포함)
    "pydantic>=2.0",     # 설정 + 데이터 모델
    "pyyaml",            # config.yaml 파싱
]
```

**언어 선택: Python** — A2A SDK(`a2a-sdk`)가 Python으로 가장 성숙하고, ACP도 Python SDK가 있음. ACP의 stdio 통신은 SDK 없이 `asyncio.subprocess`로 직접 구현.

---

## 9. 핵심 컴포넌트 설계

### 9.1 `config.py` — 설정 모델

```python
class ProxyConfig(BaseModel):
    # A2A 서버 설정
    host: str = "0.0.0.0"
    port: int = 9100
    agent_name: str = "ACP Coding Agent (via A2A)"
    agent_description: str = "Bridges A2A to ACP coding agents"

    # ACP 에이전트 설정
    acp_agent_command: str = "claude"   # ACP 에이전트 실행 명령
    acp_agent_args: list[str] = []      # 추가 인자
    acp_working_dir: str = "/workspace" # 기본 작업 디렉토리

    # 세션 관리
    session_timeout_seconds: int = 3600  # 1시간
    max_concurrent_sessions: int = 10

    # 보안 정책
    permission_policy: str = "auto_approve"  # auto_approve | escalate | allowlist
    allowed_paths: list[str] = ["/workspace"]
    allowed_commands: list[str] = ["*"]
```

### 9.2 `acp_client.py` — ACP 클라이언트 (핵심)

ACP 에이전트 subprocess를 관리하고 JSON-RPC 2.0 over stdio로 통신.

```python
class ACPClient:
    """ACP 에이전트와 stdio를 통해 JSON-RPC 2.0으로 통신하는 클라이언트"""

    def __init__(self, command: str, args: list[str], working_dir: str):
        self.command = command
        self.args = args
        self.working_dir = working_dir
        self._process: asyncio.subprocess.Process | None = None
        self._request_id_counter = 0
        self._pending_requests: dict[int, asyncio.Future] = {}
        self._notification_handler: Callable | None = None
        self._request_handler: Callable | None = None  # Agent→Client 요청 처리

    async def start(self) -> None:
        """ACP 에이전트 프로세스 시작"""
        self._process = await asyncio.create_subprocess_exec(
            self.command, *self.args,
            stdin=asyncio.subprocess.PIPE,
            stdout=asyncio.subprocess.PIPE,
            stderr=asyncio.subprocess.PIPE,
            cwd=self.working_dir,
        )
        asyncio.create_task(self._read_loop())

    async def _read_loop(self) -> None:
        """stdout에서 NDJSON 라인 읽기 (응답 + 노티피케이션 + 요청 라우팅)"""
        while self._process and self._process.stdout:
            line = await self._process.stdout.readline()
            if not line:
                break
            msg = json.loads(line)
            if "id" in msg and "method" in msg:
                # Agent→Client 요청 (fs/*, terminal/*, session/request_permission)
                response = await self._request_handler(msg)
                await self._send(response)
            elif "id" in msg:
                # 응답 — pending request에 매칭
                future = self._pending_requests.pop(msg["id"], None)
                if future:
                    future.set_result(msg)
            else:
                # 노티피케이션 (sessionUpdate)
                await self._notification_handler(msg)

    async def _send(self, message: dict) -> None:
        """JSON-RPC 메시지를 stdin으로 전송"""
        data = json.dumps(message) + "\n"
        self._process.stdin.write(data.encode())
        await self._process.stdin.drain()

    async def send_request(self, method: str, params: dict) -> dict:
        """JSON-RPC 요청 전송 및 응답 대기"""
        self._request_id_counter += 1
        req_id = self._request_id_counter
        future = asyncio.get_event_loop().create_future()
        self._pending_requests[req_id] = future
        await self._send({
            "jsonrpc": "2.0", "id": req_id,
            "method": method, "params": params
        })
        return await future

    # --- 고수준 ACP 메서드 ---

    async def initialize(self) -> dict:
        return await self.send_request("initialize", {
            "protocolVersion": "0.6.0",
            "clientCapabilities": {
                "fs": {"readTextFile": True, "writeTextFile": True},
                "terminal": True
            },
            "clientInfo": {"name": "a2a-acp-proxy", "version": "0.1.0"}
        })

    async def new_session(self, cwd: str) -> str:
        result = await self.send_request("session/new", {"cwd": cwd})
        return result["result"]["sessionId"]

    async def prompt(self, session_id: str, text: str) -> dict:
        return await self.send_request("session/prompt", {
            "sessionId": session_id,
            "prompt": [{"type": "text", "text": text}]
        })

    async def cancel(self, session_id: str) -> None:
        # 노티피케이션 (id 없음)
        await self._send({
            "jsonrpc": "2.0",
            "method": "session/cancel",
            "params": {"sessionId": session_id}
        })

    async def stop(self) -> None:
        if self._process:
            self._process.terminate()
            await self._process.wait()
```

### 9.3 `translator.py` — 프로토콜 변환

```python
class ProtocolTranslator:
    """A2A ↔ ACP 메시지 형식 변환"""

    # --- A2A → ACP 변환 ---

    @staticmethod
    def a2a_message_to_acp_prompt(message: Message) -> list[dict]:
        """A2A Message의 Part들을 ACP ContentBlock 배열로 변환"""
        blocks = []
        for part in message.parts:
            if part.text:
                blocks.append({"type": "text", "text": part.text})
            elif part.raw:
                blocks.append({
                    "type": "image",
                    "data": base64.b64encode(part.raw).decode(),
                    "mimeType": part.media_type or "application/octet-stream"
                })
            elif part.data:
                blocks.append({
                    "type": "text",
                    "text": json.dumps(part.data)
                })
        return blocks

    # --- ACP → A2A 변환 ---

    @staticmethod
    def acp_session_update_to_a2a_event(
        update: dict, task_id: str, context_id: str
    ) -> TaskStatusUpdateEvent | TaskArtifactUpdateEvent | None:
        """ACP sessionUpdate 노티피케이션을 A2A 스트림 이벤트로 변환"""
        update_type = update.get("sessionUpdate")

        if update_type == "agent_message_chunk":
            text = update.get("content", {}).get("text", "")
            return TaskStatusUpdateEvent(
                task_id=task_id,
                context_id=context_id,
                status=TaskStatus(
                    state=TaskState.WORKING,
                    message=Message(
                        message_id=str(uuid4()),
                        role=Role.AGENT,
                        parts=[Part(text=text)]
                    )
                )
            )

        elif update_type == "tool_call":
            title = update.get("title", "")
            kind = update.get("kind", "")
            return TaskStatusUpdateEvent(
                task_id=task_id,
                context_id=context_id,
                status=TaskStatus(
                    state=TaskState.WORKING,
                    message=Message(
                        message_id=str(uuid4()),
                        role=Role.AGENT,
                        parts=[Part(text=f"[Tool: {kind}] {title}")]
                    )
                )
            )

        elif update_type in ("tool_call_update",):
            # tool_call_update에 completed된 파일 변경이 있으면 Artifact로 변환
            content_items = update.get("content", [])
            for item in content_items:
                if item.get("type") == "diff":
                    return TaskArtifactUpdateEvent(
                        task_id=task_id,
                        context_id=context_id,
                        artifact=Artifact(
                            artifact_id=str(uuid4()),
                            name=f"File change: {item.get('path', '')}",
                            parts=[Part(text=f"--- {item.get('path')}\n"
                                            f"-{item.get('oldText','')}\n"
                                            f"+{item.get('newText','')}")],
                        ),
                        append=False,
                        last_chunk=True,
                    )
        return None

    @staticmethod
    def acp_stop_reason_to_a2a_state(stop_reason: str) -> TaskState:
        """ACP stopReason을 A2A TaskState로 변환"""
        mapping = {
            "end_turn": TaskState.COMPLETED,
            "max_tokens": TaskState.COMPLETED,
            "max_turn_requests": TaskState.COMPLETED,
            "refusal": TaskState.FAILED,
            "cancelled": TaskState.CANCELED,
        }
        return mapping.get(stop_reason, TaskState.FAILED)
```

### 9.4 `session_manager.py` — 세션 관리

```python
class SessionManager:
    """A2A context_id ↔ ACP session 매핑 관리"""

    @dataclass
    class SessionEntry:
        acp_client: ACPClient
        session_id: str          # ACP sessionId
        context_id: str          # A2A context_id
        created_at: datetime
        last_used: datetime

    def __init__(self, config: ProxyConfig):
        self.config = config
        self._sessions: dict[str, SessionEntry] = {}  # context_id → entry

    async def get_or_create(self, context_id: str | None) -> SessionEntry:
        """context_id로 기존 세션을 찾거나 새로 생성"""
        if context_id and context_id in self._sessions:
            entry = self._sessions[context_id]
            entry.last_used = datetime.now()
            return entry

        # 새 ACP 클라이언트 + 세션 생성
        ctx_id = context_id or str(uuid4())
        client = ACPClient(
            self.config.acp_agent_command,
            self.config.acp_agent_args,
            self.config.acp_working_dir,
        )
        await client.start()
        await client.initialize()
        session_id = await client.new_session(self.config.acp_working_dir)

        entry = self.SessionEntry(
            acp_client=client,
            session_id=session_id,
            context_id=ctx_id,
            created_at=datetime.now(),
            last_used=datetime.now(),
        )
        self._sessions[ctx_id] = entry
        return entry

    async def cleanup_expired(self) -> None:
        """타임아웃된 세션 정리"""
        now = datetime.now()
        expired = [
            k for k, v in self._sessions.items()
            if (now - v.last_used).seconds > self.config.session_timeout_seconds
        ]
        for k in expired:
            entry = self._sessions.pop(k)
            await entry.acp_client.stop()
```

### 9.5 `executor.py` — AgentExecutor (핵심 브릿지)

A2A SDK의 `AgentExecutor`를 구현하여 A2A 요청을 ACP 호출로 변환.

```python
class ACPBridgeExecutor(AgentExecutor):
    """A2A 요청을 받아 ACP 에이전트에게 전달하는 실행기"""

    def __init__(self, session_manager: SessionManager, config: ProxyConfig):
        self.session_manager = session_manager
        self.translator = ProtocolTranslator()
        self.config = config

    async def execute(self, context: RequestContext, event_queue: EventQueue):
        """A2A SendMessage/SendStreamingMessage 처리"""
        message = context.message
        context_id = message.context_id
        task_id = context.task_id or str(uuid4())

        # 1. ACP 세션 획득
        entry = await self.session_manager.get_or_create(context_id)
        acp = entry.acp_client

        # 2. 사용자 메시지를 ACP 형식으로 변환
        prompt_text = self._extract_text(message)

        # 3. ACP sessionUpdate 노티피케이션을 A2A 이벤트로 전환하는 핸들러 등록
        async def on_notification(msg: dict):
            update = msg.get("params", {}).get("update", msg.get("params", {}))
            event = self.translator.acp_session_update_to_a2a_event(
                update, task_id, entry.context_id
            )
            if event:
                await event_queue.enqueue(event)

        async def on_agent_request(msg: dict) -> dict:
            """ACP 에이전트가 보내는 요청(파일/터미널/권한) 처리"""
            method = msg.get("method")
            params = msg.get("params", {})
            req_id = msg.get("id")

            if method == "fs/read_text_file":
                content = await self._read_file(params["path"])
                return {"jsonrpc": "2.0", "id": req_id, "result": {"content": content}}
            elif method == "fs/write_text_file":
                await self._write_file(params["path"], params["content"])
                return {"jsonrpc": "2.0", "id": req_id, "result": {}}
            elif method.startswith("terminal/"):
                return await self._handle_terminal(method, params, req_id)
            elif method == "session/request_permission":
                return await self._handle_permission(
                    params, req_id, event_queue, task_id, entry.context_id
                )
            return {"jsonrpc": "2.0", "id": req_id,
                    "error": {"code": -32601, "message": "Method not found"}}

        acp._notification_handler = on_notification
        acp._request_handler = on_agent_request

        # 4. WORKING 상태 알림
        await event_queue.enqueue(TaskStatusUpdateEvent(
            task_id=task_id,
            context_id=entry.context_id,
            status=TaskStatus(state=TaskState.WORKING),
        ))

        # 5. ACP에 프롬프트 전송 → 응답 대기
        result = await acp.prompt(entry.session_id, prompt_text)
        stop_reason = result.get("result", {}).get("stopReason", "end_turn")

        # 6. 최종 상태를 A2A로 전환
        final_state = self.translator.acp_stop_reason_to_a2a_state(stop_reason)
        await event_queue.enqueue(TaskStatusUpdateEvent(
            task_id=task_id,
            context_id=entry.context_id,
            status=TaskStatus(state=final_state),
        ))

    async def cancel(self, context: RequestContext, event_queue: EventQueue):
        """A2A CancelTask → ACP session/cancel"""
        task_id = context.task_id
        # context_id로 세션 찾기
        for entry in self.session_manager._sessions.values():
            await entry.acp_client.cancel(entry.session_id)
            break
        await event_queue.enqueue(TaskStatusUpdateEvent(
            task_id=task_id,
            context_id=context.context_id or "",
            status=TaskStatus(state=TaskState.CANCELED),
        ))

    def _extract_text(self, message: Message) -> str:
        """A2A Message에서 텍스트 추출"""
        texts = [p.text for p in message.parts if p.text]
        return "\n".join(texts)
```

### 9.6 `fs_provider.py` — 파일시스템 제공자

```python
class FilesystemProvider:
    """ACP 에이전트의 파일 읽기/쓰기 요청을 처리"""

    def __init__(self, config: ProxyConfig):
        self.allowed_paths = [Path(p) for p in config.allowed_paths]

    def _check_path(self, path: str) -> Path:
        """경로가 허용 범위 내인지 검증"""
        resolved = Path(path).resolve()
        if not any(self._is_subpath(resolved, ap) for ap in self.allowed_paths):
            raise PermissionError(f"Path {path} is outside allowed directories")
        return resolved

    async def read_file(self, path: str, line: int | None = None,
                        limit: int | None = None) -> str:
        p = self._check_path(path)
        content = p.read_text()
        if line or limit:
            lines = content.splitlines()
            start = (line or 1) - 1
            end = start + (limit or len(lines))
            content = "\n".join(lines[start:end])
        return content

    async def write_file(self, path: str, content: str) -> None:
        p = self._check_path(path)
        p.parent.mkdir(parents=True, exist_ok=True)
        p.write_text(content)
```

### 9.7 `terminal_provider.py` — 터미널 제공자

```python
class TerminalProvider:
    """ACP 에이전트의 terminal/* 요청을 처리"""

    def __init__(self, config: ProxyConfig):
        self.config = config
        self._terminals: dict[str, asyncio.subprocess.Process] = {}
        self._outputs: dict[str, str] = {}

    async def create(self, command: str, args: list[str],
                     cwd: str | None = None) -> str:
        terminal_id = str(uuid4())
        proc = await asyncio.create_subprocess_exec(
            command, *args,
            stdout=asyncio.subprocess.PIPE,
            stderr=asyncio.subprocess.STDOUT,
            cwd=cwd or self.config.acp_working_dir,
        )
        self._terminals[terminal_id] = proc
        self._outputs[terminal_id] = ""
        asyncio.create_task(self._collect_output(terminal_id, proc))
        return terminal_id

    async def get_output(self, terminal_id: str) -> dict:
        proc = self._terminals.get(terminal_id)
        exit_status = None
        if proc and proc.returncode is not None:
            exit_status = {"exitCode": proc.returncode}
        return {
            "output": self._outputs.get(terminal_id, ""),
            "truncated": False,
            "exitStatus": exit_status,
        }

    async def kill(self, terminal_id: str) -> None:
        proc = self._terminals.get(terminal_id)
        if proc:
            proc.terminate()

    async def wait_for_exit(self, terminal_id: str) -> dict:
        proc = self._terminals.get(terminal_id)
        if proc:
            await proc.wait()
            return {"exitCode": proc.returncode}
        return {}
```

### 9.8 `server.py` — A2A 서버 + AgentCard

```python
def create_agent_card(config: ProxyConfig) -> AgentCard:
    return AgentCard(
        name=config.agent_name,
        description=config.agent_description,
        version="0.1.0",
        url=f"http://{config.host}:{config.port}",
        capabilities=AgentCapabilities(streaming=True, pushNotifications=False),
        defaultInputModes=["text/plain"],
        defaultOutputModes=["text/plain", "application/json"],
        skills=[
            AgentSkill(
                id="code-edit",
                name="Code Editing",
                description="Read, write, and modify code files",
                tags=["coding", "refactor", "debug", "fix"],
                examples=["Fix the bug in auth.py", "Add error handling to the API"],
            ),
            AgentSkill(
                id="code-review",
                name="Code Review",
                description="Review code changes and suggest improvements",
                tags=["review", "quality", "best-practices"],
            ),
            AgentSkill(
                id="terminal",
                name="Command Execution",
                description="Run build, test, and deployment commands",
                tags=["build", "test", "deploy", "terminal"],
                examples=["Run the test suite", "Build the project"],
            ),
        ],
    )
```

### 9.9 `__main__.py` — 엔트리포인트

```python
async def main():
    config = load_config("config.yaml")
    session_manager = SessionManager(config)

    executor = ACPBridgeExecutor(session_manager, config)
    agent_card = create_agent_card(config)

    handler = DefaultRequestHandler(
        agent_executor=executor,
        task_store=InMemoryTaskStore(),
    )

    app = A2AStarletteApplication(
        agent_card=agent_card,
        http_handler=handler,
    )

    # 세션 정리 백그라운드 태스크
    async def cleanup_loop():
        while True:
            await asyncio.sleep(300)
            await session_manager.cleanup_expired()

    asyncio.create_task(cleanup_loop())

    uvicorn.run(app.build(), host=config.host, port=config.port)
```

---

## 10. 메시지 흐름 상세

### 10.1 SendMessage (비스트리밍)

```
A2A Client                    Proxy                           ACP Agent
    │                           │                                 │
    │  POST /message:send       │                                 │
    │  {message: {text: "..."}} │                                 │
    │ ─────────────────────────→│                                 │
    │                           │ session/prompt {text: "..."}    │
    │                           │────────────────────────────────→│
    │                           │                                 │
    │                           │  ← sessionUpdate(agent_msg)    │
    │                           │  ← sessionUpdate(tool_call)    │
    │                           │  ← fs/read_text_file (요청)     │
    │                           │  → {content: "..."} (응답)      │
    │                           │  ← sessionUpdate(tool_call_upd)│
    │                           │  ← PromptResponse(end_turn)    │
    │                           │←────────────────────────────────│
    │                           │                                 │
    │  {task: {state: COMPLETED,│                                 │
    │   artifacts: [...]}}      │                                 │
    │ ←─────────────────────────│                                 │
```

### 10.2 SendStreamingMessage (스트리밍)

```
A2A Client                    Proxy                           ACP Agent
    │                           │                                 │
    │  POST /message:stream     │                                 │
    │ ─────────────────────────→│ session/prompt                  │
    │                           │────────────────────────────────→│
    │                           │                                 │
    │  SSE: TaskStatusUpdate    │  ← sessionUpdate(agent_msg)    │
    │  (state: WORKING)         │←────────────────────────────────│
    │ ←─────────────────────────│                                 │
    │                           │                                 │
    │  SSE: TaskStatusUpdate    │  ← sessionUpdate(tool_call)    │
    │  (state: WORKING, msg)    │←────────────────────────────────│
    │ ←─────────────────────────│                                 │
    │                           │                                 │
    │  SSE: TaskArtifactUpdate  │  ← sessionUpdate(tool_call_upd)│
    │  (artifact: file diff)    │←────────────────────────────────│
    │ ←─────────────────────────│                                 │
    │                           │                                 │
    │  SSE: TaskStatusUpdate    │  ← PromptResponse(end_turn)    │
    │  (state: COMPLETED)       │←────────────────────────────────│
    │ ←─────────────────────────│                                 │
```

### 10.3 Permission 에스컬레이션

```
A2A Client                    Proxy                           ACP Agent
    │                           │                                 │
    │                           │  ← session/request_permission  │
    │                           │    "이 파일 삭제해도 될까?"       │
    │                           │←────────────────────────────────│
    │                           │                                 │
    │  SSE: TaskStatusUpdate    │  (permission_policy에 따라)      │
    │  (state: INPUT_REQUIRED,  │                                 │
    │   msg: "이 파일 삭제?")    │                                 │
    │ ←─────────────────────────│                                 │
    │                           │                                 │
    │  POST /message:send       │                                 │
    │  {text: "승인"}           │                                 │
    │ ─────────────────────────→│                                 │
    │                           │  → permission response(allow)   │
    │                           │────────────────────────────────→│
```

---

## 11. 설정 파일 예시 (`config.yaml`)

```yaml
# A2A 서버 설정
host: "0.0.0.0"
port: 9100
agent_name: "Claude Code (A2A Bridge)"
agent_description: "ACP coding agent accessible via A2A protocol"

# ACP 에이전트
acp_agent_command: "claude"
acp_agent_args: ["--acp"]
acp_working_dir: "/workspace/my-project"

# 세션 관리
session_timeout_seconds: 3600
max_concurrent_sessions: 5

# 보안
permission_policy: "auto_approve"    # auto_approve | escalate | allowlist
allowed_paths:
  - "/workspace/my-project"
allowed_commands:
  - "npm"
  - "cargo"
  - "python"
  - "git"
```

---

## 12. 구현 순서

| 단계 | 파일 | 내용 |
|------|------|------|
| 1 | `config.py` | Pydantic 설정 모델 + YAML 로딩 |
| 2 | `acp_client.py` | ACP subprocess 관리 + JSON-RPC stdio 통신 |
| 3 | `fs_provider.py` | 파일 읽기/쓰기 (경로 검증 포함) |
| 4 | `terminal_provider.py` | 명령 실행 + 출력 수집 |
| 5 | `permission_handler.py` | 권한 요청 처리 정책 |
| 6 | `translator.py` | A2A ↔ ACP 메시지 변환 |
| 7 | `session_manager.py` | context_id ↔ sessionId 매핑 + 세션 풀 |
| 8 | `executor.py` | AgentExecutor 구현 (모든 컴포넌트 통합) |
| 9 | `server.py` | AgentCard + A2AStarletteApplication |
| 10 | `__main__.py` | 엔트리포인트 |
| 11 | `tests/` | 단위 테스트 |

---

## 13. 검증 방법

### 수동 테스트
```bash
# 1. 프록시 서버 시작
python -m a2a_acp_proxy

# 2. AgentCard 확인
curl http://localhost:9100/.well-known/agent-card.json

# 3. 메시지 전송 (JSON-RPC)
curl -X POST http://localhost:9100/ \
  -H "Content-Type: application/json" \
  -d '{"jsonrpc":"2.0","id":1,"method":"message/send","params":{
    "message":{"messageId":"m1","role":"user",
    "parts":[{"text":"Fix the bug in main.py"}]}}}'

# 4. 스트리밍 테스트
curl -N -X POST http://localhost:9100/ \
  -H "Content-Type: application/json" \
  -d '{"jsonrpc":"2.0","id":2,"method":"message/stream","params":{
    "message":{"messageId":"m2","role":"user",
    "parts":[{"text":"Add error handling"}]}}}'
```

### 자동 테스트
- `test_translator.py`: A2A↔ACP 메시지 변환 단위 테스트
- `test_acp_client.py`: mock subprocess로 JSON-RPC 통신 테스트
- `test_executor.py`: 전체 흐름 통합 테스트
