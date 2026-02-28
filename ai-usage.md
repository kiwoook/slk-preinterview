# AI 활용 기록

이 프로젝트는 설계 전 과정에서 AI(Claude)를 활용했습니다.
각 주요 설계 결정에 대해 어떤 의도로 AI를 활용했는지, 어떤 결과를 얻었는지 기록합니다.

---

## 공통. 세션 간 맥락 관리

| 항목 | 내용 |
|------|------|
| 의도 | AI와 대화는 세션이 끊기면 이전 맥락이 사라진다. 설계가 누적될수록 새 세션마다 배경을 다시 설명하는 비용이 커지고, 맥락 손실로 인해 이미 결정된 사항이 번복되거나 중복 논의되는 문제가 발생할 수 있다. |
| 방법 | `CLAUDE.md`에 역할·문서 구조 규칙·의사결정 원칙을 정의하고, 파트별 `context.md`(`.claude/part-a-context.md`, `.claude/part-b-context.md`)에 핵심 목표 요약과 주요 문서 파일 위치를 기록. 새 세션 시작 시 context.md를 먼저 읽어 현재 설계 상황을 즉시 파악할 수 있도록 했다. |
| 핵심 결과 | 세션이 바뀌어도 "어디까지 왔는지"를 재설명할 필요 없이 미결 이슈부터 바로 이어서 작업 가능. 이미 확정된 ADR·가정 사항이 번복되지 않고 현재 설계 상황에 집중할 수 있었다. |

---

## Part A. 서버 설계

### 1. Credential 관리 모델 결정

| 항목 | 내용 |
|------|------|
| 의도 | 각 조직이 업체와 직접 계약하는 구조(Integration)인지, 서비스 제공사가 중간에서 관리하는 구조(Reseller)인지 판단하고 DB 설계 방향을 결정해달라고 요청 |
| 핵심 결과 | 알림톡 채널은 사업자마다 별도 발급되므로 공유 불가 → Integration 모델 확정. 조직별 Credential을 별도 테이블에 JSON으로 저장하고 AES-256으로 암호화하는 방식 도출 |
| 설계 반영 | [ADR-002](part-a-server/docs/architecture/adr.md#adr-002-credential-및-조직-업체-매핑-저장-방식) · [requirements.md A-08](part-a-server/docs/requirement/requirements.md#a-08-인증-정보credential-관리-방식) |

---

### 2. API Key 보안 관리 및 서명 생성 방식

| 항목 | 내용 |
|------|------|
| 의도 | 조직별로 서로 다른 API Key를 안전하게 관리하고, 외부 API 규격에 맞는 인증 방식을 구현하고자 함 |
| 핵심 결과 | AES-256-GCM 양방향 암호화 저장 및 런타임 복호화 후 서명 생성 방식 채택 |
| 설계 반영 | 기술적 결정 사항에 암호화 알고리즘 선택 근거와 발송 시점의 서명 생성 흐름을 구체화 → [credential-encryption.md](part-a-server/docs/architecture/credential-encryption.md) |

---

### 3. Provider 추상화 수준 결정

| 항목 | 내용 |
|------|------|
| 의도 | 클라우드메시지와 센드톡 두 업체의 API 구조 차이를 분석하고, Strategy Pattern 적용 시 어느 수준까지 추상화할지 판단해달라고 요청 |
| 핵심 결과 | 템플릿 식별자·응답 상태값 등 근본적으로 다른 개념까지 통합하면 성급한 추상화 → 업체 선택만 추상화(ProviderFactory), 실제 API 호출은 각 Provider가 독립 책임 |
| 설계 반영 | [ADR-001](part-a-server/docs/architecture/adr.md#adr-001-provider-추상화-수준) · [provider-design.md](part-a-server/docs/architecture/provider-design.md) |

---

### 4. 발송 신뢰성 설계 — 인프라 및 유실 방지 전략 수립

| 항목 | 내용 |
|------|------|
| 의도 | B2B 서비스 특성상 메시지 발송의 신뢰성이 중요하므로, 인프라 복잡도를 낮추면서도 유실을 방지할 수 있는 아키텍처를 설계하고자 함 |
| 프롬프트 | "Spring Boot 환경에서 외부 메시지 API 호출 시 유실을 방지하기 위한 가장 안정적인 패턴을 알려줘. 단, B2B 환경이며 초기 단계라 Kafka 도입은 부담스러운 상황이야." |
| 핵심 결과 | Transactional Outbox Pattern과 Redis Streams를 활용한 비동기 처리 방식 제안 |
| 설계 반영 | Outbox 패턴을 도입하여 DB 트랜잭션과 발송 요청의 원자성을 보장하도록 설계. Redis Streams는 향후 볼륨 증가 시 전환 옵션으로 보류 → [outbox-batch-design.md](part-a-server/docs/architecture/outbox-batch-design.md) · [system-design.md §3](part-a-server/docs/architecture/system-design.md) |

---

### 5. 기술 스택 선택

| 항목 | 내용 |
|------|------|
| 의도 | Spring Boot + Java 21과 NestJS를 비교하여 이 시스템에 적합한 스택을 선택해달라고 요청 |
| 핵심 결과 | 트랜잭션 관리, HMAC-SHA256 CPU 처리, Spring Batch 내장, Virtual Threads + RestClient 조합이 이 시스템의 요구사항에 부합 → Spring Boot 3.x + Java 21 선택 |
| 설계 반영 | [ADR-004](part-a-server/docs/architecture/adr.md#adr-004-기술-스택-선택) |

---

### 6. API 설계 — 연결 테스트 응답 코드

| 항목 | 내용 |
|------|------|
| 의도 | 연결 테스트 실패 시 HTTP 상태 코드를 어떻게 반환해야 하는지 판단해달라고 요청 |
| 핵심 결과 | HTTP 상태 코드는 우리 서버와의 통신 성공 여부를 나타내므로 200 반환이 의미적으로 정확. 외부 업체 연결 결과는 `success` 필드로 구분 |
| 설계 반영 | [api-spec.md POST /test](part-a-server/docs/api/api-spec.md#post-organizationsorgidmessage-providertest) |

---

### 7. 과제 원문 재검토 및 Risks & Mitigation 도출

| 항목 | 내용 |
|------|------|
| 의도 | 설계 마무리 단계에서 AI에게 과제 원문을 다시 확인하고 누락된 요구사항이 있는지 검수해달라고 요청 |
| 핵심 결과 | Part A의 세 번째 필수 항목인 "Risks & Mitigation (Risk → Impact → Strategy)" 형식이 누락된 것을 발견. 이후 카테고리별 리스크 브레인스토밍 → 핵심 4개(마이그레이션 실패, 암호화 키 유실, Batch 장애, 업체 API 장기 장애) 선별 |
| 설계 반영 | [verification-rollback.md §1](part-a-server/docs/architecture/verification-rollback.md) — R-01~R-04 추가 |

---

### 8. 리스크 논의 중 설계 결함 발견 — outbox_messages.provider_type 누락

| 항목 | 내용 |
|------|------|
| 의도 | 업체 전환 시 PENDING 건의 templateKey 불일치를 리스크로 기록하려 했으나, "업체 변경해도 기존 PENDING 건은 영향이 없어야 하는 거 아닌가?"라는 의문에서 출발 |
| 핵심 결과 | `outbox_messages`에 `provider_type` 컬럼이 없어 Batch가 처리 시점의 현재 업체 설정을 읽는 구조였음을 발견. 업체 전환 시 PENDING 건이 새 업체로 처리되는 설계 결함 확인. `provider_type`을 outbox 저장 시점에 고정하는 방식으로 수정 |
| 설계 반영 | [system-design.md](part-a-server/docs/architecture/system-design.md) — `outbox_messages` 스키마에 `provider_type` 추가 · [provider-design.md](part-a-server/docs/architecture/provider-design.md) — ProviderFactory 흐름 수정 |

---

## Part B. 화면 설계

### 7. 화면 구조 설계 (2-zone 레이아웃)

| 항목 | 내용 |
|------|------|
| 의도 | 운용자 실수를 최소화하는 업체 설정 화면의 레이아웃 구조를 설계해달라고 요청 |
| 핵심 결과 | 현재 설정(읽기 전용 카드)과 편집 영역을 명확히 분리하는 2-zone 구조 도출. 편집 중에도 현재 적용 중인 설정을 항상 확인할 수 있어 실수 방지 |
| 설계 반영 | [UI-01](part-b-ui/docs/architecture/ui-decisions.md#ui-01-현재-설정-표시-방식) · [screen-flow.md](part-b-ui/docs/architecture/screen-flow.md) |

---

### 8. 프로토타입 생성

| 항목 | 내용 |
|------|------|
| 의도 | 설계 문서(화면 흐름, UI 결정, Credential 필드 명세)를 기반으로 실제 동작하는 프로토타입을 생성해달라고 요청 |
| 핵심 결과 | 프로토타입을 통해 실제 구현 전 사용자가 미리 실험해볼 수 있었으며, 엣지 케이스 식별 및 사용자 경험 검증이 가능했음 |
| 설계 반영 | [프로토타입 보기](https://claude.ai/public/artifacts/ef346d90-b364-482c-beff-0290ec0b8ed6) · [prototype.md](part-b-ui/docs/prototype/prototype.md) |

---

### 9. 연결 테스트 강제 여부 결정

| 항목 | 내용 |
|------|------|
| 의도 | 잘못된 Credential 저장을 막기 위해 연결 테스트를 저장의 필수 조건으로 할지 판단해달라고 요청 |
| 핵심 결과 | 테스트 실패 상태에서 저장 버튼 비활성화. 단, 테스트 미실행 상태는 저장 허용 (테스트 API 자체가 낮은 우선순위임을 감안) |
| 설계 반영 | [UI-04](part-b-ui/docs/architecture/ui-decisions.md#ui-04-연결-테스트-위치-및-필수-여부) |

---

### 10. 업체 변경 확인 다이얼로그 설계

| 항목 | 내용 |
|------|------|
| 의도 | 업체 변경이 미치는 영향을 운용자에게 어떻게 안내하고 실수를 방지할지 설계해달라고 요청 |
| 핵심 결과 | 업체 변경 시에만 확인 다이얼로그 표시 (Credential만 변경 시 생략). 다이얼로그에 "진행 중인 발송은 기존 업체로 완료됩니다. 변경은 다음 발송부터 적용됩니다" 문구 명시 |
| 설계 반영 | [UI-05](part-b-ui/docs/architecture/ui-decisions.md#ui-05-업체-변경-시-확인-다이얼로그) |
