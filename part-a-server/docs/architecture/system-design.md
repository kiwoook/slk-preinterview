[← Part A 홈](../../README.md)

`Path: part-a-server/docs/architecture/system-design.md`

# 시스템 설계

## 1. 변경 전 아키텍처

```
[발송 요청 (orgId 포함)]
         ↓
 [MessageService]
         ↓
 [CloudMessageClient]        ← 단일 업체 직접 호출, 추상화 없음
         ↓
 [클라우드메시지 API]
         ↓
 [발송 내역 DB 저장]          ← 동기, 단일 테이블
```

**문제점**
- 업체 호출 로직이 MessageService에 직접 결합되어 있어 업체 변경 시 서비스 코드 수정 필요
- API 호출 실패 시 발송 내역 손실 위험 (재시도 메커니즘 없음)
- 단일 업체 전제로 설계되어 조직별 업체 분기 불가

---

## 2. 변경 후 아키텍처

### 전체 흐름

```
[발송 요청 (orgId 포함)]
         ↓
 [MessageService] ─── @Transactional ───┐
         ↓                               │
 [outbox_messages 저장 (PENDING)]  ◄────┘
         ↓ (트랜잭션 종료)


 ┌─── [Spring Batch Scheduler] ────────────────────────────┐
 │                                                          │
 │  ItemReader                                              │
 │  └─ outbox_messages WHERE status = PENDING (청크 단위)   │
 │         ↓                                                │
 │  ItemProcessor                                                   │
 │  └─ [ProviderFactory]                                            │
 │       └─ outbox_messages.provider_type 사용 (저장 시점 고정값)   │
 │            └─ organization_message_providers 조회 (credentials만) │
 │            ├─ cloud_message → [CloudMessageProvider]             │
 │            │                  └─ credentials 복호화              │
 │            │                  └─ HMAC 서명 생성                  │
 │            │                  └─ 클라우드메시지 API 호출          │
 │            └─ sendtalk    → [SendtalkProvider]                   │
 │                             └─ credentials 복호화                │
 │                             └─ Bearer 토큰 헤더 설정             │
 │                             └─ 센드톡 API 호출                   │
 │         ↓                                                │
 │  ItemWriter                                              │
 │  ├─ outbox_messages 상태 → SENT / FAILED 업데이트        │
 │  └─ @Async → [message_logs + 업체별 상세 로그 저장]      │
 │                                                          │
 └──────────────────────────────────────────────────────────┘
```

### 핵심 설계 원칙

| 원칙 | 적용 방식 |
|------|-----------|
| 발송 신뢰성 보장 | Outbox Pattern — 발송 요청을 DB에 먼저 저장, 실패 시 재시도 |
| 업체 분리 | Strategy Pattern — ProviderFactory가 업체 선택, 각 Provider가 독립 처리 |
| 트랜잭션 점유 최소화 | 발송 요청 저장까지만 트랜잭션. 로그 저장은 @Async |
| DB 쓰기 부하 분산 | Spring Batch 청크 처리 — PENDING 건을 N개씩 묶어 처리 |

---

## 3. Outbox Pattern 상세

### outbox_messages 테이블

```sql
CREATE TABLE outbox_messages (
    id            BIGINT          NOT NULL AUTO_INCREMENT,
    org_id        BIGINT          NOT NULL,
    provider_type ENUM('cloud_message', 'sendtalk') NOT NULL,  -- 저장 시점의 업체 고정. 이후 업체 변경 영향 차단.
    message_type  ENUM('SMS', 'MMS', 'ALIMTALK') NOT NULL,
    payload       JSON            NOT NULL,   -- SendMessageRequest DTO 직렬화 값. Batch 인프라 레이어에서 역직렬화.
    status        ENUM('PENDING', 'SENT', 'FAILED') NOT NULL DEFAULT 'PENDING',
    retry_count   INT             NOT NULL DEFAULT 0,
    created_at    DATETIME        NOT NULL DEFAULT CURRENT_TIMESTAMP,
    processed_at  DATETIME,
    PRIMARY KEY (id),
    INDEX idx_status_created (status, created_at)  -- Batch 조회 최적화
);
```

### payload JSON 구조

payload는 `MessageService`가 받은 발송 요청 DTO를 그대로 직렬화한 값이다. 업체별 필드명 차이는 Provider 레이어에서 흡수한다.

**수신자 단위**: 1건 = 1수신자. 수신자별 SENT/FAILED 추적과 개별 재시도를 위해 row per recipient로 저장한다.

```json
// SMS
{
  "recipient": "01012345678",
  "messageType": "SMS",
  "content": "인증번호는 [123456]입니다."
}

// MMS
{
  "recipient": "01012345678",
  "messageType": "MMS",
  "subject": "공지사항",
  "content": "본문 내용입니다."
}

// 알림톡
{
  "recipient": "01012345678",
  "messageType": "ALIMTALK",
  "templateKey": "ORDER_CONFIRM_001",
  "variables": {
    "name": "홍길동",
    "orderNo": "20240101-001",
    "amount": "15,000"
  }
}
```

**역직렬화 흐름 (Batch ItemProcessor)**

```
outbox_messages.messageType
  ├─ SMS / MMS   → SmsMessageDto (content, subject)
  └─ ALIMTALK    → AlimtalkMessageDto (templateKey, variables)
                      ↓
              Provider.send(dto)
```

**Provider 변환 매핑**

| payload 필드 | CloudMessageProvider | SendtalkProvider |
|---|---|---|
| `templateKey` | → `templateId` | → `templateCode` |
| `variables` | → `params` | → `variables` (동일) |
| `recipient` | E.164 변환 (`+821012345678`) | 그대로 (`01012345678`) |

> 설계 근거 → [ADR-005](adr.md#adr-005-outbox_messagespayload-json-구조)

---

### Batch 청크 처리 흐름

```
Scheduler (주기적 실행)
  └─ Job: OutboxProcessingJob
       └─ Step: OutboxProcessingStep
            ├─ ItemReader:     SELECT * FROM outbox_messages
            │                  WHERE status = 'PENDING'
            │                  ORDER BY created_at
            │                  LIMIT {chunkSize}          ← 청크 단위로 읽기
            │
            ├─ ItemProcessor:  ProviderFactory로 업체 선택 → API 호출
            │                  성공 → status = SENT
            │                  실패 → retry_count + 1
            │                         retry_count >= MAX → status = FAILED
            │
            └─ ItemWriter:     outbox 상태 일괄 업데이트
                               @Async → 로그 저장 트리거
```

### 재시도 정책 (지수 백오프)

| 항목 | 값 | 근거 |
|------|-----|------|
| 최대 재시도 횟수 | 3회 | 일시적 장애 대응. 초과 시 FAILED로 전환하여 무한 루프 방지 |
| 재시도 대상 | HTTP 5xx, 타임아웃 | 업체 측 일시 장애. 4xx는 요청 자체가 잘못된 것이므로 재시도 불필요 |
| 백오프 전략 | 지수 백오프 + Jitter | 재시도 간격을 지수적으로 늘려 외부 API 부하 집중 방지. Jitter로 동시 재시도 분산 |

---

## 4. DB 스키마 변경

### 4-1. organization_message_providers 테이블 신규 생성

각 조직이 자체 업체 계약을 보유하므로, provider_type과 Credential을 조직별로 저장한다.

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

**credentials 암호화 JSON 구조 (복호화 후)**

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

**마이그레이션 전략**
- 기존 조직은 마이그레이션 스크립트로 `cloud_message` + 기존 전역 설정값을 기본값으로 삽입
- 마이그레이션 완료 후 기존 전역 환경변수 제거

### 4-2. 발송 내역 테이블 신규 구성

```sql
-- 공통 기본 테이블
CREATE TABLE message_logs (
    id            BIGINT      NOT NULL AUTO_INCREMENT,
    org_id        BIGINT      NOT NULL,
    provider_type ENUM('cloud_message', 'sendtalk') NOT NULL,  -- 반정규화
    message_type  ENUM('SMS', 'MMS', 'ALIMTALK')   NOT NULL,
    recipient     VARCHAR(20) NOT NULL,
    status        VARCHAR(20) NOT NULL,
    sent_at       DATETIME    NOT NULL,
    PRIMARY KEY (id),
    INDEX idx_org_sent (org_id, sent_at),
    INDEX idx_provider (provider_type)          -- 업체별 조회 최적화
);

-- 클라우드메시지 전용 상세 테이블
CREATE TABLE cloud_message_logs (
    id                BIGINT          NOT NULL AUTO_INCREMENT,
    message_log_id    BIGINT          NOT NULL,
    request_id        VARCHAR(100),
    billing_units     INT,
    billing_unit_price DECIMAL(10,2),
    status_history    JSON,                     -- statusHistory 배열 보존
    PRIMARY KEY (id),
    FOREIGN KEY (message_log_id) REFERENCES message_logs(id)
);

-- 센드톡 전용 상세 테이블
CREATE TABLE sendtalk_logs (
    id             BIGINT         NOT NULL AUTO_INCREMENT,
    message_log_id BIGINT         NOT NULL,
    message_id     VARCHAR(100),
    charge_amount  DECIMAL(10,2),
    PRIMARY KEY (id),
    FOREIGN KEY (message_log_id) REFERENCES message_logs(id)
);
```

**provider_type 반정규화 이유**
- Organizations 테이블 JOIN 없이 업체별 로그 필터링 가능
- 조직이 업체를 변경한 이후에도 발송 당시 업체 정보를 정확히 보존

---

## 5. 로그 비동기 저장

```
Batch ItemWriter
  └─ outbox 상태 업데이트 (동기, 트랜잭션)
  └─ ApplicationEventPublisher.publishEvent(MessageSentEvent)
                                    ↓ @Async
                          [MessageLogEventHandler]
                            ├─ message_logs 저장
                            └─ cloud_message_logs
                               또는 sendtalk_logs 저장
```

**트레이드오프 기록**

| 항목 | 내용 |
|------|------|
| 장점 | Batch 트랜잭션 점유 시간 최소화. 발송 처리량에 로그 쓰기가 영향 주지 않음 |
| 단점 | 비동기 저장 실패 시 상세 로그 유실 가능 |
| 허용 근거 | 발송 성공 여부는 outbox로 보장. 상세 로그(과금 정보 등) 유실은 현 단계에서 허용 |
| 향후 전환 시점 | 발송 실패 원인 분석, 과금 정확도가 중요해지는 시점에 Redis Stream 또는 Kafka 도입 |

---

## 6. 로그 아카이빙 권장 사항

발송 내역(`message_logs`, `cloud_message_logs`, `sendtalk_logs`)은 시간이 지날수록 데이터가 빠르게 누적된다.

| 기간 | 저장 위치 | 비고 |
|------|-----------|------|
| 3개월 이내 | MySQL | 빠른 조회 필요 구간 |
| 3개월 이후 | S3 또는 외부 스토리지 이관 권장 | 운영 조회 빈도 낮음, DB 용량 관리 필요 |

- 3개월 기준은 일반적인 CS 대응 및 정산 조회 범위를 고려한 값이다. 서비스 특성에 따라 조정 가능.
- 이관 후 MySQL 원본 데이터는 삭제하거나 파티셔닝으로 분리해 DB 성능을 유지한다.
- S3 저장 시 Athena 등 쿼리 도구를 활용하면 아카이브 데이터도 필요 시 조회 가능하다.

---

## 7. 환경변수 구성

Credential은 조직별 DB에 저장되므로 환경변수에는 **시스템 운영에 필요한 값만** 관리한다.

```bash
# Credential 암호화 키 (AES-256)
CREDENTIAL_ENCRYPTION_KEY=...     # 반드시 외부 노출 금지

# Batch 설정
OUTBOX_BATCH_CHUNK_SIZE=100
OUTBOX_BATCH_SCHEDULER_CRON=0/10 * * * * ?    # 10초마다 실행
OUTBOX_BATCH_MAX_RETRY=3
```
