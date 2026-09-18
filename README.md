# MOVI Backend

MOVI의 Spring Backend 저장소입니다.

Backend는 인증, 계좌/수취인 검증, 음성 세션, 거래 확인, FDS 결과 처리, 이체 실행, idempotency 등을 담당합니다.

전체 프로젝트 설명:
- [MOVI Overview](https://github.com/moonaneul/movi-overview)

---

## Backend Responsibility

- 인증 및 사용자 세션
- 연결 계좌 / 수취인 / 거래 상태 관리
- Voice session 상태 관리
- Voice AI 호출 및 응답 검증
- 누락 슬롯 저장 / 병합 / 재질문
- 거래 내용 확인 상태 관리
- FDS 입력 구성 및 결과 처리
- 이체 실행 / 차단
- idempotency 기반 중복 실행 방어
- 보호자 알림 이벤트 처리
- Frontend API 응답 제공

핵심 원칙:

> **AI는 해석하고, Backend는 검증하고 실행한다.**

---

## Architecture

```text
[ Frontend ]
      │
      ▼
[ Spring Backend ]
      │
      ├────► Voice AI
      │       STT / Intent / Entity
      │
      ├────► FDS AI
      │       Risk Assessment
      │
      └────► Financial Adapter
              Mock 중심 거래 흐름
```

Frontend는 AI/FDS를 직접 호출하지 않습니다.

AI가 반환한 Intent / Entity는 Backend에서 실제 계좌·수취인·소유권·한도·세션 상태와 다시 대조합니다.

---

## Voice Transfer Flow

```text
음성 입력
→ 인증 / 세션 / 파일 검증
→ Voice AI 분석
→ Intent / Entity 수신
→ Backend 재검증

   ├─ 정보 누락 → 재질문
   ├─ 검증 실패 → 오류 안내
   └─ 검증 완료 → 거래 내용 확인

→ 사용자 명시적 확인
→ FDS 평가

   ├─ LOW    → 거래 실행
   ├─ MEDIUM → 거래 실행 + 보호자 알림 요청
   ├─ HIGH   → 거래 차단 + 보호자 알림 요청
   └─ 오류   → 거래 중단

→ 결과 저장 및 응답
```

---

## Key Decisions

### Conversation state
세션, 슬롯, 확인 상태는 Backend가 관리합니다.

### Re-validation
AI confidence와 별개로 실제 금융 데이터를 다시 확인합니다.

### Explicit confirmation
사용자가 확인한 거래 값과 실제 실행 값이 일치해야 합니다.

### Idempotency
같은 요청이 반복되더라도 중복 거래가 실행되지 않도록 방어합니다.

### Fail-closed
FDS 오류나 검증 실패 시 거래를 계속 진행하지 않습니다.

---

## Tech Stack

- Java 21
- Spring Boot
- Spring Security
- Spring Data JPA
- MySQL
- JWT
- Bean Validation
- Actuator
- Prometheus registry
- JUnit
- H2 for tests

정확한 의존성은 [build.gradle](./build.gradle)을 참고하세요.

---

## Documentation

- [integration-spec.md](./docs/integration-spec.md)
- [ai-api-contract.md](./docs/ai-api-contract.md)
- [api-response.md](./docs/api-response.md)
- [error-codes.md](./docs/error-codes.md)
- [domain-guide.md](./docs/domain-guide.md)
- [ERD.md](./docs/ERD.md)

---

## Local Validation

```bash
./gradlew test
./gradlew build
```

---

## Integration Boundary

금융 흐름은 Mock 기반 환경을 중심으로 검증했습니다.

실제 금융기관 production 송금이나 상용 운영 환경을 전제로 한 저장소는 아닙니다.
