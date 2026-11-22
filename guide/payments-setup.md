# 결제 연동 설정 가이드 (PortOne · 토스페이먼츠)

> 이 문서는 결제 키를 받은 뒤 채워 넣는 **빈칸 템플릿**이에요. 실제 값은 여기에 적지 마세요.
> 모든 값은 콘솔 [설정] 화면에 붙여넣어 저장해요. 서버 코드·앱·웹 파일·로그에는 넣지 않아요.

## 0. 키 종류와 공개 범위

| 항목 | 예시 형식 | 어디에 넣나 | 공개 여부 |
|---|---|---|---|
| PortOne 상점 아이디 | `store-…` | 콘솔 [설정] › 결제 › 상점 아이디 | 브라우저 SDK에 전달 가능 |
| PortOne V2 API Secret | (PortOne 발급) | 콘솔 › API Secret | **서버 전용** |
| PortOne 웹훅 시크릿 | `whsec_…` | 콘솔 › 웹훅 시크릿 | **서버 전용** |
| PortOne 채널 키 | `channel-key-…` | 콘솔 › 채널 키 (결제수단별) | 브라우저 SDK에 전달 가능 (현금영수증 채널 키는 서버 전용) |
| 토스페이먼츠 클라이언트 키 (결제창 MID) | `test_ck_…` / `live_ck_…` | 콘솔 › 결제 (토스페이먼츠) › 클라이언트 키 | 브라우저 SDK에 전달 가능 |
| 토스페이먼츠 시크릿 키 (결제창 MID) | `test_sk_…` / `live_sk_…` | 콘솔 › 결제 (토스페이먼츠) › 시크릿 키 | **서버 전용** |
| 토스페이먼츠 자동결제 키 쌍 (빌링 MID, 선택) | `test_ck_…` + `test_sk_…` | 콘솔 › 결제 (토스페이먼츠) › 자동결제 클라이언트 키 / 시크릿 키 | 클라이언트 키만 브라우저, 시크릿 키는 **서버 전용** |
| 토스페이먼츠 보안 키 (선택) | 64자리 16진수 | 콘솔 › 결제 (토스페이먼츠) › 보안 키 | **서버 전용** (웹훅 서명) |
| 토스 지급대행 시크릿 키 | `test_sk_…` / `live_sk_…` | 콘솔 › 지급대행 › 시크릿 키 | **서버 전용** |
| 토스 보안 키 (64자리 16진수) | (토스 개발자센터) | 콘솔 › 지급대행 › 보안 키 | **서버 전용** (JWE 암호화 + 웹훅 서명) |

규칙: `test_` 키와 `live_` 키는 섞지 않아요. 키를 바꾸면 [연결 확인]를 다시 해야 해요.
토스페이먼츠 결제 카드에는 **API 개별 연동 키**(`ck`/`sk`)를 넣어요. 위젯 연동 키(`gck`/`gsk`)는 받지 않아요.

## 1. 채워 넣을 값 (받으면 여기에 표시)

| 항목 | 상태 | 값 (마스킹 후 기록) |
|---|---|---|
| PortOne 상점 아이디 | 대기 | `<STORE_ID>` |
| PortOne 테스트 API Secret | 대기 | `<PORTONE_TEST_API_SECRET>` (서버 전용) |
| PortOne 채널 키 (결제수단마다 1개: card, card_billing, kakaopay, naverpay, tosspay, transfer, virtual_account, identity, cash_receipt 중 쓰는 것) | 대기 | `<CHANNEL_KEYS>` (cash_receipt 는 서버 전용) |
| PortOne 웹훅 시크릿 | 대기 | `<PORTONE_WEBHOOK_SECRET>` (서버 전용) |
| 토스페이먼츠 결제창 MID 테스트 키 쌍 | 대기 | `<TOSSPAYMENTS_TEST_CK>` / `<TOSSPAYMENTS_TEST_SK>` (sk 는 서버 전용) |
| 토스페이먼츠 자동결제 MID 테스트 키 쌍 (선택) | 대기 | `<TOSSPAYMENTS_BILLING_TEST_CK>` / `<TOSSPAYMENTS_BILLING_TEST_SK>` (sk 는 서버 전용) |
| 토스 테스트 시크릿 키 | 대기 | `<TOSS_TEST_SECRET_KEY>` (서버 전용) |
| 토스 보안 키 | 대기 | `<TOSS_SECURITY_KEY>` (서버 전용) |
| 테스트 계정 정보 | 대기 | `<TEST_ACCOUNT_NOTE>` |
| 공개 주소 (PUBLIC_BASE_URL) | 대기 | `<PUBLIC_BASE_URL>` |

## 2. 콘솔 설정 순서 (붙여넣고 잠그기)

콘솔 **설정 › 결제 연결** 에서 카드별로 값을 붙여넣어요. 칸 오른쪽의 눈 아이콘으로 입력 중인 값을 가리거나 보여줄 수 있어요.

PortOne 과 토스페이먼츠는 둘 다 선택이에요. 하나만 또는 둘 다 설정할 수 있고, 고객은 설정된 결제사의 결제수단을 모두 봐요. 둘 다 없으면 결제를 받을 수 없어요.

1. **PortOne** 카드 (선택. 본인인증(PASS)에는 필요해요): 상점 아이디, V2 API Secret, 웹훅 시크릿, 그리고 결제수단별 채널 키 칸을 채우고 **저장**.
   - 채널 키 칸: 일반결제(카드), 정기결제(빌링키), 카카오페이, 네이버페이, 토스페이, 가상계좌, 본인인증, 현금영수증, 계좌이체(transfer). 콘솔에는 페이코·삼성페이·애플페이·휴대폰 칸도 있어요. 쓰지 않는 칸은 비워 두세요. 채널 키가 없는 결제수단은 토스페이먼츠가 켜져 있지 않으면 고객 앱에 보이지 않아요 (개발 모드 제외).
   - 각 칸 아래 한 줄 설명이 값을 어디서 복사하는지 알려줘요. 형식이 맞지 않으면 저장 전에 칸 아래에 문구가 떠요.
   - 카드를 처음 저장하는 것은 소유자 인증 코드가 필요해요. 결제 금액이 가는 곳을 정하기 때문이에요.
   - [API 키 확인]은 API Secret만 확인해요. 상점 아이디나 채널 키가 틀린 것은 첫 결제 시도 때 드러날 수 있어요.
   - 기존 설치 호환: 카드 채널 키만 넣었던 설치는 정기결제가 그 `card` 키를 그대로 써요. 정기결제 칸을 채우면 그 칸이 우선해요. 현금영수증은 `cash_receipt` 칸이 없으면 결제수단(계좌이체·가상계좌)의 채널 키를 써요.
   - PortOne 콘솔(V2 웹훅)에 카드 옆 복사 버튼의 **웹훅 주소** `<PUBLIC_BASE_URL>/v1/webhooks/portone` 을 등록해요.
2. **결제 (토스페이먼츠)** 카드 (선택. 혼자 써도 되고 PortOne 과 함께 써도 돼요): 자세한 순서는 [6절](#6-토스페이먼츠로-결제받기-단독-또는-portone과-함께)을 보세요.
   - **사용** 스위치를 건드리지 않고 처음 저장하면 라이브 키는 켜진 채, 테스트 키는 꺼진 채 저장돼요. 이미 결제를 받고 있는 서버라면 처음 저장하기 전에 **사용**을 꺼 두세요. 키를 넣고 **저장** (처음 저장은 소유자 인증 코드) → [API 키 확인] → 토스 개발자센터에 웹훅 등록 → 결제 수단 확인 → 잠금 해제 → **사용** 켜고 저장.
   - 화면의 "결제 연결" 제목 아래 **결제를 받는 곳**에 설정한 결제사가 모두 보여요 (PortOne 먼저, 테스트 키인 결제사 옆에 "테스트 키"). 결제를 받는 곳이 없으면 (토스페이먼츠를 저장했지만 사용이 꺼져 있을 때도) "지금 결제를 받는 곳이 없어요"가 떠요.
3. **지급대행 (토스)** 카드: 시크릿 키(`test_sk_` 또는 `live_sk_`)와 보안 키(64자리)를 붙여넣고 **저장** → [연결 확인] → 성공 후 지급 방식을 `toss` 로 바꿔요.
   - 카드의 **테스트** / **실거래** 표시는 시크릿 키 종류에서 와요. 테스트 통과가 실서비스 키를 열어주지 않아요.
   - 저장하면 카드가 잠겨요. [연결 확인]은 사장님(소유자) 계정이 잠긴 카드에서 바로 할 수 있어요 (잠금 해제 필요 없음).
   - 토스 개발자센터 웹훅 주소: `<PUBLIC_BASE_URL>/v1/webhooks/toss` (`payout.changed`, `seller.changed`). 요청에는 `tosspayments-webhook-signature` 헤더가 있어야 하고, 서명이 틀리면 401 이에요.
4. **잠금**: 저장하면 카드가 **잠김** 상태가 돼요. 비밀 값(API Secret, 시크릿 키, 웹훅 시크릿, 보안 키, 채널 키)은 `••••` 로만 보이고 일부도 보이지 않아요. 상점 아이디와 지급 방식처럼 식별용 값만 그대로 보여요.
   - 값을 바꾸려면 **잠금 해제**를 눌러요. 소유자 계정만 가능하고, 소유자 인증 앱의 6자리 코드가 필요해요. 새 값을 붙여넣고 **저장**하면 다시 잠겨요.
   - 잠금 해제는 5분 뒤 자동으로 잠겨요. 카드에 남은 시간이 "N분 후 잠겨요"로 보여요. 바로 잠그려면 **잠그기**를 눌러요.
   - 잠금, 잠금 해제, 저장, 실패한 해제 시도는 감사 기록(감사 로그)에 남아요. 기록에는 값이 들어가지 않아요.
5. 저장 뒤 [설정] 화면의 상태가 "연결됨"인지 확인해요.

## 3. 웹훅 점검

| 공급자 | 주소 | 검증 | 실패 시 |
|---|---|---|---|
| PortOne | `/v1/webhooks/portone` | Standard Webhooks 서명 (`webhook-signature`) | 401, PortOne이 재전송 |
| 토스페이먼츠 (결제) | `/v1/webhooks/tosspayments` | 보안 키가 있고 `tosspayments-webhook-signature` 헤더가 오면 HMAC-SHA256 확인. 헤더가 없으면 알림으로만 받고 다시 조회 | 서명이 틀리면 401. 5xx 면 토스가 최대 7번 재전송 |
| 토스 (지급대행) | `/v1/webhooks/toss` | `tosspayments-webhook-signature` (보안 키 기반 HMAC-SHA256) | 401 |

웹훅은 결제 상태를 바로 바꾸지 않아요. 알림을 받으면 서버가 결제 기관에 다시 조회해서 확인해요.

## 4. 테스트 계정과 테스트 결제

- PortOne 테스트 상점 채널 키로 카드 결제 1회, 부분 환불 1회를 해 보세요. 부분 환불 뒤 `refunded_amount` 가 바뀌지 않는지 확인해요 (환불 금액은 예약 환불 경로에서만 기록돼요).
- 자동결제(빌링키)는 테스트 상점에서 한 번 발급·결제·삭제까지 확인한 뒤 실서비스에 열어요.
- 현금영수증은 설정의 현금영수증 스위치가 꺼진 상태가 기본이에요. 테스트 상점에서 한 번 발급 응답 형식을 확인한 뒤 켜요.
- 토스페이먼츠는 [6절의 테스트 순서](#테스트-키로-먼저-확인할-것)를 따라요.
- 테스트 계정 정보는 `<TEST_ACCOUNT_NOTE>` 칸에만 메모하고, 이 문서나 저장소에는 올리지 않아요.

## 5. 아직 확인하지 않은 항목

- PortOne 결제 거절 코드: 명세의 `type` 이름만 알려져 있어요. 실제 카드 거절 코드는 테스트 상점에서 한 번 확인해야 해요.
- 현금영수증 발급 응답 (`cashReceipt.issueNumber`): 테스트 상점 1회 발급으로 확인해요.
- 토스 웹훅 서명 계산: 명세대로 구현했어요. 테스트 웹훅 1건으로 서명이 맞는지 확인해요.
- 토스 지급 생성 응답의 래퍼 형식(`entityBody`): 두 형식 모두 읽도록 했어요. 실제 응답을 한 번 확인해요.
- 토스페이먼츠 결제: 실제 테스트 키로만 확인할 수 있는 항목은 [6절](#테스트-키로-먼저-확인할-것) 끝에 있어요.

## 6. 토스페이먼츠로 결제받기 (단독 또는 PortOne과 함께)

PortOne 과 토스페이먼츠는 둘 다 선택이에요. 토스페이먼츠만 써도 되고, PortOne 과 함께 써도 돼요. 고객은 설정된 결제사의 결제수단을 모두 봐요. 둘 다 있으면 결제수단마다 한 곳으로 가요: PortOne 에 그 수단의 채널 키가 있으면 PortOne, 없으면 토스페이먼츠예요.

### 먼저 필요한 계약

- 토스페이먼츠 **전자결제** 계약 (결제창용 상점아이디, MID).
- 카카오페이·네이버페이 등 간편결제는 결제사마다 따로 심사·승인을 받아요.
- **자동결제(빌링)** 계약: 저장한 카드로 결제하거나 정기 방문을 쓸 때만. 정기결제 전용 심사가 따로 있고, MID 도 결제창과 달라요.
- 가상계좌와 현금영수증은 토스페이먼츠 상점관리자에서 설정해요.

새 카드는 PortOne 이 저장할 수 있으면 PortOne 에 저장돼요 (정기결제 또는 카드 채널 키). PortOne 이 저장할 수 없고 토스페이먼츠에 자동결제 키가 없으면 카드를 저장할 수 없어요: 카드는 매번 결제창으로 결제하고, 저장한 카드가 없는 고객은 새 정기 방문을 만들 수 없어요. 이미 저장한 카드는 저장한 곳에서 계속 결제돼요.

### 키 받기 (개발자센터)

1. developers.tosspayments.com 에 로그인 → **API 키** 메뉴 → 결제창 MID 선택 → **API 개별 연동 키**의 클라이언트 키(`test_ck_`)와 시크릿 키(`test_sk_`)를 복사해요. 위젯 연동 키(`gck`/`gsk`)가 아니에요.
2. 자동결제 계약이 있으면 자동결제 MID 를 골라 그 MID 의 `ck`/`sk` 를 복사해요. 두 칸은 함께 넣거나 함께 비워요.
3. 보안 키(64자리)는 선택이에요. 넣으면 웹훅 서명을 확인해요.
4. 네 개의 키는 모두 테스트 키이거나 모두 라이브 키여야 해요. 섞으면 저장할 때 거절돼요 (`key_mode_mismatch`).

### 콘솔 순서

스위치 이름은 **사용: 토스페이먼츠로 결제 받기** 예요. 건드리지 않고 처음 저장하면 라이브 키는 켜진 채, 테스트 키는 꺼진 채 저장되고, 그 값은 키를 바꿔도 그대로예요. 켜져 있으면 PortOne 채널 키가 없는 켜진 결제수단을 토스페이먼츠로 받고, PortOne 이 카드를 저장할 수 없으면 새 카드도 토스페이먼츠에 저장해요 (자동결제 키가 있을 때). 끄면 새 토스 결제와 새 토스 카드 저장만 멈춰요. 진행 중인 결제, 환불, 대사, 웹훅, 저장된 카드와 정기결제는 계속 토스페이먼츠에서 처리해요.

1. **사용 끄기 (이미 결제를 받는 서버만)**: 테스트 키는 사용이 꺼진 채 저장되지만, 처음 저장하기 전에 **사용**을 꺼 두면 확실해요. 켠 채로 저장하면 PortOne 채널 키가 없는 켜진 수단이 바로 고객에게 보여요.
2. **테스트 키로 저장**: **결제 (토스페이먼츠)** 카드에 테스트 키(`test_`) 네 개를 넣고 저장해요. 처음 저장은 소유자 인증 코드가 필요하고, 저장하면 카드가 잠겨요.
3. **[API 키 확인]**: 결제창 MID 키, 그다음 자동결제 MID 키를 확인해요. 자동결제 키만 틀리면 "자동결제(빌링) 키가 올바르지 않아요"가 떠요. 카드에 **테스트** / **실거래** 표시가 시크릿 키 종류대로 떠요. 이 확인은 시크릿 키만 봐요. 다른 MID 의 클라이언트 키를 넣었다면 첫 테스트 결제에서야 드러나요.
4. **웹훅 등록**: 카드 옆 복사 버튼의 주소 `<PUBLIC_BASE_URL>/v1/webhooks/tosspayments` 를 개발자센터 **웹훅** 메뉴에서 **결제창 MID** 에 등록하고, `PAYMENT_STATUS_CHANGED` 와 `DEPOSIT_CALLBACK` 이벤트를 골라요. 자동결제 MID 에는 등록하지 않아도 돼요 (자동결제 승인은 웹훅이 오지 않아요). 지급대행 주소 `/v1/webhooks/toss` 와 헷갈리지 마세요.
5. **결제 수단 고르기**: 설정 > **결제 수단**에서 토스페이먼츠와 계약·승인된 수단만 켜 두세요 (처음에는 10개 수단이 모두 켜져 있어요). 사용이 켜지면 PortOne 채널 키가 없는 켜진 수단이 모두 고객에게 토스페이먼츠로 보여요. 승인 안 된 수단을 고른 고객은 토스페이먼츠 오류를 봐요. 이 화면에 수단마다 결제사("결제사: PortOne" 또는 "결제사: 토스페이먼츠")가 보여요.
6. **켜고 시험하기**: 잠금 해제(소유자 코드) → **사용** 켜기 → 저장. "결제 연결" 제목 아래 "결제를 받는 곳"에 토스페이먼츠와 "(테스트 키: 실제 돈이 오가지 않아요)"가 보여요. 예: "결제를 받는 곳: PortOne, 토스페이먼츠 (테스트 키: 실제 돈이 오가지 않아요)". 아래 "테스트 키로 먼저 확인할 것"을 결제수단마다 해 보고, 테스트 결제는 모두 환불해요. 이미 영업 중이면 손님이 테스트 키 결제를 하지 않도록 한가한 시간에 짧게 해요.
7. **라이브 키로 바꾸기**: 잠금 해제 → 네 키를 모두 라이브 키(`live_`)로 바꿔 **사용**이 켜진 채 저장 → [API 키 확인] 을 다시 눌러요. 토스페이먼츠 옆 "테스트 키" 표시가 사라졌는지 봐요. 테스트 키로 받는 동안 받은 예약은 결제 완료로 보여도 실제 돈이 들어오지 않아요.
8. **멈추기**: 잠금 해제 → **사용** 끄기 → 저장. 바꿀 때마다 감사 기록에 남아요.

고객 앱이 오래된 버전이면: v1.5.1 스토어 앱은 PortOne 결제 페이지만 열 수 있어서, 서버가 그 앱에는 PortOne 결제수단과 PortOne 카드 저장만 보여줘요. 토스페이먼츠로 가는 수단은 그 앱에 보이지 않아요. 스토어 앱 고객도 토스페이먼츠 수단을 쓰게 하려면 새 앱을 출시하고, 콘솔 앱 버전 화면에서 고객 앱의 **최소 버전**을 올려요. 웹 앱은 항상 최신이에요.

### 무엇이 어디로 가나

"–" 는 고객에게 보이지 않는다는 뜻이에요. 모든 줄은 그 수단이 설정 > 결제 수단에서 켜져 있다고 가정해요.

| 설정 | 카드 | 카카오페이 | 계좌이체 | 새로 저장하는 카드 | 결제를 받는 곳 |
|---|---|---|---|---|---|
| 아무것도 없음 | – | – | – | 안 됨 | 아직 없음 |
| PortOne (카드·카카오페이 채널 키) | PortOne | PortOne | – | PortOne (카드 채널 키) | PortOne |
| 토스페이먼츠만, 자동결제 키 있음 | 토스페이먼츠 (저장 카드) | 토스페이먼츠 | 토스페이먼츠 | 토스페이먼츠 | 토스페이먼츠 |
| 토스페이먼츠만, 자동결제 키 없음 | 토스페이먼츠 (결제창) | 토스페이먼츠 | 토스페이먼츠 | 안 됨 | 토스페이먼츠 |
| 둘 다: PortOne 카드·카카오페이 채널 키, 토스 사용 켬 | PortOne | PortOne | 토스페이먼츠 | PortOne | PortOne, 토스페이먼츠 |
| 둘 다, PortOne `card`·`card_billing` 채널 키를 지움 | 토스페이먼츠 | PortOne | 토스페이먼츠 | 토스페이먼츠 (자동결제 키가 있을 때), 없으면 안 됨 | PortOne, 토스페이먼츠 |
| 둘 다, 토스 사용 끔 | PortOne | PortOne | – | PortOne | PortOne |
| PortOne 채널 키는 있지만 API Secret 이나 상점 아이디가 없음 | – | – | – | 안 됨 | 아직 없음 |

한 수단을 토스페이먼츠로 옮기려면 잠금을 풀고 PortOne 카드에서 그 수단 칸의 **이 칸 지우기**를 누른 뒤 저장해요. PortOne 에 정기결제(`card_billing`)나 카드(`card`) 키가 있는 동안 카드는 PortOne 에 저장하는 카드로 남고, 결제 수단 화면에도 "결제사: PortOne"으로 보여요. 카드를 토스페이먼츠로 옮기려면 두 칸을 모두 지워요.

이미 시작한 것은 옮겨지지 않아요:

| 대상 | 처리하는 곳 |
|---|---|
| 이미 시작한 결제 | 시작한 곳. 도중에 설정을 바꿔도 그 결제는 시작한 곳에서 끝나요 |
| 환불, 결제 조회, 대사 | 결제한 곳 (결제마다 기록돼 있어요) |
| 이미 저장한 카드, 정기 방문 결제 | 카드를 저장한 곳 |
| 본인인증 (PASS) | 항상 PortOne (PortOne 본인인증 채널이 필요해요) |
| 현금영수증 | PortOne 결제는 지금처럼. 토스페이먼츠 결제는 결제창(계좌이체·가상계좌)에서 바로 발급되고, 결제를 취소하면 토스페이먼츠가 함께 취소해요 |
| 지급대행 (파트너 지급) | 바뀌지 않아요 (지급대행 카드) |

그래서 **각 결제 카드는 그 결제나 저장된 카드가 남아 있는 동안 지우지 마세요.** 진행 중인 결제 (하루 안에 만료돼 늦게 결제될 수 있는 결제 포함), 저장된 카드나 남은 환불이 있으면 [연결 끊기]가 거절돼요 (`gateway_in_use`). 결제 후 1년이 안 된 결제 중 아직 환불할 수 있는 결제가 있어도 거절돼요 (`gateway_has_refundable_payments`): 지우면 그 환불이 실패하기 때문이에요. 아직 결제되지 않은 정기 청소 청구가 있어도 거절돼요 (데모 데이터 결제는 세지 않아요). 이런 결제가 남아 있는 동안 다른 PortOne 상점으로 바꾸거나, 토스페이먼츠 시크릿 키를 그 MID 의 결제를 읽지 못하는 키(다른 MID)로 바꾸는 저장도 거절돼요 (`gateway_account_changed`). 같은 계정에서 새로 받은 키로 바꾸는 건 돼요. PortOne 결제의 환불, PortOne 으로 저장한 카드, 본인인증은 PortOne 카드를 써요.

카카오페이·네이버페이·토스페이·페이코는 어느 결제사로 가든 고객 앱에서 1회 결제로 보여요 (카드 저장 화면은 카드만 등록해요). 웹과 Windows 앱에서는 카드도 매번 결제창으로 결제해요. PortOne 테스트 채널로 받은 결제나 저장한 카드는 "PortOne 테스트 채널로 결제가 들어왔어요" 알림을 띄우고, 시스템 상태의 "PortOne 테스트 채널 결제" 칸이 빨강이 돼요 (`payment_test_channel`, `portone_test_channel`). 테스트 키로 시험 중이면 괜찮아요. 운영 중이라면 채널 키가 라이브 채널 키인지 확인하고 PortOne 관리자 화면에서 이 상점의 테스트 채널을 삭제하세요. 이 서버에 넣은 키가 아닌 테스트 채널로 들어온 결제와 카드는 거절해서 예약되지 않아요 (채널 키는 공개 값이라 누구나 고를 수 있어요). 이미 PortOne 으로 저장한 간편결제는 그대로 써요.

### 테스트 키로 먼저 확인할 것

테스트 키에서는 돈이 움직이지 않아요. 가상계좌 번호는 `X` 로 시작해요. 카카오페이는 계약된 MID 키로만, 페이코는 라이브 키로만, 애플페이는 Safari 에서만 테스트돼요.

1. 결제수단마다 1회 결제: 카드, 간편결제(계약된 것), 계좌이체(현금영수증 단계 포함), 가상계좌(입금 시뮬레이션 → 결제 완료로 바뀌는지).
2. 결제 1건 전액 환불, 1건 부분 환불.
3. 결제창을 닫거나 실패했을 때 앱이 결제 취소로 돌아오는지.
4. 자동결제 키가 있으면: 카드 저장(테스트 인증번호 000000) → 그 카드로 예약 결제 → 환불 → 카드 삭제.
5. 개발자센터 웹훅 기록에서 우리 서버가 200 으로 답했는지.
6. Android, iOS 기기에서 카드사 앱, 토스, 카카오페이로 넘어갔다가 앱으로 돌아오는지.
7. 모두 통과하면 테스트 결제를 모두 환불하고, 위 "콘솔 순서" 7번대로 라이브 키로 바꿔요.

개발자가 확인할 항목(웹훅 서명 헤더, 거래 내역 조회, 자동결제 응답 시간 등)은 [payments-reference.md](payments-reference.md) 4장에 있어요.

토스페이먼츠를 멈추려면 **사용**을 끄면 돼요. 새 토스 결제와 새 토스 카드 저장만 멈추고, 이미 있는 토스 결제·환불·저장된 카드는 계속 처리돼요.

서버를 토스 이전 버전으로 내려야 할 때: 먼저 **사용**을 끄고, 토스페이먼츠 결제 중 진행 중이거나 입금을 기다리는 결제, 대기 중인 토스 환불이 하나도 없을 때까지 기다리세요. 토스 이전 버전은 토스 결제를 조회하거나 환불하지 못해요.

---

# Payment setup guide (PortOne · Toss Payments) — English

This is a **fill-in template**. Enter the real values only in the console (Settings), never in this file, code, apps, web bundles or logs.

- PortOne store id (`store-…`) and channel keys (`channel-key-…`) may reach the browser SDK. Channel key for cash receipts is server-only.
- PortOne API Secret, PortOne webhook secret (`whsec_…`), Toss secret keys (`test_sk_` / `live_sk_`) and Toss security keys are **server-only**. The Toss Payments client keys (`test_ck_` / `live_ck_`) may reach the browser.
- Never mix `test_` and `live_` keys. Changing a key requires a new connection test.

PortOne and Toss Payments are both optional: set up either one, or both. Customers are offered the methods of every gateway set up; with neither, payment is unavailable.

Console order (Settings › Payments): (1) PortOne card (optional; identity checks (PASS) need it): store id, V2 API secret, webhook secret and the channel key slots (card, card billing, KakaoPay, NaverPay, TossPay, virtual account, identity, cash receipt, bank transfer; also PAYCO, Samsung Pay, Apple Pay and phone), then Save (the first save needs the owner's code), then the owner presses Check API key. Installs that only had the card channel key keep billing and cash receipts on it. (2) Toss Payments card (optional; alone or next to PortOne): see "Taking payments with Toss Payments" below. Saved without touching its Active switch, live keys are stored on and test keys off; on a server that already takes payments switch it off before the first save anyway. The payments heading, "Payments go to: …", lists every gateway set up, PortOne first, with "test keys" next to any gateway on test keys, or says no gateway takes payments right now (also while Toss is saved with Active off). (3) Toss payout card: secret key and security key, Save, Check connection, then mode `toss`.

Each card locks after it is saved. Secrets and channel keys show only a mask and cannot be read back; identifiers such as the store id and the Toss mode show in full. To replace a value, press Unlock (owner only, 6-digit authenticator code), paste the new value and Save. An unlock ends after 5 minutes (the card shows the minutes left), or sooner with Lock now. The owner can run a connection check on a locked card; no unlock is needed. Lock, unlock, save and failed unlock attempts are written to the audit log without the values.

Webhook URLs (set in each provider's dashboard):
- PortOne: `<PUBLIC_BASE_URL>/v1/webhooks/portone`
- Toss Payments (payments): `<PUBLIC_BASE_URL>/v1/webhooks/tosspayments`, registered on the payment-window MID with `PAYMENT_STATUS_CHANGED` and `DEPOSIT_CALLBACK`. The signature is checked when the security key is set and the header is present.
- Toss payouts: `<PUBLIC_BASE_URL>/v1/webhooks/toss` (`payout.changed`, `seller.changed`), signed with `tosspayments-webhook-signature`; an invalid signature gets 401.

Webhooks never move money by themselves: the server re-reads the payment from the provider before acting.

Items still to confirm in a test store: PortOne decline codes, cash receipt response shape, Toss webhook signature against one live test webhook, Toss payout create response wrapper.

Fill-in values to provide: store id, PortOne test API secret, one channel key per method used (card, card_billing, kakaopay, naverpay, tosspay, transfer, virtual_account, identity, cash_receipt), PortOne webhook secret, Toss test secret key, Toss security key, test account notes, public base URL.

## Taking payments with Toss Payments (alone or next to PortOne)

Toss Payments (direct integration) can be the only gateway, or run next to PortOne. With both, each payment method goes to one gateway: PortOne when it has that method's channel key, otherwise Toss Payments. New saved cards go to PortOne when it can save cards (a card_billing or card channel key), otherwise to Toss Payments when it has its billing pair.

Contracts first: the Toss Payments online-payment contract (payment-window MID); each easy pay approved separately (KakaoPay and NaverPay review the merchant); an auto-billing (빌링) contract with its own MID only if you want saved cards or recurring visits; virtual-account and cash-receipt settings in the Toss merchant admin. When PortOne cannot save cards and Toss has no billing pair, cards cannot be saved: they pay in the window each time, and customers without a saved card cannot start a subscription. Cards already saved keep charging where they were saved.

Keys (developer center, API keys menu): the **API individual integration** client key (`test_ck_`) and secret key (`test_sk_`) of the payment-window MID, not the widget keys (`gck`/`gsk`); optionally the billing MID's pair (both or neither) and the 64-character security key. All keys must be test keys, or all live keys (`key_mode_mismatch` otherwise).

Console order. The switch is **Active: take payments with Toss Payments**. Saved for the first time without touching it, live keys are stored on and test keys off, and the stored value stays when keys change. On, switched-on methods without a PortOne channel key are paid through Toss Payments, and new cards are saved with Toss Payments (with its billing pair) when PortOne cannot save cards. Off stops only new Toss payments and new Toss card saves; payments in progress, refunds, reconciliation, webhooks, saved cards and subscriptions keep running on Toss Payments. (1) On a server that already takes payments, switch **Active** off before the first save (test keys are stored off anyway): saved with it on, every switched-on method without a PortOne channel key is offered to customers at once; (2) save the four test keys (owner code on the first save); (3) Check API key: the window keys, then the billing keys (`billing_invalid_key` when only the billing pair fails). It checks the secret keys only: a client key from another MID shows up only at the first test payment; (4) register `<PUBLIC_BASE_URL>/v1/webhooks/tosspayments` on the payment-window MID with `PAYMENT_STATUS_CHANGED` and `DEPOSIT_CALLBACK` (not on the billing MID: billing approvals send no webhook); (5) under Setup > **Payment methods**, leave on only the methods Toss Payments contracted and approved (all 10 are on at first): with Active on, every switched-on method without a PortOne channel key is offered through Toss Payments. Each row there shows its gateway ("Gateway: PortOne" or "Gateway: Toss Payments"); (6) unlock (owner code), switch **Active** on, Save: the heading shows Toss Payments with "(test keys: no real money moves)", for example "Payments go to: PortOne, Toss Payments (test keys: no real money moves)". Run the test list below per method and refund every test payment; (7) unlock, replace all four keys with live keys (`live_`), Save with Active on, Check API key again, and see the "test keys" note next to Toss Payments gone: bookings taken on test keys look paid but bring in no money; (8) to pause Toss Payments, unlock, switch Active off, Save. Every change is in the audit log.

Older customer apps: v1.5.1 store apps open PortOne pages only. Current apps send `?gateways=portone,tosspayments`; without it the server offers PortOne methods and PortOne card saving only, so v1.5.1 never sees a Toss method. For store users to get the Toss methods, publish the new apps and raise the customer app's **Minimum version** under App versions. The web app is always current.

Routing ("–" means not offered; every row assumes the method is switched on under Setup > Payment methods):

| Set up | card | kakaopay | transfer | New saved card | Payments go to |
|---|---|---|---|---|---|
| Nothing | – | – | – | off | none yet |
| PortOne with keys for card and kakaopay | PortOne | PortOne | – | PortOne (legacy card key) | PortOne |
| Toss only, with the billing pair | Toss, saved card | Toss | Toss | Toss | Toss Payments |
| Toss only, no pair | Toss, window | Toss | Toss | off | Toss Payments |
| Both: PortOne keys for card and kakaopay, Toss on | PortOne | PortOne | Toss | PortOne | PortOne, Toss Payments |
| Both, with the PortOne `card` and `card_billing` keys cleared | Toss | PortOne | Toss | Toss if it has the pair, else off | PortOne, Toss Payments |
| Both, Toss Active off | PortOne | PortOne | – | PortOne | PortOne |
| PortOne channel keys, but no API secret or no store id | – | – | – | off | none yet |

To move a method to Toss Payments, unlock the PortOne card, press **Clear this slot** on that key and Save. While PortOne has the `card_billing` (or `card`) key, card stays a saved card on PortOne and its row shows "Gateway: PortOne"; to send card to Toss Payments clear both.

Nothing already started moves: a payment already started, its refunds, lookups and reconciliation stay on the gateway it started on; saved cards and their subscription charges stay on the gateway they were saved with; identity checks (PASS) always use PortOne and need its identity channel; cash receipts for Toss payments come from the Toss window and are cancelled by Toss with the payment; payouts are unchanged. So **keep each gateway's card while its payments or saved cards remain**: Disconnect is refused (`gateway_in_use`) while the card has open checkouts (including checkouts that expired within the last day, which can still be paid late), live saved cards or queued refunds, and (`gateway_has_refundable_payments`) while a payment on it from the last year can still be refunded. An open subscription charge also blocks it; demo sample payments do not. While such rows remain, saving the card with another PortOne store, or a Toss secret key that cannot read that MID's newest payment (another MID), is refused too (`gateway_account_changed`); a new key of the same account is accepted. KakaoPay, NaverPay, Toss Pay and PAYCO are one-off payments on either gateway (card setup registers cards only), and on the web and Windows card is paid in the window too. A PortOne payment or saved card on a TEST channel raises its own alert, "PortOne 테스트 채널로 결제가 들어왔어요" (rule `payment_test_channel`, code `portone_test_channel`), and turns the status page's PortOne test-channel check red.

Test with test keys first (no money moves; virtual accounts start with `X`; KakaoPay needs a contracted MID key, PAYCO live keys, Apple Pay Safari): one payment per method, a full and a partial refund, a closed or failed window, a saved card (test code 000000) charge, refund and delete when billing keys are set, the webhook log showing 200, and the hand-off to card apps, Toss and KakaoPay and back on Android and iOS. Then refund every test payment and switch to live keys (console order step 7). Developer checks (webhook signature header, the transaction list, billing latency) are in payments-reference.md section 4. To pause Toss Payments, switch Active off: only new Toss payments and card saves stop. Rolling the server back to a release from before Toss: switch Active off first and wait until no Toss payment is pending or awaiting deposit and no Toss refund is queued; older releases cannot look up or refund Toss payments.
