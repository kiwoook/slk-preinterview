[← Part A 홈](../../README.md)

`Path: part-a-server/docs/architecture/verification-rollback.md`

# 통합 검증 및 롤백 계획

배포 전후 검증 시나리오와 문제 발생 시 롤백 절차를 정의합니다.

---

## 1. 리스크 및 대응 전략

### R-01. V4 마이그레이션 실행 시 기존 전역 환경변수 누락

| 항목 | 내용 |
|------|------|
| 리스크 | V4 스크립트는 기존 전역 환경변수(API 키류)를 읽어 조직별 Credential을 암호화·삽입한다. 환경변수가 누락된 상태로 실행하면 삽입이 실패한다. |
| 영향 | 기존 조직 전체 발송 불가 — 마이그레이션 직후 서비스 중단 |
| 대응 전략 | 배포 전 체크리스트에 환경변수 존재 확인 필수화. V4 스크립트 시작 시 환경변수 누락 여부를 사전 검증하고, 없으면 즉시 중단(부분 실행 방지). |

---

### R-02. 암호화 키(`CREDENTIAL_ENCRYPTION_KEY`) 유실

| 항목 | 내용 |
|------|------|
| 리스크 | 암호화 키가 유실되면 DB에 저장된 모든 조직의 Credential을 복호화할 수 없다. |
| 영향 | 전체 조직 발송 불가. 기술적 복구 방법 없음 — 각 조직 관리자가 Credential을 재입력해야 한다. |
| 대응 전략 | 키를 환경변수에만 보관하지 않고 AWS Secrets Manager 등 외부 키 관리 시스템에 백업. 접근 권한 최소화. 키 교체(rotation) 정책 수립. |

---

### R-03. Batch 장애로 PENDING 건 누적 및 감지 실패

| 항목 | 내용 |
|------|------|
| 리스크 | Batch 스케줄러가 장애로 중단되면 `outbox_messages`의 PENDING 건이 처리되지 않고 누적된다. 현재 설계에 외부 감지 수단이 없어 대응이 늦어진다. |
| 영향 | 발송 지연이 쌓이는 동안 인지 불가. SLA 위반 및 사용자 민원 가능성. |
| 대응 전략 | PENDING 건 수가 임계값(예: 500건) 초과 시 알람 설정. Batch 미실행 감지 모니터링 추가. 수동 실행 절차 문서화. |

---

### R-04. 업체 API 장기 장애 후 FAILED 대량 발생

| 항목 | 내용 |
|------|------|
| 리스크 | 업체 API가 장기간 5xx/타임아웃 상태이면 재시도(최대 3회)가 모두 소진되어 FAILED 건이 대량 발생한다. 현재 설계에 FAILED 건 수동 재처리 방안이 없다. |
| 영향 | 발송 유실. 관리자에게 알림 없음. FAILED 건 재발송 방법 미정. |
| 대응 전략 | FAILED 건 수 임계값 초과 시 알람 설정. 운영 시 필요한 경우 outbox `status`를 PENDING으로 수동 전환하는 쿼리를 준비해둠(`UPDATE outbox_messages SET status='PENDING', retry_count=0 WHERE status='FAILED' AND org_id=?`). |

---


## 2. 검증 시나리오

### 시나리오 1. 기존 조직 — 마이그레이션 후 정상 발송

마이그레이션(V4) 실행 후 기존 조직이 클라우드메시지로 계속 정상 발송되는지 확인합니다.

```
사전 조건: 기존 조직이 organization_message_providers에 cloud_message로 삽입되어 있음

검증 절차:
  1. 기존 조직 중 하나로 메시지 발송 요청
  2. outbox_messages에 PENDING 레코드 생성 확인
  3. Batch 실행 후 SENT로 상태 전환 확인
  4. message_logs에 provider_type = 'cloud_message' 기록 확인

기대 결과: 마이그레이션 전과 동일하게 발송 성공
```

---

### 시나리오 2. 업체 변경 — 센드톡으로 전환 후 발송

조직 관리자가 업체를 센드톡으로 변경한 뒤 발송이 정상 동작하는지 확인합니다.

```
사전 조건: 특정 조직이 cloud_message로 설정되어 있음

검증 절차:
  1. PUT /organizations/{orgId}/message-provider 로 sendtalk 변경
  2. 해당 조직으로 메시지 발송 요청
  3. Batch 실행 → SendtalkProvider 경유 확인
  4. message_logs에 provider_type = 'sendtalk' 기록 확인

기대 결과: 센드톡 API를 통해 발송 성공
```

---

### 시나리오 3. 업체 변경 중 진행 중인 발송 처리

업체 변경 시점에 이미 PENDING 상태인 발송 건이 기존 업체로 완료되는지 확인합니다.

```
사전 조건: 조직 A가 cloud_message로 설정, outbox에 PENDING 건이 존재

검증 절차:
  1. PENDING 건이 있는 상태에서 sendtalk으로 업체 변경
  2. Batch 실행
  3. 기존 PENDING 건 → cloud_message로 처리되는지 확인
  4. 이후 새 발송 요청 → sendtalk으로 처리되는지 확인

기대 결과:
  - 변경 전 PENDING 건 → cloud_message 완료
  - 변경 후 신규 건    → sendtalk 처리
```

> 근거: [A-07 가정 사항](../requirement/requirements.md#a-07-업체-변경-시-진행-중인-발송-처리) — 업체 변경은 다음 발송 요청부터 적용

---

### 시나리오 4. 재시도 정책 — 일시 장애 상황

업체 API가 일시적으로 실패할 때 재시도 후 정상 처리되는지 확인합니다.

```
검증 절차:
  1. 업체 API 엔드포인트를 일시적으로 차단 (테스트 환경)
  2. 발송 요청 → Batch 실행 → 1차 실패 확인
  3. 차단 해제 후 재시도 실행
  4. SENT 전환 확인

기대 결과: 최대 3회 재시도 후 성공 처리
```

---

### 시나리오 5. Daily Limit 초과 처리

업체 일일 발송 한도 초과 시 즉시 FAILED 처리되고 클라이언트에 명확히 전달되는지 확인합니다.

```
검증 절차:
  1. 업체 API Mock으로 429 + DAILY_LIMIT_EXCEEDED 응답 재현
  2. outbox 상태 → 즉시 FAILED 전환 확인 (재시도 없음)
  3. API 응답 body에 code: DAILY_LIMIT_EXCEEDED 포함 확인

기대 결과: 재시도 없이 즉시 실패 처리, 클라이언트 구분 가능
```

---

### 시나리오 6. Credential 미설정 조직 처리

업체 설정이 없는 조직이 발송 시도할 때 명확히 실패하는지 확인합니다.

```
검증 절차:
  1. organization_message_providers에 레코드 없는 조직으로 발송 요청
  2. outbox PENDING 생성 확인
  3. Batch 실행 → ProviderNotConfiguredException 발생
  4. outbox 상태 → FAILED 전환 확인

기대 결과: 조용히 유실되지 않고 FAILED로 기록됨
```

---

## 2. 롤백 계획

### 롤백 트리거 조건

| 상황 | 판단 기준 |
|------|-----------|
| 마이그레이션 후 기존 조직 발송 실패 | 시나리오 1 검증 실패 시 |
| 신규 업체 분기 오작동 | 시나리오 2 검증 실패 시 |
| 서비스 전체 발송 중단 | PENDING 건이 계속 누적되고 SENT 전환 없음 |

---

### 롤백 절차

**1단계: 즉시 대응 — 특정 조직 업체 원복**

특정 조직만 문제가 있을 경우, provider_type을 cloud_message로 되돌립니다.

```sql
UPDATE organization_message_providers
SET provider_type = 'cloud_message',
    credentials   = '{기존 암호화된 Credential}',
    updated_at    = NOW()
WHERE org_id = {문제 조직 ID};
```

적용 즉시 다음 Batch 실행부터 기존 업체로 처리됩니다.

---

**2단계: 전체 롤백 — 전체 조직 cloud_message 원복**

서비스 전체에 문제가 있을 경우, 모든 조직을 cloud_message로 일괄 원복합니다.

```sql
UPDATE organization_message_providers
SET provider_type = 'cloud_message',
    updated_at    = NOW();
```

---

**3단계: DB 스키마 롤백 (최후 수단)**

신규 테이블 자체에 문제가 있을 경우, Flyway 롤백 스크립트를 실행합니다.

```
실행 순서 (역순):
  V4 롤백: organization_message_providers 전체 삭제
  V3 롤백: message_logs, cloud_message_logs, sendtalk_logs 삭제
  V2 롤백: outbox_messages 삭제
  V1 롤백: organization_message_providers 삭제
```

> V1~V3은 신규 테이블이므로 롤백 시 기존 서비스 데이터에 영향 없음.
> V4 롤백 전 organization_message_providers 데이터 백업 필수.

---

## 3. 배포 전 체크리스트

| 항목 | 확인 |
|------|------|
| `CREDENTIAL_ENCRYPTION_KEY` 환경변수 설정 완료 | □ |
| V4 마이그레이션 실행 전 organizations 테이블 백업 | □ |
| Batch 실행 주기 및 청크 크기 설정 확인 | □ |
| 기존 전역 API 키 환경변수 제거 준비 완료 | □ |
| 시나리오 1~6 검증 환경 준비 완료 | □ |
