[← Part A 홈](../../README.md)

`Path: part-a-server/docs/milestone/work-plan.md`

# 작업 단계 계획

## 전체 흐름

```
Phase 1 → Phase 2 → Phase 3 → Phase 4 → Phase 5 → Phase 6 → Phase 7
요구사항    DB 스키마   Credential  Provider    Outbox     API       통합
분석·확정   구성        암호화      추상화      + Batch    구현      검증
           마이그레이션  서비스      레이어      구현
```

---

## Phase 1. 요구사항 분석 및 설계 확정 ✅

**목표:** 현재 시스템을 파악하고 변경 범위 및 기술 방향을 확정합니다.

**작업 항목:**
- 현재 MessageService 코드 및 클라우드메시지 연동 방식 분석
- 클라우드메시지 / 센드톡 API 문서 비교 및 차이점 도출
- 가정 사항 정의 (Integration 모델 확정, 조직별 Credential 등)
- 기술 결정 사항 문서화 (ADR 작성)

**산출물:**
- [`requirements.md`](../requirement/requirements.md) — 가정 사항 및 변경 요구사항
- [`adr.md`](../architecture/adr.md) — 기술적 의사결정 (ADR-001 ~ ADR-004)
- [`system-design.md`](../architecture/system-design.md) — 전체 아키텍처 및 DB 스키마
- [`api-spec.md`](../api/api-spec.md) — API 명세

---

## Phase 2. DB 스키마 구성 및 마이그레이션 ✅

**목표:** 변경된 설계에 맞는 DB 구조를 생성하고 기존 데이터를 안전하게 이관합니다.

**실행 순서 (Flyway 버전 순)**

| 순서 | 파일명 (Flyway 버전 규칙) | 내용 |
|------|--------------------------|------|
| 1 | `V1__create_organization_message_providers.sql` | 조직별 업체 설정 테이블 생성 |
| 2 | `V2__create_outbox_messages.sql` | 발송 신뢰성 보장 Outbox 테이블 생성 |
| 3 | `V3__create_message_logs.sql` | 공통 발송 내역 + 업체별 상세 테이블 생성 |
| 4 | `V4__migrate_existing_organizations.sql` | 기존 조직 → cloud_message 기본값 삽입 |

**V4 마이그레이션 전략 (무중단)**

```
1. 배포 전: CREDENTIAL_ENCRYPTION_KEY 환경변수 설정 확인
2. 배포 시: Flyway V4 자동 실행
   → organizations 테이블의 모든 org_id를 읽어
   → organization_message_providers에 (org_id, 'cloud_message', 암호화된 기존 Credential) 삽입
3. 배포 후: 기존 전역 환경변수(API 키류) 제거
```

> V4 실행 전 기존 전역 환경변수가 반드시 존재해야 한다. 없으면 마이그레이션 실패.

**산출물:**
- Flyway 마이그레이션 SQL 파일 4개 (스키마 설계는 [system-design.md](../architecture/system-design.md#4-db-스키마-변경) 참고)

**주의:**
- V1~V3은 신규 테이블 생성이므로 기존 서비스에 영향 없음
- V4는 기존 조직 데이터를 읽는 DML — 실행 전 조직 수 확인 및 백업 권장
- 마이그레이션 완료 후 기존 조직의 발송이 정상 동작하는지 즉시 검증

---

## Phase 3. Credential 암호화 서비스 구현 ✅

**목표:** 조직별 Credential을 안전하게 저장하고 조회할 수 있는 암호화 레이어를 구현합니다.

**작업 항목:**
- AES-256 기반 `CredentialEncryptionService` 구현 (암호화 / 복호화)
- 암호화 키를 환경변수(`CREDENTIAL_ENCRYPTION_KEY`)에서 주입
- 저장 시 암호화, 조회 시 복호화 흐름 구현
- 단위 테스트 (암복호화 정합성 검증)

**산출물:**
- [`credential-encryption.md`](../architecture/credential-encryption.md) — 암호화 알고리즘 선택, 흐름 설계, 키 관리 전략, 로그 마스킹 정책

**주의:**
- 로그에 복호화된 Credential이 절대 출력되지 않도록 처리
- 암호화 키 분실 시 전체 Credential 복구 불가 → 키 관리 전략 문서화

---

## Phase 4. Provider 추상화 레이어 구현 ✅

**목표:** ProviderFactory가 조직의 업체 설정을 읽어 맞는 Provider를 선택하고, 각 Provider가 독립적으로 업체 API를 호출하는 구조를 구현합니다.

**설계 원칙:** 업체 선택만 추상화. 완전 통합 인터페이스는 성급한 추상화로 판단하여 적용하지 않음. ([ADR-001 참고](../architecture/adr.md#adr-001-provider-추상화-수준))

**작업 항목:**
- `ProviderFactory` 구현
  - `organization_message_providers`에서 provider_type + credentials 조회
  - Credential 복호화 후 맞는 Provider 인스턴스 반환
- `CloudMessageProvider` 구현
  - HMAC-SHA256 서명 생성 로직
  - 클라우드메시지 API 호출 (RestClient)
  - 전화번호 E.164 변환 처리
- `SendtalkProvider` 구현
  - Bearer 토큰 헤더 설정
  - 센드톡 API 호출 (RestClient, 유형별 엔드포인트)
- 단위 테스트 (각 Provider 독립 테스트)

**산출물:**
- [`provider-design.md`](../architecture/provider-design.md) — ProviderFactory 흐름, CloudMessageProvider(HMAC-SHA256), SendtalkProvider(Bearer 토큰), 업체별 차이점

---

## Phase 5. Outbox Pattern + Spring Batch 구현 ✅

**목표:** 발송 신뢰성을 보장하는 Outbox Pattern과 청크 기반 Batch 처리를 구현합니다.

**작업 항목:**
- `MessageService` 수정: 업체 API 직접 호출 → `outbox_messages` 저장으로 변경 (`@Transactional`)
- Spring Batch `OutboxProcessingJob` 구현
  - `ItemReader`: PENDING 상태 outbox 청크 단위 조회
  - `ItemProcessor`: ProviderFactory → Provider API 호출
  - `ItemWriter`: outbox 상태 업데이트 (SENT / FAILED)
- 지수 백오프 재시도 정책 적용 (최대 3회, Jitter 포함)
- 비동기 로그 저장 구현 (`@Async` + Spring Event)
  - `message_logs` 저장
  - `cloud_message_logs` 또는 `sendtalk_logs` 저장

**산출물:**
- [`outbox-batch-design.md`](../architecture/outbox-batch-design.md) — Outbox Pattern 개념, 청크 처리, 재시도 정책, 장애 시나리오별 동작

**주의:**
- 429 응답은 지수 백오프 재시도 대상
- Daily Limit 초과(429 + `DAILY_LIMIT_EXCEEDED`)는 재시도 불필요 → 즉시 FAILED 처리

---

## Phase 6. API 구현 ✅

**목표:** 조직 관리자가 업체 설정을 조회·변경할 수 있는 API를 제공합니다.

**작업 항목:**
- `GET /organizations/{orgId}/message-provider` — 현재 설정 조회 (Credential 마스킹)
- `PUT /organizations/{orgId}/message-provider` — 업체 및 Credential 변경
- `POST /organizations/{orgId}/message-provider/test` — 연결 테스트 (**우선순위 낮음**)
- JWT 권한 검증 (role = ADMIN, 본인 조직만 접근)
- Credential 필드 유효성 검사 (업체별 필수 필드 검증)

**범위 명확화:** 메시지 발송 API는 기존 시스템에 존재. 이번 과제 범위는 조직별 업체 설정 관리 API에 한정.

**산출물:**
- [`api-spec.md`](../api/api-spec.md) — 업체 설정 조회/변경/테스트 3개 엔드포인트

---

## Phase 7. 통합 검증 및 롤백 계획 ✅

**목표:** 전체 흐름이 정상 동작하는지 검증하고 롤백 시나리오를 준비합니다.

**작업 항목:**
- 조직별 업체 분기 통합 테스트 (cloud_message / sendtalk 각각)
- 기존 클라우드메시지 발송 회귀 테스트 (마이그레이션 이후 정상 동작 확인)
- 업체 변경 중 진행 중인 발송 처리 확인 (기존 업체로 완료되는지)
- 429 / Daily Limit 시나리오 테스트
- 롤백 계획: `organization_message_providers`의 provider_type을 cloud_message로 되돌리면 즉시 복구

**산출물:**
- [`verification-rollback.md`](../architecture/verification-rollback.md) — 검증 시나리오 6개, 롤백 3단계 절차, 배포 전 체크리스트
