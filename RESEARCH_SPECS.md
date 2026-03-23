# A2A (Agent-to-Agent) Protocol - 스펙 조사 보고서

## 1. 프로젝트 개요

**A2A(Agent2Agent) Protocol**은 서로 다른 프레임워크로 구축되고 다른 조직에서 배포한 독립적인 AI 에이전트 간의 보안 통신 및 상호운용성을 가능하게 하는 오픈 프로토콜입니다.

- **라이선스**: Apache 2.0
- **주관**: The Linux Foundation (Google 기여)
- **현재 버전**: Release Candidate v1.0 (이전 안정 버전: v0.3.0)
- **MCP와의 관계**: MCP는 Agent-to-Tool, A2A는 Agent-to-Agent 통신을 담당 (상호보완적)

### 핵심 목표
- 에이전트 간 능력 발견 (Discovery)
- 상호작용 모드 협상 (텍스트, 폼, 미디어)
- 장기 실행 태스크에 대한 안전한 협업
- 불투명 실행 (내부 상태/메모리/도구 노출 없음)

---

## 2. 전송 프로토콜 및 메시지 포맷

| 항목 | 상세 |
|------|------|
| **주요 전송** | HTTP(S) + JSON-RPC 2.0 |
| **추가 바인딩** | gRPC/Protocol Buffers 3, HTTP+REST |
| **보안** | HTTPS 필수 (TLS 1.2+) |
| **정규 스펙** | `specification/a2a.proto` (807줄, proto3) |
| **생성 산출물** | `a2a.json` (JSON Schema 2020-12) |

---

## 3. 인증 및 보안

### 지원 인증 방식
| 방식 | 설명 |
|------|------|
| **OAuth 2.0** | Authorization Code (PKCE), Client Credentials, Device Code |
| **OpenID Connect** | OIDC Discovery 엔드포인트 |
| **HTTP Authentication** | Bearer Token, Basic Auth |
| **API Key** | Header, Query, Cookie |
| **Mutual TLS (mTLS)** | 인증서 기반 인증 |

- 인증 정보는 A2A 페이로드가 아닌 HTTP 헤더로 전달 (예: `Authorization: Bearer <TOKEN>`)
- Agent Card의 `securitySchemes` 필드에 지원 방식 선언
- 태스크 중 추가 인증 요청 가능 (`AUTH_REQUIRED` 상태)

### 엔터프라이즈 보안
- GDPR/CCPA/HIPAA 규정 준수 지원
- OpenTelemetry 분산 추적
- W3C Trace Context 헤더
- 감사 로깅 (태스크 생성, 상태 전이, 민감 데이터 접근)

---

## 4. 핵심 RPC 메서드 (11개)

### 메시지 관련
| 메서드 | 엔드포인트 | 설명 |
|--------|-----------|------|
| `SendMessage` | `POST /message:send` | 동기 요청/응답 |
| `SendStreamingMessage` | `POST /message:stream` | SSE 기반 실시간 스트리밍 |

### 태스크 관리
| 메서드 | 엔드포인트 | 설명 |
|--------|-----------|------|
| `GetTask` | `GET /tasks/{id}` | 태스크 상태 조회 |
| `ListTasks` | `GET /tasks` | 태스크 목록 (커서 기반 페이지네이션) |
| `CancelTask` | `POST /tasks/{id}:cancel` | 태스크 취소 |
| `SubscribeToTask` | `GET /tasks/{id}:subscribe` | SSE 재구독 |

### 푸시 알림
| 메서드 | 엔드포인트 | 설명 |
|--------|-----------|------|
| `CreateTaskPushNotificationConfig` | `POST /tasks/{task_id}/pushNotificationConfigs` | 푸시 알림 설정 |
| `GetTaskPushNotificationConfig` | `GET /tasks/{task_id}/pushNotificationConfigs/{id}` | 설정 조회 |
| `ListTaskPushNotificationConfigs` | `GET /tasks/{task_id}/pushNotificationConfigs` | 설정 목록 |
| `DeleteTaskPushNotificationConfig` | `DELETE /tasks/{task_id}/pushNotificationConfigs/{id}` | 설정 삭제 |

### Agent Card
| 메서드 | 엔드포인트 | 설명 |
|--------|-----------|------|
| `GetExtendedAgentCard` | `GET /extendedAgentCard` | 인증된 Agent Card 조회 |

> 모든 엔드포인트는 멀티테넌시 지원: `/{tenant}/...` 형태도 사용 가능

---

## 5. 핵심 데이터 구조

### Agent Card
- 위치: `https://{domain}/.well-known/agent-card.json`
- 포함 정보: 에이전트 이름, 설명, 버전, 프로바이더, 스킬, 보안 요구사항
- JWS 서명 지원 (RFC 7515 + JSON Canonicalization RFC 8785)

### Message
```
Message {
  messageId: string         // 고유 식별자
  role: ROLE_USER | ROLE_AGENT
  contextId: string         // 관련 태스크/메시지 그룹핑
  taskId: string            // 연관 태스크
  parts: Part[]             // 콘텐츠 배열
  metadata: map             // 커스텀 키-값 데이터
  extensions: string[]      // 적용 확장 URI
  referenceTaskIds: string[] // 참조 태스크
}
```

### Part (콘텐츠 컨테이너)
```
Part {
  oneof content:
    text: string            // 텍스트
    raw: bytes              // 바이너리 (JSON에서 base64)
    url: string             // 파일 참조 URI
    data: JSON object       // 구조화된 JSON
  mediaType: string         // MIME 타입
  filename: string          // 파일명 (선택)
  metadata: map             // 추가 컨텍스트
}
```

### Task
```
Task {
  id: string                // UUID
  contextId: string         // 상호작용 그룹
  status: TaskStatus        // 상태 객체
  artifacts: Artifact[]     // 태스크 출력물
  history: Message[]        // 메시지 이력
  metadata: map             // 커스텀 데이터
  createdAt: timestamp      // 생성 시각 (ISO 8601)
  lastModified: timestamp   // 수정 시각 (ISO 8601)
}
```

### Task 상태 머신
```
SUBMITTED → WORKING → COMPLETED (종료)
                    → FAILED (종료)
                    → CANCELED (종료)
                    → REJECTED (종료)
                    → INPUT_REQUIRED (중단 - 사용자 입력 대기)
                    → AUTH_REQUIRED (중단 - 인증 대기)
```

### Artifact (태스크 산출물)
```
Artifact {
  artifactId: string        // 태스크 내 고유
  name: string              // 이름
  description: string       // 설명
  parts: Part[]             // 구성 요소
  metadata: map             // 커스텀 데이터
  extensions: string[]      // 확장 URI
}
```

---

## 6. 상호작용 패턴

### 6.1 동기 (Request/Response)
```
Client → SendMessage → Agent → 즉시 응답 (Message 또는 Task)
```

### 6.2 스트리밍 (Server-Sent Events)
```
Client → SendStreamingMessage → Agent
Agent → HTTP 200 + Content-Type: text/event-stream
Agent → 실시간 이벤트 스트림 (Task, TaskStatusUpdateEvent, TaskArtifactUpdateEvent)
Agent → 태스크 종료/중단 시 연결 종료
```

### 6.3 비동기 (Push Notifications)
```
Client → SendMessage (PushNotificationConfig 포함) → Agent
Agent → 즉시 태스크 ID 응답
Agent → [나중에] 클라이언트 웹훅 URL로 POST
```
- 장기 실행 태스크, 모바일/서버리스 클라이언트에 적합

### 6.4 폴링 (Hybrid)
```
Client → GetTask(taskId) → Agent (주기적 호출)
```

---

## 7. 에이전트 발견 (Discovery)

| 방법 | 설명 |
|------|------|
| **Well-Known URI** | `/.well-known/agent-card.json` (RFC 8615) |
| **큐레이션 레지스트리** | 중앙 중개자가 Agent Card 컬렉션 관리 |
| **직접 설정** | 하드코딩 URL, 환경변수, 설정 파일 |
| **인증 확장 카드** | 기본 카드(공개) + 확장 카드(인증 후) |

---

## 8. SDK 생태계

| 언어 | 패키지 | 저장소 |
|------|--------|--------|
| **Python** | `pip install a2a-sdk` | github.com/a2aproject/a2a-python |
| **Go** | `go get github.com/a2aproject/a2a-go` | github.com/a2aproject/a2a-go |
| **JavaScript/TypeScript** | `npm install @a2a-js/sdk` | github.com/a2aproject/a2a-js |
| **Java** | Maven | github.com/a2aproject/a2a-java |
| **.NET** | `dotnet add package A2A` | github.com/a2aproject/a2a-dotnet |

### SDK 공통 구성요소
- Client Library / Server Library
- Message/Task Builders
- Authentication Adapters (OAuth2, OIDC, API Key, mTLS)
- Streaming Handlers (SSE)
- Protocol Bindings (JSON-RPC, gRPC, REST)
- Extension Handlers

---

## 9. 확장 (Extensions) 메커니즘

- Agent Card의 `capabilities.extensions[]`에 선언
- `A2A-Extensions` HTTP 헤더로 활성화
- 추가 가능 항목: 새 데이터 필드, 새 RPC 메서드, 상태 머신 변경, 프로필 요구사항
- URI 기반 버전 관리 (예: `https://example.com/ext/my-ext/v1`)
- 필수/선택 표시 가능

### 예시 확장
- **Secure Passport**: 컨텍스트 개인화
- **Timestamp**: 메타데이터에 타임스탬프 추가
- **Traceability**: 감사 로깅 강화
- **Agent Gateway Protocol (AGP)**: 자율 스쿼드 라우팅

---

## 10. 버전 히스토리

### v1.0 (RC) - 최신
- Enum 값 표준화 (SCREAMING_SNAKE_CASE)
- 통합 Part 구조
- 인터페이스별 프로토콜 버전 관리
- 멀티테넌시 지원
- 커서 기반 페이지네이션

### v0.3.0 - 이전 안정 버전
- Well-known URI → `agent-card.json`으로 변경
- 서명 지원 추가
- mTLS 보안 스킴 추가
- 확장 Agent Card 지원

---

## 11. 핵심 설계 원칙

1. **단순성**: HTTP, JSON-RPC 2.0, SSE 등 기존 웹 표준 재사용
2. **엔터프라이즈 레디**: 보안, 인증, 모니터링 내장
3. **비동기 우선**: 장기 실행 작업의 네이티브 지원
4. **모달리티 무관**: 텍스트, 바이너리, 구조화 데이터, 임베디드 UI
5. **불투명 실행**: 에이전트 내부 미노출
6. **상호운용성**: 프레임워크, 벤더, 조직 간 동작
7. **확장성**: 도메인별 확장 메커니즘

---

## 12. 주요 파일 경로

| 파일 | 설명 |
|------|------|
| `specification/a2a.proto` | 정규 프로토콜 스펙 (807줄) |
| `specification/json/` | JSON Schema 산출물 |
| `docs/specification.md` | 전체 스펙 문서 |
| `docs/topics/` | 개념 문서 (7개 파일) |
| `docs/tutorials/python/` | Python 튜토리얼 (8개 모듈) |
| `CHANGELOG.md` | 버전 변경 이력 |
| `mkdocs.yml` | 문서 사이트 설정 |

---

*조사일: 2026-03-23*
