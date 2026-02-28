[← Part A 홈](../../README.md)

`Path: part-a-server/docs/architecture/provider-design.md`

# Provider 추상화 레이어 설계

조직별 메시지 업체 분기 로직의 구체적인 설계입니다.

> 추상화 수준 결정 배경 — [ADR-001](adr.md#adr-001-provider-추상화-수준) 참고

---

## 1. 전체 구조

```
MessageService
  └── ProviderFactory                  ← 업체 선택만 책임
        ├── CloudMessageProvider       ← 클라우드메시지 API 전담
        └── SendtalkProvider           ← 센드톡 API 전담
```

**설계 원칙**: 업체 선택만 추상화. 각 Provider는 자신의 업체 API에만 의존하며 독립적으로 동작한다.

---

## 2. ProviderFactory

### 책임

`organization_message_providers` 테이블에서 해당 조직의 `provider_type`과 암호화된 `credentials`를 읽어, 복호화 후 맞는 Provider 인스턴스를 반환한다. **업체 선택 로직은 오직 이곳에만 존재한다.**

### 동작 흐름

```
ProviderFactory.getProvider(outboxMessage)
  ↓
  1. outboxMessage.provider_type 읽기 (저장 시점에 고정된 값)
  ↓
  2. organization_message_providers WHERE org_id = {orgId} 조회 (credentials만 사용)
     └─ 없으면 → ProviderNotConfiguredException 발생
  ↓
  3. CredentialEncryptionService.decrypt(credentials)
     └─ 복호화 실패(변조 감지) → CredentialDecryptionException 발생
  ↓
  4. provider_type에 따라 분기
     ├─ cloud_message → CloudMessageProvider(credential) 반환
     └─ sendtalk      → SendtalkProvider(credential) 반환
```

### 예외 처리 원칙

| 상황 | 처리 방식 |
|------|-----------|
| 업체 미설정 조직 | 예외 발생 → 발송 FAILED 처리 |
| Credential 복호화 실패 | 예외 발생 → 발송 FAILED 처리, 알림 필요 |
| 지원하지 않는 provider_type | 예외 발생 (ENUM 타입이므로 DB 레벨에서 사전 방어) |

---

## 3. CloudMessageProvider

### 책임

클라우드메시지 API를 호출하는 모든 로직을 담당한다. HMAC-SHA256 서명 생성, 전화번호 변환, 요청/응답 처리를 포함한다.

### HMAC-SHA256 서명 생성 흐름

클라우드메시지 API는 매 요청마다 HMAC-SHA256 서명을 요구한다.

```
서명 생성 흐름
  1. 서명 대상 문자열 구성: "{HTTP Method}\n{Path}\n{Timestamp}\n{AccessKey}"
  2. SecretKey로 HMAC-SHA256 서명 생성
  3. Base64 인코딩
  4. 요청 헤더에 포함:
       X-CM-Access-Key: {accessKey}
       X-CM-Timestamp:  {timestamp}
       X-CM-Signature:  {base64-encoded-signature}
```

> **CPU-bound 작업**: HMAC 서명 생성은 CPU 집약 작업이다. Virtual Threads 환경에서 I/O 대기(HTTP 발송)와 혼재하지 않도록 스레드 풀을 분리 구성할 수 있다. ([ADR-004 참고](adr.md#adr-004-기술-스택-선택))

### 전화번호 변환

클라우드메시지 API는 E.164 형식을 요구한다 (`01012345678` → `+821012345678`).

```
전화번호 변환
  010XXXXXXXX → +8210XXXXXXXX
  0XXXXXXXXXX → +82XXXXXXXXX  (선행 0 제거 후 +82 추가)
```

이 변환 로직은 CloudMessageProvider 내부에서만 처리한다. MessageService는 이 변환을 알지 못한다.

### 메시지 타입별 엔드포인트

| 메시지 타입 | 엔드포인트 |
|------------|-----------|
| SMS | `POST /sms` |
| MMS | `POST /mms` |
| 알림톡 | `POST /alimtalk` |

---

## 4. SendtalkProvider

### 책임

센드톡 API를 호출하는 모든 로직을 담당한다. Bearer 토큰 인증 헤더 설정 및 요청/응답 처리를 포함한다.

### 인증 방식

```
요청 헤더
  Authorization: Bearer {apiKey}
  Content-Type: application/json
```

HMAC 서명 없이 API Key를 Bearer 토큰으로 직접 사용한다.

### 메시지 타입별 엔드포인트

| 메시지 타입 | 엔드포인트 |
|------------|-----------|
| SMS | `POST /messages/sms` |
| MMS | `POST /messages/mms` |
| 알림톡 | `POST /messages/alimtalk` |

### 응답 처리

센드톡 응답은 `{ success, data }` 래퍼 구조다. `success: false` 응답은 HTTP 200이더라도 발송 실패로 처리한다.

```
응답 처리 흐름
  HTTP 200 + success: true  → SENT
  HTTP 200 + success: false → FAILED (업체 측 거부)
  HTTP 4xx              → FAILED (재시도 불필요)
  HTTP 5xx / 타임아웃   → 재시도 대상
```

---

## 5. 업체별 주요 차이점 요약

| 항목 | 클라우드메시지 | 센드톡 |
|------|--------------|--------|
| 인증 | HMAC-SHA256 서명 (매 요청 생성) | Bearer API Key |
| 템플릿 식별자 | `templateId` | `templateCode` |
| 템플릿 변수 | `params` | `variables` |
| 발송 직후 상태 | `QUEUED` | `ACCEPTED` |
| 응답 구조 | 직접 객체 반환 | `{ success, data }` 래퍼 |
| 전화번호 형식 | E.164 필요 (`+821012345678`) | 국내 형식 (`01012345678`) |

각 Provider가 이 차이를 내부적으로 흡수한다. MessageService는 Provider 구현 세부사항을 알지 못한다.

---

## 6. MessageService 관점

```
MessageService (발송 요청 처리)
  ↓
  outbox_messages 저장 (PENDING)   ← 트랜잭션 범위 끝

  ─ ─ ─ ─ ─ [Spring Batch] ─ ─ ─ ─ ─

  ProviderFactory.getProvider(orgId)
  → provider.send(outboxMessage)
```

MessageService는 Provider가 무엇인지, 어떻게 동작하는지 모른다. Outbox에 저장하는 것까지만 책임진다.
