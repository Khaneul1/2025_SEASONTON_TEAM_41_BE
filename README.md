# 📌 핀로그 — 청년을 위한 빚 공유 플랫폼

> 카카오 × 구름 구름톤 유니브 시즌톤 참가작  

<br>

## 🔍 왜 만들었나요?

"나만 이렇게 힘든 건가?"

청년 세대는 학자금 대출, 전월세 보증금, 생활비 부채 등  
경제적 어려움을 혼자 감당하는 경우가 많습니다.  
하지만 주변의 실제 재산·부채 현황을 알 수 없어 자신과 비교하기 어렵고,  
부채에 대한 현실적인 조언을 구할 공간도 마땅치 않습니다.

**핀로그**는 사용자가 자신의 부채와 재산을 자유롭게 공개하고,  
나이대별 현황을 시각화해 비교할 수 있는 빚 공유 플랫폼입니다.  
게시글을 통해 공감과 조언을 나누며, 함께 청산을 향해 나아갑니다.

<br>

## 🙋 내가 맡은 역할 — BE 팀장

**홈 대시보드 (데이터 제공)**  
- 총 부채 금액: `GET /api/debts/total?email=` 등으로 원금 합계 조회  
- 부채 목록·잔여 상환 기간: `GET /api/debts?email=` — 부채별 `remainingPeriod` 문자열(예: `N년 M개월 D일`)  
- 월 상환 목표·실제 상환 누적·진행률 등은 **가계부·부채 API 조합**으로 프론트 대시보드에서 구성

**가계부 (`domain/ledger`)**  
- 일자·금액·카테고리·설명 입력: `POST /api/ledger` — 요청 본문에 `email`, `date`, `amount`, `category`, `description`  
- 특정 유저 **전체** 내역: `GET /api/ledger/{email}`  
- **월별** 수입·지출 합계: `GET /api/ledger/{email}/monthly/income-expense?year=&month=` — 금액 **양수=수입, 음수=지출**으로 집계  
- 수정·삭제: `PUT /api/ledger/{ledgerId}`, `DELETE /api/ledger/{ledgerId}`  

**빚 청산 시뮬레이터 (`domain/debtsSimulation`)**  
- 부채 등록: `POST /api/debts/add?email=` — 원금·연금리·기간(개월)·상환 방식 등 저장  
- 목록·총 부채: `GET /api/debts`, `GET /api/debts/total`  
- 잔여 기간은 `startDate + periodMonths`와 오늘 날짜를 `Period.between`으로 계산해 문자열로 반환

<br>

## 🧩 코드 기준 구현 메모 (레포 최신 main)

| 영역 | 패키지 / 클래스 | 설명 |
|------|-----------------|------|
| 가계부 | `LedgerController`, `LedgerService`, `Ledger`, `LedgerRepository` | JPA로 `User`와 `@ManyToOne` 연관, `LedgerRepository.findByUserAndDateBetween`로 월 범위 조회 |
| 부채 시뮬 | `DebtController`, `DebtService`, `Debt` | `User` 단위 부채 CRUD·합계·잔여 기간 문자열 |
| 공통 | `GlobalExceptionHandler`, `ErrorCode`, JWT·`SecurityConfig` 등 | 예외·인증 처리 |

가계부 금액 집계 예시 (서비스 핵심 로직):

```java
// 월별: 양수 합 = 수입, 음수 합 = 지출
int totalIncome = ledgerList.stream()
    .filter(ledger -> ledger.getAmount() > 0)
    .mapToInt(Ledger::getAmount).sum();
int totalExpense = ledgerList.stream()
    .filter(ledger -> ledger.getAmount() < 0)
    .mapToInt(Ledger::getAmount).sum();
```

## 🔧 가계부·부채 쪽에서 겪었던 일 (커밋 + 코드 기준 트러블슈팅)

### 1) userId → email로 식별자 통일
- 프론트·회원 API가 이메일 기준으로 맞춰지면서, 가계부 경로·요청 DTO·`UserRepository` 조회가 `findByUserId` / `userId`에서 `findByEmail` / `email`로 **여러 번에 나눠 수정**되었습니다.  
  - 관련 커밋(예): `userId -> email`, `ledger userId -> email 수정`, `ledger Service userId -> email`
- **교훈**: 팀 전체에서 “유저 키(식별자)”를 하나로 정한 뒤 도메인별로 나눠 개발하면 이런 리팩터링을 크게 줄일 수 있습니다.

### 2) 월별 API 설계의 변화
- `LedgerController`에 `/{userId}/monthly` 형태의 월별 상세 조회 메서드가 **주석 처리**되어 있고, 실제로는 `.../monthly/income-expense`로 **월 합계(수입·지출)** 중심으로 제공됩니다.
- “일별 상세”까지 백엔드에서 나누려다, 일정·우선순위에 맞춰 **합계 API + 전체 목록 조회 조합**으로 정리된 흔적이 남아 있습니다.

### 3) 부채 시뮬레이션: 머지 → 리버트 → 다시 반영
- 커밋 히스토리에서 `feat/04debtsSimulation` 머지 후 `Revert "Feat/04debts simulation"`이 있었고, 이후 리버트 머지 PR로 복구된 기록이 있습니다.
- 초기에는 `빚 청산 시뮬레이션 (컨트롤러 아직 구현 안 됨)` 수준으로 올라온 뒤, `컨트롤러 추가` 등으로 단계적으로 완성되었습니다.
- **교훈**: 통합 브랜치에서 한 번에 큰 기능을 넣을 때, 되돌리기(리버트) 비용이 크다는 것을 체감했습니다.

### 4) 잔여 기간 표시 방식 변경
- 커밋 메시지: `총 개월 수 -> 연도 및 개월 수로 변경`  
- 현재 코드는 `Period.between`으로 **년·월·일 문자열**(`%d년 %d개월 %d일`)로 표현합니다.

### 5) 기타
- 엔티티 `@Table(name = "leger")`는 `ledger` 오타로 보이며, DB 스키마와 맞춰 두었을 가능성이 있습니다(마이그레이션 시 주의).
- `Debt` 엔티티에는 `annualRate`가 있으나, 응답 DTO는 상환 진행률·월 상환액 자동 계산까지는 노출하지 않습니다. 기획 대비 “이자 계산기” 커밋 이후에도 **API 응답은 단순화된 상태**로 남아 있을 수 있습니다.

<br>

## 🛠 기술 스택 (레포 기준)

| 기술 | 용도 |
|---|---|
| **Java 17** | 언어 |
| **Spring Boot 3.x** (`build.gradle` 기준) | 웹·설정 |
| **Spring Data JPA** | `Ledger`, `Debt`, `User` 등 영속화 |
| **Spring Security + JWT** | 인증 (`JwtUtil`, `SecurityConfig` 등) |
| **MySQL / H2** | 런타임·로컬·테스트 DB |
| **Lombok** | DTO·엔티티 보일러플레이트 |
| **Swagger** (`SwaggerConfig`) | API 문서 |

> 백엔드 팀장으로서 서버 구현을 담당했습니다.  
> 세부 의존성은 레포의 `build.gradle`을 참고해 주세요.

<br>

## 🧠 아쉬운 점과 반성

### 1️⃣ 배포 일정을 고려하지 않은 수정 반복
실시간으로 변경되는 기획과 디자인에 맞춰 백엔드 서버 배포 시간을  
고려하지 않고 계속해서 수정 작업을 이어갔습니다.

BE 팀장으로서 할 수 있는 일과 없는 일을 명확히 구분하고,  
일정 협의 없이 진행되는 수정 요청에 대해 먼저 입장을 전달했어야 했습니다.  
기술적 한계를 팀과 공유하는 것도 팀장의 역할이라는 것을 배웠습니다.

### 2️⃣ 기획·디자인팀과의 소통 부재
기획·디자인팀과 백엔드 간의 소통이 원활하지 않아,  
삭제된 기능을 모르는 채로 개발을 계속 진행했고  
결국 완성된 코드를 전부 걷어내야 하는 일이 발생했습니다.

더 주도적으로 대화를 이끌고, 기능 변경에 대한 충분한 협의를  
먼저 제안했어야 했다는 아쉬움이 남습니다.  
개발 착수 전 기능 확정이 얼마나 중요한지 이 경험으로 직접 체감했습니다.
