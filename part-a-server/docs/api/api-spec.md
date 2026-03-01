[← Part A 홈](../../README.md)

`Path: part-a-server/docs/api/api-spec.md`

# API 명세

## 공통 사항

**Base URL** (가정)
```
/api/v1
```

**인증** (가정 — [A-05 참고](../requirement/requirements.md#a-05-인증-및-권한-처리-방식))
```
Authorization: Bearer {JWT token}
```
모든 엔드포인트는 `role = ADMIN`이고 `token.orgId = {orgId}`인 경우에만 접근 가능하다.

**공통 에러 응답**

| HTTP Status | 설명 |
|-------------|------|
| 401 | 인증 토큰 없음 또는 만료 |
| 403 | 권한 없음 (다른 조직의 리소스 접근 시도) |
| 404 | 조직을 찾을 수 없음 |
| 500 | 서버 내부 오류 |

> **실제 발송의 Rate Limit · Daily Limit 초과**: 발송은 Outbox에 저장 후 Spring Batch가 비동기로 처리하므로, 업체 API의 Rate Limit(429)이나 Daily Limit 초과는 클라이언트 응답에 직접 반영되지 않는다. 해당 발송 건은 `outbox_messages.status = FAILED`로 기록되며, 발송 내역에서 확인할 수 있다. 429는 벤더 API를 동기 호출하는 **연결 테스트(`POST /test`) 전용** 응답 코드다.

---

## 엔드포인트 목록

| 우선순위 | 메서드 | 경로 | 설명 |
|----------|--------|------|------|
| 필수 | `GET` | `/organizations/{orgId}/message-provider` | 현재 업체 설정 조회 |
| 필수 | `PUT` | `/organizations/{orgId}/message-provider` | 업체 변경 |
| 낮음 | `POST` | `/organizations/{orgId}/message-provider/test` | 업체 연결 테스트 |

---

## GET `/organizations/{orgId}/message-provider`

현재 조직에 설정된 메시지 전송 업체를 조회한다.

### Path Parameters

| 파라미터 | 타입 | 설명 |
|----------|------|------|
| orgId | Long | 조직 ID |

### Response `200 OK`

업체별로 반환되는 `credentials` 필드가 다르다.

```json
// 클라우드메시지인 경우
{
  "orgId": 1,
  "providerType": "cloud_message",
  "credentials": {
    "accessKey": "AKCL••••••••••••",
    "secretKey": "secr••••••••••••",
    "senderProfileId": "PROF••••••••••••",
    "senderPhoneNumber": "0215881234"
  },
  "updatedAt": "2024-01-15T10:30:00"
}

// 센드톡인 경우
{
  "orgId": 1,
  "providerType": "sendtalk",
  "credentials": {
    "apiKey": "abcd••••••••••••",
    "secretKey": "secr••••••••••••",
    "senderPhoneNumber": "0215881234"
  },
  "updatedAt": "2024-01-15T10:30:00"
}
```

| 필드 | 타입 | 설명 |
|------|------|------|
| orgId | Long | 조직 ID |
| providerType | String | 현재 설정된 업체 (`cloud_message` \| `sendtalk`) |
| credentials | Object | 업체별 설정값. **민감 필드(key류)는 앞 4자리만 표시 후 `••••••••••••` 마스킹** |
| credentials.senderPhoneNumber | String | 발신번호. 마스킹 없이 그대로 반환 |
| updatedAt | DateTime | 마지막 변경 시각 |

> **마스킹 규칙**: 민감 필드는 서버에서 앞 4자리 + `••••••••••••` 형태로 가공하여 반환한다. 클라이언트는 받은 값을 그대로 표시한다.

### Response `404 Not Found`

조직에 업체 설정이 아직 등록되지 않은 경우.

```json
{
  "code": "PROVIDER_NOT_CONFIGURED",
  "message": "업체 설정이 등록되지 않았습니다."
}
```

---

## PUT `/organizations/{orgId}/message-provider`

조직의 메시지 전송 업체를 변경한다. 변경은 **다음 발송 요청부터** 적용된다.

### Path Parameters

| 파라미터 | 타입 | 설명 |
|----------|------|------|
| orgId | Long | 조직 ID |

### Request Body

업체 타입에 따라 `credentials` 내 필드가 달라진다.

```json
// 클라우드메시지로 설정 시
{
  "providerType": "cloud_message",
  "credentials": {
    "accessKey": "AKCLOUDMSG...",
    "secretKey": "...",
    "senderProfileId": "PROFILE_...",
    "senderPhoneNumber": "0215881234"
  }
}

// 센드톡으로 설정 시
{
  "providerType": "sendtalk",
  "credentials": {
    "apiKey": "...",
    "secretKey": "...",
    "senderPhoneNumber": "0215881234"
  }
}
```

| 필드 | 타입 | 필수 | 설명 |
|------|------|------|------|
| providerType | String | Y | 변경할 업체 (`cloud_message` \| `sendtalk`) |
| credentials.accessKey | String | cloud_message 필수 | 클라우드메시지 AccessKey |
| credentials.secretKey | String | Y | 서명 또는 웹훅 검증용 SecretKey |
| credentials.senderProfileId | String | cloud_message 필수 | 알림톡 발신 프로필 ID |
| credentials.senderPhoneNumber | String | Y | 발신번호 |
| credentials.apiKey | String | sendtalk 필수 | 센드톡 Bearer Token (인증 헤더에 사용) |
| credentials.secretKey | String | sendtalk 필수 | Webhook 수신 결과 서명 검증용 (HMAC-SHA256) |

### Response `200 OK`

```json
{
  "orgId": 1,
  "providerType": "sendtalk",
  "updatedAt": "2024-01-20T14:00:00"
}
```

### Error Response

```json
{
  "code": "INVALID_PROVIDER_TYPE",
  "message": "지원하지 않는 업체입니다.",
  "allowedValues": ["cloud_message", "sendtalk"]
}
```

| HTTP Status | 코드 | 설명 |
|-------------|------|------|
| 400 | `INVALID_PROVIDER_TYPE` | 존재하지 않는 업체값 입력 |
| 400 | `MISSING_CREDENTIALS_FIELD` | 업체 필수 Credential 필드 누락 |

### 동작 원칙

- 현재와 동일한 업체로 변경 요청 시 오류 없이 `200 OK` 반환 (멱등성 보장)
- 진행 중인 발송 건은 기존 업체로 계속 처리됨 ([A-07 참고](../requirement/requirements.md#a-07-업체-변경-시-진행-중인-발송-처리))

---

## POST `/organizations/{orgId}/message-provider/test`

> ⚠️ **우선순위: 낮음** — 초기 구현 범위 제외. Credential이 조직별 DB에 저장되므로, 이 엔드포인트는 **해당 조직의 Credential로 실제 업체 API 연결을 검증**한다.

설정된 업체의 API 연결 상태를 검증한다. 실제 메시지를 발송하지 않고 인증 및 연결 여부만 확인한다.

> 이 엔드포인트는 업체 API를 **동기 호출**하므로, 공통 에러 응답 외에 `429 Too Many Requests`가 추가로 발생할 수 있다. 업체 API Rate Limit 초과 시 `Retry-After` 헤더를 참고한다.

### Path Parameters

| 파라미터 | 타입 | 설명 |
|----------|------|------|
| orgId | Long | 조직 ID |

### Response `200 OK`

```json
{
  "providerType": "sendtalk",
  "success": true,
  "message": "연결 성공"
}
```

### Response `200 OK` (연결 실패)

```json
{
  "providerType": "sendtalk",
  "success": false,
  "providerErrorCode": "UNAUTHORIZED",
  "message": "인증 실패: API Key가 유효하지 않습니다."
}
```

| 필드 | 타입 | 설명 |
|------|------|------|
| success | Boolean | 연결 성공 여부 |
| providerErrorCode | String | 업체 API가 반환한 에러 코드. 성공 시 null |
| message | String | 사람이 읽을 수 있는 결과 메시지 |

**[설계 결정] 연결 실패 시 HTTP 200 반환**

| 항목 | 내용 |
|------|------|
| 고민한 것 | 외부 업체 연결 실패 시 어떤 HTTP 상태 코드로 응답할 것인가 |
| 선택한 것 | `200 OK` + `success: false` |
| 근거 | HTTP 상태 코드는 "우리 서버와의 통신 성공 여부"를 나타낸다. 연결 테스트 요청은 우리 서버에 정상 도달했고 처리됐으므로 200이 의미적으로 정확하다. 외부 업체 API의 결과는 별개 레이어의 문제이며 `success` 필드로 구분한다. 클라이언트는 성공·실패 모두 동일한 응답 구조로 처리할 수 있어 분기 로직이 단순해진다. |

**고려한 대안**

| 대안 | 선택하지 않은 이유 |
|------|-------------------|
| `503 Service Unavailable` | 우리 서버가 다운된 것처럼 오해됨. 실제로는 외부 업체 API의 문제 |
| `400 Bad Request` | 요청 형식 자체가 잘못된 것처럼 보임. 실제 원인은 외부 업체 인증 실패이므로 의미가 어긋남 |

---

## 업체별 에러 코드 참조

우리 서버가 업체 API를 호출할 때 수신할 수 있는 에러 코드다.
`providerErrorCode` 필드로 클라이언트에 전달되며, Outbox Batch의 재시도 전략 판단에도 사용된다.

### 센드톡 (sendtalk)

| 코드 | 설명 | 발생 시점 | 서버 처리 방침 |
|------|------|-----------|--------------|
| `UNAUTHORIZED` | API Key 인증 실패 | 연결 테스트, 발송 | 재시도 없이 즉시 FAILED |
| `INVALID_PHONE_NUMBER` | 유효하지 않은 전화번호 | 연결 테스트, 발송 | 재시도 없이 즉시 FAILED |
| `INSUFFICIENT_BALANCE` | 잔액 부족 | 발송 | 재시도 없이 즉시 FAILED |
| `INVALID_TEMPLATE` | 유효하지 않은 템플릿 코드 | 발송 | 재시도 없이 즉시 FAILED |
| `RATE_LIMIT_EXCEEDED` | 요청 한도 초과 | 연결 테스트, 발송 | 지수 백오프 재시도 (최대 3회) |
| `INTERNAL_ERROR` | 센드톡 서버 내부 오류 | 연결 테스트, 발송 | 지수 백오프 재시도 (최대 3회) |

> **재시도 불가 에러** (`UNAUTHORIZED`, `INVALID_PHONE_NUMBER`, `INSUFFICIENT_BALANCE`, `INVALID_TEMPLATE`):
> 재시도해도 결과가 바뀌지 않는 확정적 실패다. 즉시 FAILED 처리하여 불필요한 업체 API 호출을 방지한다.
>
> **재시도 가능 에러** (`RATE_LIMIT_EXCEEDED`, `INTERNAL_ERROR`):
> 일시적 상태이므로 지수 백오프 + Jitter 적용 후 재시도한다. ([Outbox Batch 설계 참고](../architecture/outbox-batch-design.md))

### 클라우드메시지 (cloud_message)

| 코드 | HTTP | 설명 | 발생 시점 | 서버 처리 방침 |
|------|------|------|-----------|--------------|
| `AUTHENTICATION_FAILED` | 401 | API Key 인증 실패 | 연결 테스트, 발송 | 재시도 없이 즉시 FAILED |
| `INVALID_SIGNATURE` | 401 | HMAC-SHA256 서명 검증 실패 | 연결 테스트, 발송 | 재시도 없이 즉시 FAILED |
| `INVALID_RECIPIENT` | 400 | 수신자 정보 오류 | 발송 | 재시도 없이 즉시 FAILED |
| `INVALID_SENDER` | 400 | 발신번호 미등록 | 연결 테스트, 발송 | 재시도 없이 즉시 FAILED |
| `INVALID_TEMPLATE` | 400 | 템플릿 없음 또는 미승인 | 발송 | 재시도 없이 즉시 FAILED |
| `TEMPLATE_PARAM_MISMATCH` | 400 | 템플릿 변수 불일치 | 발송 | 재시도 없이 즉시 FAILED |
| `INSUFFICIENT_CREDIT` | 402 | 크레딧 부족 | 발송 | 재시도 없이 즉시 FAILED |
| `QUOTA_EXCEEDED` | 429 | 발송 한도 초과 | 연결 테스트, 발송 | 지수 백오프 재시도 (최대 3회) |
| `SERVICE_UNAVAILABLE` | 503 | 일시적 서비스 장애 | 연결 테스트, 발송 | 지수 백오프 재시도 (최대 3회) |

> **재시도 불가 에러** (`AUTHENTICATION_FAILED`, `INVALID_SIGNATURE`, `INVALID_RECIPIENT`, `INVALID_SENDER`, `INVALID_TEMPLATE`, `TEMPLATE_PARAM_MISMATCH`, `INSUFFICIENT_CREDIT`):
> 확정적 실패. 즉시 FAILED 처리.
>
> **재시도 가능 에러** (`QUOTA_EXCEEDED`, `SERVICE_UNAVAILABLE`):
> 일시적 상태. 지수 백오프 + Jitter 적용 후 재시도. ([Outbox Batch 설계 참고](../architecture/outbox-batch-design.md))
