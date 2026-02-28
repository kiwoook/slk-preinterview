[← Part A 홈](../../README.md)

`Path: part-a-server/docs/architecture/credential-encryption.md`

# Credential 암호화 서비스 설계

조직별 API Credential(AccessKey, SecretKey 등)을 DB에 안전하게 저장하고 조회하는 암호화 레이어 설계입니다.

> Credential 저장 방식 결정 배경 — [ADR-002](adr.md#adr-002-credential-및-조직-업체-매핑-저장-방식) 참고

---

## 1. 암호화 알고리즘 선택

### 고민

DB에 저장되는 Credential을 암호화할 알고리즘을 선택한다.

| | AES-256-GCM | AES-256-CBC |
|---|---|---|
| 기밀성 | ✅ | ✅ |
| 무결성 검증 | ✅ (인증 태그 내장) | ❌ (별도 HMAC 필요) |
| 변조 감지 | ✅ 복호화 시 자동 감지 | ❌ 별도 처리 필요 |
| 구현 복잡도 | 낮음 (무결성 내장) | 높음 (MAC 별도 구현) |
| 표준 권고 | NIST 권장 | 구식, 오용 가능성 높음 |

### 선택 → AES-256-GCM

### 근거

- **인증 있는 암호화(AEAD)**: GCM은 암호화와 동시에 인증 태그(Authentication Tag)를 생성한다. DB에 저장된 값이 변조됐을 경우 복호화 시 예외가 발생하여 자동으로 감지된다. CBC는 이 기능이 없어 별도 HMAC을 구현해야 한다.
- **구현 단순성**: 무결성 보장을 위한 추가 로직 없이 하나의 알고리즘으로 처리 가능하다.
- **NIST 권고 표준**: GCM은 현재 표준 기관에서 권장하는 암호화 모드다.

---

## 2. 암호화 흐름

### 저장 흐름 (PUT 요청 → DB)

```
[PUT 요청 — credentials JSON]
         ↓
[CredentialEncryptionService.encrypt()]
  1. 32바이트 랜덤 IV 생성 (매 암호화마다 신규 생성)
  2. 환경변수에서 AES-256 키 로드 (CREDENTIAL_ENCRYPTION_KEY)
  3. AES-256-GCM 암호화 수행
  4. IV + CipherText + AuthTag → Base64 인코딩 후 단일 문자열 결합
         ↓
[organization_message_providers.credentials 컬럼에 저장]
```

**저장 형식**

```
{IV(Base64)}:{CipherText+AuthTag(Base64)}
```

IV를 암호문과 함께 저장하는 이유: 복호화 시 동일한 IV가 필요하기 때문이다. IV는 비밀값이 아니며 예측 불가능성(랜덤성)이 중요하다.

---

### 조회 흐름 (ProviderFactory → Provider)

```
[ProviderFactory — org_id로 organization_message_providers 조회]
         ↓
[CredentialEncryptionService.decrypt()]
  1. 저장된 문자열에서 IV와 CipherText 분리
  2. 환경변수에서 AES-256 키 로드
  3. AES-256-GCM 복호화 (인증 태그 자동 검증 — 변조 시 예외 발생)
  4. 복호화된 JSON 파싱 → Credential 객체 반환
         ↓
[CloudMessageProvider 또는 SendtalkProvider에 Credential 전달]
```

---

## 3. 암호화 키 관리

### 키 로드 방식

```bash
# 환경변수 (256비트 = 32바이트, Base64 인코딩)
CREDENTIAL_ENCRYPTION_KEY=<Base64로 인코딩된 32바이트 랜덤값>
```

- 애플리케이션 시작 시 키 존재 여부 및 길이(256비트)를 검증한다. 키가 없거나 길이가 맞지 않으면 **기동 실패**로 처리한다 (묵시적 진행 금지).
- 키는 메모리에 보관하며, 로그에 절대 출력하지 않는다.

### 키 분실 시 영향

- 암호화 키를 분실하면 DB에 저장된 **전체 Credential 복구가 불가능**하다.
- 대응: 조직 관리자가 UI에서 Credential을 재입력해야 한다.

### 향후 키 관리 전략

| 단계 | 방식 | 적용 시점 |
|------|------|-----------|
| 현재 | 환경변수 직접 주입 | 초기 운영 |
| 중기 | AWS Secrets Manager 또는 Vault에서 런타임 주입 | 멀티 인스턴스 운영 시 |
| 장기 | AWS KMS — 키 자체를 외부 관리, 로테이션 자동화 | 보안 요구 수준 상승 시 |

---

## 4. 로그 마스킹 정책

복호화된 Credential이 로그에 노출되면 암호화 의미가 없어진다.

| 위치 | 정책 |
|------|------|
| API 요청 로그 | `credentials` 필드 전체를 `[REDACTED]`로 대체 |
| Credential 객체 | `toString()` 오버라이드 — 민감 필드 마스킹 |
| 예외 스택 트레이스 | Credential 객체가 메시지에 포함되지 않도록 처리 |
| GET 응답 | key류 필드 마지막 4자리만 노출 (`AKCLOU****`) |

---

## 5. CredentialEncryptionService 책임 범위

```
CredentialEncryptionService
  ├── encrypt(plainJson: String) → encryptedString: String
  └── decrypt(encryptedString: String) → plainJson: String
```

- **단일 책임**: 암호화/복호화만 담당. JSON 파싱, Provider 선택 등 다른 책임은 갖지 않는다.
- **호출 위치**:
  - `encrypt` — 업체 설정 저장 시 (PUT API 처리 레이어)
  - `decrypt` — ProviderFactory가 Provider 생성 시
