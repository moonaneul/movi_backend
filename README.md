# MOVI Backend

MOVI의 **Spring Backend 저장소**입니다.

MOVI는 시각 중심 금융 UI 이용에 어려움이 있는 사용자를 고려해,  
음성으로 계좌 조회와 송금 과정을 보조하는 금융 서비스 프로토타입입니다.

이 Backend의 핵심 책임은 **AI가 해석한 결과를 그대로 금융 실행에 사용하지 않고, 실제 금융 상태와 권한을 다시 검증한 뒤 거래를 실행하는 것**입니다.

> 전체 프로젝트 설명과 개인 기여 범위는 [MOVI Overview](https://github.com/moonaneul/movi-overview)를 기준으로 합니다.  
> 이 저장소는 팀 Backend 전체 코드를 포함하며, 저장소 전체를 문하늘 개인 단독 구현으로 표현하지 않습니다.

---

## 1. Backend Responsibility

MOVI Backend는 다음 역할을 담당합니다.

- 인증 및 사용자 세션
- 연결 계좌 / 수취인 / 거래 상태 관리
- Voice session 생성 및 상태 관리
- Voice AI 호출 및 응답 검증
- 누락 슬롯 저장·병합·재질문 결정
- 거래 내용 확인 상태 관리
- FDS 입력 데이터 구성 및 결과 해석
- 이체 실행 / 차단
- idempotency 기반 중복 실행 방어
- 보호자 알림 이벤트 처리
- Frontend에 일관된 API 응답 제공

핵심 원칙:

> **AI는 해석하고, Backend는 검증하고 실행한다.**

---

## 2. Architecture

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

AI가 반환한 Intent / Entity는 금융 사실의 최종 값으로 사용하지 않고,  
Backend가 실제 계좌·수취인·소유권·한도·세션 상태를 다시 검증합니다.

상세 구조:
- [Frontend–AI–Backend Integration Spec](./docs/integration-spec.md)
- [AI Voice / FDS API Contract](./docs/ai-api-contract.md)

---

## 3. Voice Transfer Flow

```text
음성 입력
→ Backend 인증 / 세션 / 파일 검증
→ Voice AI 분석
→ Intent / Entity 수신
→ Backend 필수값 및 실제 금융 데이터 재검증

   ├─ 정보 누락 → 재질문
   ├─ 검증 실패 → 오류 안내
   └─ 검증 완료 → 거래 내용 확인

→ 사용자 명시적 확인
→ FDS 평가

   ├─ LOW    → 거래 실행
   ├─ MEDIUM → 거래 실행 + 보호자 알림 요청
   ├─ HIGH   → 거래 차단 + 보호자 알림 요청
   └─ 오류   → 거래 중단

→ 결과 저장 및 Frontend 응답
```

---

## 4. Key Backend Decisions

### Backend owns conversation state

음성 대화에서 세션, 슬롯, 확인 상태는 Backend가 단일 소유합니다.

AI는 현재 발화를 해석하지만 이전 슬롯을 임의로 합치지 않습니다.  
Frontend 역시 실제 금융 실행 상태를 결정하지 않습니다.

### Re-validate AI output

AI confidence가 높더라도 다음은 Backend가 다시 확인합니다.

- 실제 사용자의 계좌인지
- 수취인이 유효한지
- 금액이 유효한지
- 잔액 / 한도를 충족하는지
- 세션이 유효한지
- 사용자가 확인한 값과 실행할 값이 같은지

### Explicit confirmation

사용자가 확인한 수취인·금액·출금 계좌와 실제 실행 값이 일치하도록 합니다.

핵심 거래 값이 바뀌면 이전 확인 상태를 그대로 재사용하지 않습니다.

### Idempotency

네트워크 timeout이나 재시도로 같은 송금 요청이 반복될 수 있기 때문에  
idempotency key를 사용해 중복 거래를 방어합니다.

### Fail-closed

FDS timeout, 오류, 잘못된 payload, 검증 실패를 정상 거래로 간주하지 않습니다.

확인할 수 없으면 거래를 진행하지 않는 방향으로 처리합니다.

---

## 5. Tech Stack

- **Java 21**
- **Spring Boot 4.1**
- Spring Web MVC
- Spring Security
- Spring Data JPA
- MySQL
- JWT
- Bean Validation
- Actuator
- Prometheus registry
- JUnit / Spring Security Test
- H2 for tests

정확한 의존성은 [build.gradle](./build.gradle)을 참고하세요.

---

## 6. Main Documentation

| Document | Purpose |
|---|---|
| [integration-spec.md](./docs/integration-spec.md) | Frontend / Backend / AI 책임 및 통합 정책 |
| [ai-api-contract.md](./docs/ai-api-contract.md) | Voice AI / FDS 내부 API 계약 |
| [api-response.md](./docs/api-response.md) | 공통 API 응답 형식 |
| [error-codes.md](./docs/error-codes.md) | 오류 코드와 사용자 안내 기준 |
| [domain-guide.md](./docs/domain-guide.md) | Backend 도메인 구조 |
| [ERD.md](./docs/ERD.md) | 데이터 모델 |
| [execution-plan.md](./docs/execution-plan.md) | 구현 당시 실행 계획 |

문서와 현재 코드가 충돌할 경우 최신 구현과 실제 테스트 결과를 우선 확인합니다.

---

## 7. Local Validation

```bash
./gradlew test
./gradlew build
```

테스트 통과만으로 실제 금융기관 연동이나 운영 완료를 의미하지 않습니다.

---

## 8. Mock / Real Boundary

이 저장소에는 실제 연동을 고려한 adapter 구조가 포함되어 있지만,  
포트폴리오에서 검증된 MOVI의 금융 흐름은 **Mock 기반 환경을 중심으로 설명합니다.**

현재 다음을 상용 운영 경험으로 주장하지 않습니다.

- 실제 금융기관 OpenBanking 운영
- 실제 은행 계좌 production 송금
- 실제 SMS production 운영
- 대규모 트래픽 / 고가용성 운영

---

## 9. Portfolio Context

문하늘의 검증된 개인 기여는 다음 문서에서 별도로 확인할 수 있습니다.

- [MOVI Overview](https://github.com/moonaneul/movi-overview)
- [My Contribution](https://github.com/moonaneul/movi-overview/blob/master/docs/contribution.md)
- [System Architecture](https://github.com/moonaneul/movi-overview/blob/master/docs/architecture.md)
- [Validation](https://github.com/moonaneul/movi-overview/blob/master/docs/validation.md)
- [Limitations & Evidence Boundaries](https://github.com/moonaneul/movi-overview/blob/master/docs/limitations.md)

대표 GitHub 기록:

- [Backend PR #3 — Front / AI / Backend 통합 명세 수립](https://github.com/movi-ai-challenge/movi_backend/pull/3)
- [Backend PR #35 — 음성 이체 명령 및 실행 흐름 구현](https://github.com/movi-ai-challenge/movi_backend/pull/35)

> 구현 과정에는 Claude / Codex 등 AI 코딩 도구가 활용되었습니다.  
> 개인 기여는 코드 라인 수가 아니라 문제 정의, 책임 경계, 계약 설계, 구현 검증 범위를 기준으로 설명합니다.
