[← Part A 홈](../../README.md)

`Path: part-a-server/docs/architecture/adr.md`

# 기술적 의사결정 (ADR)

---

## ADR-001. Provider 추상화 수준

### 고민

Strategy Pattern을 적용해 업체를 추상화할 때, **어느 수준까지 추상화할 것인가?**

두 업체의 API를 분석한 결과 구조적 차이가 상당하다:

| 항목 | 클라우드메시지 | 센드톡 |
|------|--------------|--------|
| 알림톡 템플릿 식별자 | `templateId` (업체 콘솔에 별도 등록) | `templateCode` (업체 콘솔에 별도 등록) |
| 템플릿 변수 | `params` | `variables` |
| 발송 직후 상태값 | `QUEUED` (큐 대기) | `ACCEPTED` (요청 수락) |
| 응답 구조 | 직접 객체 반환 | `{ success, data }` 래퍼 |

- **대안 1: 완전 통합 인터페이스** — 파라미터·응답까지 모두 공통 인터페이스로 정규화
  - 문제: 템플릿 ID는 업체별로 별도 등록된 값이므로 공통 파라미터로 받으면 "어떤 업체 기준 ID인가?" 혼동 발생.
    응답 상태값(`QUEUED` vs `ACCEPTED`)도 의미가 달라 억지 정규화 시 정보 손실 우려.
  - **성급한 추상화의 위험**: 두 업체가 근본적으로 다른 개념을 쓰는 영역까지 하나로 맞추면 추상화가 오히려 복잡도를 높인다.

- **대안 2: 업체 선택만 추상화** — ProviderFactory가 어떤 업체를 쓸지만 결정. 실제 발송 로직은 각 업체 구현체가 독립적으로 책임.
  - 장점: 각 업체의 고유한 특성을 그대로 보존. 인터페이스가 얇아 유지보수 용이.
  - 단점: MessageService가 Provider에 위임 후 업체별 응답 차이를 일부 인지할 수 있음.

### 선택 → 업체 선택만 추상화 (대안 2)

### 근거

- 템플릿 식별자와 변수명은 단순 필드명 차이가 아니라 **업체별 독립 등록 체계**를 가진다. 이를 하나의 파라미터로 강제하면 호출부에서 혼동이 발생한다.
- 응답 상태값의 의미 차이(`QUEUED` vs `ACCEPTED`)는 정규화 시 정보 손실이 발생한다.
- 현재 업체가 2개인 상황에서 완전 통합 인터페이스는 성급한 추상화다. 업체가 늘어나고 공통 패턴이 명확해지는 시점에 통합을 검토한다.

### 설계 구조

```
MessageService
  └── ProviderFactory              ← 업체 선택만 책임
        ├── CloudMessageProvider  ← 클라우드메시지 API 호출 전담 (독립)
        └── SendtalkProvider      ← 센드톡 API 호출 전담 (독립)
```

**ProviderFactory** — `organization_message_providers`에서 `provider_type`과 암호화된 Credential을 읽어 복호화 후 맞는 Provider 인스턴스를 반환한다. 업체 선택 로직이 여기에만 존재한다.

**각 Provider** — 자신의 업체 API 명세에만 의존한다. 전화번호 형식 변환, 인증 헤더 생성, 응답 파싱 등 업체 고유의 처리는 Provider 내부에서만 처리한다.

**MessageService** — Provider가 무엇인지 모른다. ProviderFactory에서 받은 Provider를 호출할 뿐이다.

### 향후 고려사항

공통 인터페이스가 필요해지는 시점의 신호:
- 업체가 3개 이상으로 늘어나고 공통 패턴이 눈에 보일 때
- MessageService에서 업체별 분기 처리가 반복될 때

---

## ADR-002. Credential 및 조직-업체 매핑 저장 방식

> **[재설계]** 초기 설계(전역 환경변수)에서 변경됨.
> 재설계 배경: 각 Organization이 메시지 업체와 직접 계약하는 Integration 모델(Model 2)로 확정됨에 따라, 조직별 독립 Credential 저장이 필수 요구사항이 됨.

### 고민

각 조직이 자신의 업체 API 키를 보유하며, UI에서 직접 입력한다. 이를 어디에, 어떻게 저장할 것인가?

두 업체의 Credential 구조가 이질적이다:

| 필드 | 클라우드메시지 | 센드톡 |
|------|--------------|--------|
| 인증 키 | AccessKey + SecretKey | API Key |
| 웹훅 검증 | - | Secret Key |
| 알림톡 발신 프로필 | SenderProfileId | - |
| 발신번호 | senderPhoneNumber | senderPhoneNumber |

- **대안 1: Organization 테이블에 컬럼 추가** — provider별 credential 컬럼을 Organization에 직접 추가
  - 단점: 업체마다 구조가 달라 타 업체 컬럼이 NULL로 낭비. Organization 테이블 비대화. 업체 추가 시 스키마 변경.

- **대안 2: 별도 테이블 + 고정 컬럼** — `organization_message_providers` 테이블에 업체별 컬럼 고정
  - 단점: 대안 1과 동일하게 업체 추가 시 스키마 변경 필요.

- **대안 3: 별도 테이블 + 암호화 JSON 컬럼** — provider_type + credentials(JSON) 구조
  - 장점: 업체별 이질적 구조를 유연하게 수용. 업체 추가 시 스키마 변경 불필요.
  - 단점: DB 레벨 타입 검증 불가 → 애플리케이션 레이어에서 보완.

### 선택 → 대안 3 (별도 테이블 + 암호화 JSON Credential)

```sql
CREATE TABLE organization_message_providers (
    id             BIGINT       NOT NULL AUTO_INCREMENT,
    org_id         BIGINT       NOT NULL UNIQUE,
    provider_type  ENUM('cloud_message', 'sendtalk') NOT NULL,
    credentials    TEXT         NOT NULL,   -- AES-256 암호화된 JSON
    created_at     DATETIME     NOT NULL DEFAULT CURRENT_TIMESTAMP,
    updated_at     DATETIME     NOT NULL DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP,
    PRIMARY KEY (id),
    FOREIGN KEY (org_id) REFERENCES organizations(id)
);
```

**credentials JSON 구조 (암호화 전)**

```json
// 클라우드메시지
{
  "accessKey": "...",
  "secretKey": "...",
  "senderProfileId": "PROFILE_...",
  "senderPhoneNumber": "0215881234"
}

// 센드톡
{
  "apiKey": "...",
  "secretKey": "...",
  "senderPhoneNumber": "0215881234"
}
```

### 근거

- 각 Organization이 업체와 직접 계약하므로 조직별 독립 Credential 저장이 필수다.
- 두 업체의 Credential 구조가 이질적이므로 JSON 컬럼이 가장 유연하게 수용 가능하다.
- Organization 테이블을 오염시키지 않기 위해 별도 테이블로 분리한다.
- Credential은 AES-256으로 암호화하여 저장한다. DB 접근 권한이 탈취되더라도 평문 노출을 방지한다.
- 타입 검증은 애플리케이션 레이어(DTO 유효성 검사)에서 처리한다.

### 향후 고려사항

- 암호화 키 관리가 중요해지는 시점에 AWS KMS 등 외부 키 관리 시스템 도입을 검토한다.
- Organization당 복수 업체 설정이 필요해지면 `UNIQUE` 제약을 제거하고 복수 레코드를 허용한다.

---

## ADR-003. 발송 내역(MessageLog) 테이블 설계

### 고민

업체가 다변화되면서 두 가지 문제가 생긴다:

1. **어떤 업체로 발송됐는지** 기록이 필요한가?
2. 두 업체의 응답 데이터 구조가 다른데, 이를 **하나의 테이블에 저장**할 것인가, **분리**할 것인가?

두 업체 응답의 실제 차이:

| 항목 | 클라우드메시지 | 센드톡 |
|------|--------------|--------|
| 발송 ID 필드명 | `message.id` | `data.messageId` |
| 과금 단위 | `billing.units`, `billing.unitPrice` | `data.chargeAmount` |
| 상태 이력 | `statusHistory[]` (배열) | 단일 상태값 |
| 초기 상태값 | `QUEUED` | `ACCEPTED` |

- **대안 1: 단일 테이블 + nullable 컬럼** — 모든 필드를 하나의 테이블에 두고 타 업체 컬럼은 NULL
  - 단점: 업체가 늘어날수록 NULL 컬럼이 증가. 테이블이 비대해짐.

- **대안 2: 공통 기본 테이블 + 업체별 상세 테이블**
  - 공통 필드(org_id, 수신자, 발송 시각 등)는 기본 테이블에
  - 업체별 고유 필드는 별도 테이블에서 FK로 연결
  - 장점: 각 업체의 데이터 구조를 그대로 보존. 공통 조회는 기본 테이블 단독으로 가능.

- **대안 3: 업체별 완전 독립 테이블**
  - 단점: 전체 발송 내역 조회 시 UNION 필요. 공통 조회 비효율.

### 선택 → 대안 2 (공통 기본 테이블 + 업체별 상세 테이블) + provider_type 반정규화

```
message_logs (공통)
- id
- org_id
- provider_type  ENUM('cloud_message', 'sendtalk')  ← 반정규화
- message_type   ENUM('SMS', 'MMS', 'ALIMTALK')
- recipient
- sent_at
- status

cloud_message_logs (클라우드메시지 전용)
- id
- message_log_id  → message_logs.id
- request_id
- billing_units
- billing_unit_price
- status_history  JSON

sendtalk_logs (센드톡 전용)
- id
- message_log_id  → message_logs.id
- message_id
- charge_amount
```

### 근거

**테이블 분리 이유**
- 두 업체의 응답 데이터가 구조적으로 달라 하나의 테이블에 넣으면 NULL 컬럼이 다수 발생한다.
- 업체별 테이블 분리로 각 업체의 데이터 구조를 그대로 보존할 수 있다.

**provider_type 반정규화 이유**
- `provider_type`은 원래 `organization_message_providers` 테이블에서 관리되는 값이다.
- 반정규화 없이 로그를 업체별로 필터링하려면 매번 `organization_message_providers` 테이블과 JOIN이 필요하다.
- `message_logs`에 직접 기록하면 단순 WHERE 조건으로 조회 가능하다.
- **추가 이점**: 조직이 업체를 변경한 이후에도 **발송 당시 어떤 업체를 사용했는지** 정확히 보존된다. `organization_message_providers`의 현재 값이 변경되더라도 발송 시점의 업체가 기록에 남는다.

### 향후 고려사항

- 업체가 늘어날수록 상세 테이블도 늘어난다. 업체 수가 많아지면 JSON 컬럼 방식으로의 전환을 검토한다.

---

## ADR-004. 기술 스택 선택

### 고민

서버 구현에 사용할 기술 스택을 선택한다. 주요 대안은 다음과 같다:

| | **Spring Boot 3.x + Java 21** | NestJS |
|---|---|---|
| 트랜잭션 관리 | `@Transactional` 선언적·전파 기반 관리, 검증된 생태계 | ORM(TypeORM)에 트랜잭션 관리 의존, Spring 수준의 선언적·전파 기반 트랜잭션 관리 기능은 제한적 |
| HMAC-SHA256 등 CPU 집약 암호화 처리 | 스레드 풀 분리로 I/O · CPU 작업 분리 가능 | Worker Threads로 회피 가능하나 별도 구성 필요 |
| 배치 처리 | Spring Batch — 재시도, 청크, 잡 관리 내장 | 별도 라이브러리 필요 (Bull 등) |
| 대규모 I/O 처리 | Virtual Threads (Java 21) 지원 | 이벤트 루프 기반 비동기 모델 |

### 선택 → Spring Boot 3.x + Java 21 + JPA + RestClient

### 근거

**① 트랜잭션 관리**
메시지 발송 요청 수신 → 업체 API 호출 → 발송 내역 DB 저장 흐름에서 DB 작업의 원자성 보장이 필요하다. Spring의 `@Transactional`은 선언적·전파 기반으로 트랜잭션 경계를 관리하며, 업체 설정 변경(Organization 업데이트)과 같은 작업도 일관성 있게 처리할 수 있다. NestJS는 ORM(TypeORM)에 트랜잭션 관리가 의존하며, Spring 수준의 선언적·전파 기반 트랜잭션 관리 기능은 제한적이다.

**② HMAC-SHA256 암호화 CPU 부하 처리**
클라우드메시지는 매 요청마다 HMAC-SHA256 서명을 생성한다. 이는 CPU-bound 작업으로, I/O-bound 작업(HTTP 호출)과 성격이 다르다. Spring은 스레드 풀을 목적에 따라 분리 구성할 수 있어 HMAC 서명 생성(CPU-bound)과 HTTP 발송(I/O-bound)을 자연스럽게 독립 처리할 수 있다. NestJS는 Worker Threads로 회피 가능하나 별도 구성이 필요하다.

**③ 배치 처리 (Spring Batch)**
대량 메시지 발송은 배치 작업으로 처리되는 경우가 많다. Spring Batch는 청크 기반 처리, 실패 시 재시도, 잡 이력 관리 등을 내장하고 있어 별도 구현 없이 강력한 배치 처리가 가능하다.

**④ Virtual Threads + RestClient**
Java 21의 Virtual Threads를 사용하면 RestClient(동기 방식)의 blocking I/O가 carrier thread를 점유하지 않는다. 비동기 방식과 유사한 처리량을 달성하면서도 **동기 코드의 단순함을 그대로 유지**할 수 있다는 것이 핵심이다.

```
# Virtual Threads 활성화 (Spring Boot 3.2+)
spring.threads.virtual.enabled=true
```

**⑤ 친숙도**
Spring 생태계에 대한 깊은 이해를 보유하고 있어 설계 의도를 코드로 빠르게 구현할 수 있다. 기술 선택의 불확실성을 줄이고 설계 품질에 집중할 수 있다.

### 최종 스택 요약

| 항목 | 선택 | 이유 |
|------|------|------|
| Framework | Spring Boot 3.x | 트랜잭션, DI, Batch 내장 |
| Language | Java 21 | Virtual Threads 지원 |
| ORM | JPA (Hibernate) | Enum 매핑, 관계 매핑 편리 |
| HTTP Client | RestClient | Virtual Threads와 조합 시 동기 코드로 고성능 달성 |
| DB | MySQL | ENUM, JSON 컬럼 네이티브 지원 |

### 향후 고려사항

- **Spring Batch → 메시지 큐 전환**: 실시간 발송 요건이 생기거나 10초 지연이 문제가 되는 시점에 Kafka 또는 Redis Streams 도입을 검토한다.
- **RestClient → Reactive Stack 전환**: 발송 볼륨이 폭발적으로 증가해 현재 처리량으로 감당이 어려워지는 시점에 WebClient + R2DBC(완전 비동기) 전환을 검토한다.

---

## ADR-005. outbox_messages.payload JSON 구조

### 고민

Batch ItemProcessor가 `outbox_messages`를 읽어 Provider에 발송을 위임할 때, payload에 담긴 데이터를 어떻게 구조화할 것인가? 두 가지 이슈가 있었다.

**이슈 1: 공통 구조 vs 업체별 구조**

두 업체의 알림톡 필드명이 다르다 (`templateId`/`params` vs `templateCode`/`variables`). payload를 업체별 원본 형식으로 담을지, 공통 구조로 담을지가 미결이었다.

**이슈 2: 1건 = 1수신자 vs 1건 = N수신자**

outbox 1 row에 수신자를 1명만 담을지, 배열로 N명을 담을지가 미결이었다.

### 선택

- 이슈 1 → **DTO 직렬화 + 인프라 레이어 역직렬화**
- 이슈 2 → **1건 = 1수신자**

### 근거

**이슈 1**

payload를 업체별로 다르게 설계하면 MessageService가 어떤 업체가 설정되어 있는지 알고 저장해야 한다. 이는 ADR-001 원칙("업체 차이는 Provider 내부에서 흡수한다")에 위배된다.

대신 `MessageService`가 받은 발송 요청 DTO를 그대로 JSON 직렬화해서 저장한다. Batch ItemProcessor(인프라 레이어)가 `messageType`을 기준으로 적절한 DTO로 역직렬화하고, Provider가 자신의 API 형식으로 매핑한다. 업체별 필드명 변환(`templateKey`→`templateId`/`templateCode`, `variables`→`params`/`variables`)은 Provider 내부에서 처리한다.

이 방식은 MessageService를 업체 변경 영향에서 완전히 격리하며, 추가 설계 없이 기존 DTO 구조를 재사용한다.

**이슈 2**

발송 신뢰성 보장(Outbox Pattern 핵심 목적)을 위해 수신자별 상태 추적이 필요하다. 1건 = 1수신자로 설계하면 수신자별 SENT/FAILED 상태 관리와 개별 재시도가 단순해진다. 1건 = N수신자 구조는 부분 실패(일부 수신자만 실패) 처리 및 재시도 단위가 복잡해진다.

### 처리 흐름 요약

```
MessageService
  └─ SendMessageRequest DTO → JSON 직렬화 → outbox_messages.payload 저장

Batch ItemProcessor
  └─ payload 읽기
       └─ messageType 기준 역직렬화
            ├─ SMS / MMS   → SmsMessageDto
            └─ ALIMTALK    → AlimtalkMessageDto
                                ↓
                        Provider.send(dto)
                          ├─ CloudMessageProvider: templateKey→templateId, variables→params
                          └─ SendtalkProvider: templateKey→templateCode, variables→variables
```

### 운영 주의사항

조직이 업체를 전환하면 PENDING 상태인 outbox 건들의 `templateKey` 값이 신규 업체의 템플릿 코드와 맞지 않아 발송 실패 가능성이 있다. 업체 전환 전 PENDING 건 소진 여부를 확인하거나, PENDING 건을 FAILED 일괄 처리 후 전환하는 운영 절차가 필요하다.
