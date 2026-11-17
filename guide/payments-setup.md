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
| 토스 지급대행 시크릿 키 | `test_sk_…` / `live_sk_…` | 콘솔 › 지급대행 › 시크릿 키 | **서버 전용** |
| 토스 보안 키 (64자리 16진수) | (토스 개발자센터) | 콘솔 › 지급대행 › 보안 키 | **서버 전용** (JWE 암호화 + 웹훅 서명) |

규칙: `test_` 키와 `live_` 키는 섞지 않아요. 키를 바꾸면 [연결 확인]를 다시 해야 해요.

## 1. 채워 넣을 값 (받으면 여기에 표시)

| 항목 | 상태 | 값 (마스킹 후 기록) |
|---|---|---|
| PortOne 상점 아이디 | 대기 | `<STORE_ID>` |
| PortOne 테스트 API Secret | 대기 | `<PORTONE_TEST_API_SECRET>` (서버 전용) |
| PortOne 채널 키 · 카드 | 대기 | `<CHANNEL_KEY_CARD>` |
| PortOne 채널 키 · 간편결제 | 대기 | `<CHANNEL_KEY_EASY_PAY>` |
| PortOne 채널 키 · 자동결제(빌링키) | 대기 | `<CHANNEL_KEY_BILLING>` |
| PortOne 채널 키 · 본인인증 | 대기 | `<CHANNEL_KEY_IDENTITY>` |
| PortOne 채널 키 · 현금영수증 | 대기 | `<CHANNEL_KEY_CASH_RECEIPT>` (서버 전용) |
| PortOne 웹훅 시크릿 | 대기 | `<PORTONE_WEBHOOK_SECRET>` (서버 전용) |
| 토스 테스트 시크릿 키 | 대기 | `<TOSS_TEST_SECRET_KEY>` (서버 전용) |
| 토스 보안 키 | 대기 | `<TOSS_SECURITY_KEY>` (서버 전용) |
| 테스트 계정 정보 | 대기 | `<TEST_ACCOUNT_NOTE>` |
| 공개 주소 (PUBLIC_BASE_URL) | 대기 | `<PUBLIC_BASE_URL>` |

## 2. 콘솔 설정 순서 (붙여넣고 잠그기)

콘솔 **설정 › 결제 연결** 에서 카드별로 값을 붙여넣어요. 칸 오른쪽의 눈 아이콘으로 입력 중인 값을 가리거나 보여줄 수 있어요.

1. **PortOne** 카드: 상점 아이디, V2 API Secret, 웹훅 시크릿, 그리고 결제수단별 채널 키 칸을 채우고 **저장**.
   - 채널 키 칸: 일반결제(카드), 정기결제(빌링키), 카카오페이, 네이버페이, 토스페이, 가상계좌, 본인인증, 현금영수증. 쓰지 않는 칸은 비워 두세요. 채널 키가 없는 결제수단은 고객 앱에 보이지 않아요 (개발 모드 제외).
   - 각 칸 아래 한 줄 설명이 값을 어디서 복사하는지 알려줘요. 형식이 맞지 않으면 저장 전에 칸 아래에 문구가 떠요.
   - 카드를 처음 저장하는 것은 소유자 인증 코드가 필요해요. 결제 금액이 가는 곳을 정하기 때문이에요.
   - [API 키 확인]은 API Secret만 확인해요. 상점 아이디나 채널 키가 틀린 것은 첫 결제 시도 때 드러날 수 있어요.
   - 기존 설치 호환: 카드 채널 키만 넣었던 설치는 정기결제가 그 `card` 키를 그대로 써요. 정기결제 칸을 채우면 그 칸이 우선해요. 현금영수증은 `cash_receipt` 칸이 없으면 결제수단(계좌이체·가상계좌)의 채널 키를 써요.
   - PortOne 콘솔(V2 웹훅)에 카드 옆 복사 버튼의 **웹훅 주소** `<PUBLIC_BASE_URL>/v1/webhooks/portone` 을 등록해요.
2. **지급대행 (토스)** 카드: 시크릿 키(`test_sk_` 또는 `live_sk_`)와 보안 키(64자리)를 붙여넣고 **저장** → [연결 확인] → 성공 후 지급 방식을 `toss` 로 바꿔요.
   - 카드의 **테스트** / **실거래** 표시는 시크릿 키 종류에서 와요. 테스트 통과가 실서비스 키를 열어주지 않아요.
   - 저장하면 카드가 잠기고 [연결 확인]도 잠금 해제 중에만 할 수 있어요. 연결 확인은 기록을 남기므로 잠긴 카드에서는 막아요.
   - 토스 개발자센터 웹훅 주소: `<PUBLIC_BASE_URL>/v1/webhooks/toss` (`payout.changed`, `seller.changed`). 요청에는 `tosspayments-webhook-signature` 헤더가 있어야 하고, 서명이 틀리면 401 이에요.
3. **잠금**: 저장하면 카드가 **잠김** 상태가 돼요. 비밀 값(API Secret, 시크릿 키, 웹훅 시크릿, 보안 키, 채널 키)은 `••••` 로만 보이고 일부도 보이지 않아요. 상점 아이디와 지급 방식처럼 식별용 값만 그대로 보여요.
   - 값을 바꾸려면 **잠금 해제**를 눌러요. 소유자 계정만 가능하고, 소유자 인증 앱의 6자리 코드가 필요해요. 새 값을 붙여넣고 **저장**하면 다시 잠겨요.
   - 잠금 해제는 5분 뒤 자동으로 잠겨요. 카드에 남은 시간이 "N분 후 잠겨요"로 보여요. 바로 잠그려면 **잠그기**를 눌러요.
   - 잠금, 잠금 해제, 저장, 실패한 해제 시도는 감사 기록(감사 로그)에 남아요. 기록에는 값이 들어가지 않아요.
4. 저장 뒤 [설정] 화면의 상태가 "연결됨"인지 확인해요.

## 3. 웹훅 점검

| 공급자 | 주소 | 검증 | 실패 시 |
|---|---|---|---|
| PortOne | `/v1/webhooks/portone` | Standard Webhooks 서명 (`webhook-signature`) | 401, PortOne이 재전송 |
| 토스 | `/v1/webhooks/toss` | `tosspayments-webhook-signature` (보안 키 기반 HMAC-SHA256) | 401 |

웹훅은 결제 상태를 바로 바꾸지 않아요. 알림을 받으면 서버가 결제 기관에 다시 조회해서 확인해요.

## 4. 테스트 계정과 테스트 결제

- PortOne 테스트 상점 채널 키로 카드 결제 1회, 부분 환불 1회를 해 보세요. 부분 환불 뒤 `refunded_amount` 가 바뀌지 않는지 확인해요 (환불 금액은 예약 환불 경로에서만 기록돼요).
- 자동결제(빌링키)는 테스트 상점에서 한 번 발급·결제·삭제까지 확인한 뒤 실서비스에 열어요.
- 현금영수증은 설정의 현금영수증 스위치가 꺼진 상태가 기본이에요. 테스트 상점에서 한 번 발급 응답 형식을 확인한 뒤 켜요.
- 테스트 계정 정보는 `<TEST_ACCOUNT_NOTE>` 칸에만 메모하고, 이 문서나 저장소에는 올리지 않아요.

## 5. 아직 확인하지 않은 항목

- PortOne 결제 거절 코드: 명세의 `type` 이름만 알려져 있어요. 실제 카드 거절 코드는 테스트 상점에서 한 번 확인해야 해요.
- 현금영수증 발급 응답 (`cashReceipt.issueNumber`): 테스트 상점 1회 발급으로 확인해요.
- 토스 웹훅 서명 계산: 명세대로 구현했어요. 테스트 웹훅 1건으로 서명이 맞는지 확인해요.
- 토스 지급 생성 응답의 래퍼 형식(`entityBody`): 두 형식 모두 읽도록 했어요. 실제 응답을 한 번 확인해요.

---

# Payment setup guide (PortOne · Toss Payments) — English

This is a **fill-in template**. Enter the real values only in the console (Settings), never in this file, code, apps, web bundles or logs.

- PortOne store id (`store-…`) and channel keys (`channel-key-…`) may reach the browser SDK. Channel key for cash receipts is server-only.
- PortOne API Secret, PortOne webhook secret (`whsec_…`), Toss secret key (`test_sk_` / `live_sk_`) and Toss security key are **server-only**.
- Never mix `test_` and `live_` keys. Changing a key requires a new connection test.

Console order (Settings › Payments): (1) PortOne card: store id, V2 API secret, webhook secret and the channel key slots (card, card billing, KakaoPay, NaverPay, TossPay, virtual account, identity, cash receipt), then Save (the first save needs the owner's code) and, while unlocked, Check API key. Installs that only had the card channel key keep billing and cash receipts on it. (2) Toss payout card: secret key and security key, Save, Check connection, then mode `toss`.

Each card locks after it is saved. Secrets and channel keys show only a mask and cannot be read back; identifiers such as the store id and the Toss mode show in full. To replace a value, press Unlock (owner only, 6-digit authenticator code), paste the new value and Save. An unlock ends after 5 minutes (the card shows the minutes left), or sooner with Lock now. Connection checks run only while a card is unlocked. Lock, unlock, save and failed unlock attempts are written to the audit log without the values.

Webhook URLs (set in each provider's dashboard):
- PortOne: `<PUBLIC_BASE_URL>/v1/webhooks/portone`
- Toss payouts: `<PUBLIC_BASE_URL>/v1/webhooks/toss` (`payout.changed`, `seller.changed`), signed with `tosspayments-webhook-signature`; an invalid signature gets 401.

Webhooks never move money by themselves: the server re-reads the payment from the provider before acting.

Items still to confirm in a test store: PortOne decline codes, cash receipt response shape, Toss webhook signature against one live test webhook, Toss payout create response wrapper.

Fill-in values to provide: store id, PortOne test API secret, channel keys (card, easy pay, billing, identity, cash receipt), PortOne webhook secret, Toss test secret key, Toss security key, test account notes, public base URL.
