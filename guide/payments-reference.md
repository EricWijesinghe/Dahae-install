# 결제 연동 참고 (PortOne V2 · 토스페이먼츠 결제 · 토스페이먼츠 지급대행)

# Payment integration reference (PortOne V2 · Toss Payments payments · Toss Payments payouts)

이 문서는 공식 문서와 명세에서 확인한 사실만 모아요. 키 값은 적지 않아요.
This page lists only facts confirmed against the official docs and specs. It never holds key values.

확인 기준: 공식 문서·OpenAPI 명세 (2026-11). `[확인]` 표시는 원문을 직접 읽어 확인한 항목, `[요약]` 표시는 요약본 기준이라 테스트 상점에서 한 번 더 확인할 항목이에요.
Basis: official docs and OpenAPI spec (2026-11). `[verified]` = read in the original page; `[summary]` = taken from a summary, confirm in a test store.

---

## 1. PortOne V2 (고객 결제 · 자동결제 · 현금영수증 · 본인인증)

### 인증 · 호출 · Authentication and calls
- API 주소: `https://api.portone.io`. 헤더: `Authorization: PortOne <API_SECRET>` `[확인]`
- Bearer는 액세스 토큰에만 써요. PG사 시크릿을 넣으면 401이 나요 `[확인]`
- 오류 본문: `{ "type": "<ErrorName>", "message": "..." }` `[확인]`
- 상태 코드: 400·401·403은 모든 연산에 있어요. 쓰기 연산은 404·409·502도 있어요. 502는 PG 쪽 실패라 결과가 불확실해요 `[확인]`
- 결제 취소 요청에는 멱등성 키 필드가 없어요. `currentCancellableAmount`로 취소 가능 금액을 맞춰 보내요 `[확인]`

### 결제 상태 · Payment status
- `READY | PENDING | VIRTUAL_ACCOUNT_ISSUED | PAID | FAILED | PARTIAL_CANCELLED | CANCELLED` `[확인]`

### 웹훅 · Webhooks
- 규격: Standard Webhooks. 헤더 `webhook-id`, `webhook-timestamp`, `webhook-signature` `[확인]`
- 시크릿: `whsec_` + base64. 활성 시크릿은 최대 2개(교체용) `[확인]`
- 서명 대상: `{webhook-id}.{webhook-timestamp}.{원문 본문}`, HMAC-SHA256, base64, `v1,<서명>` `[확인]`
- 본문: `{ type, timestamp, data: { storeId, paymentId, transactionId, cancellationId?, billingKey? } }` `[확인]`
- 결제 이벤트: `Transaction.Ready, Paid, VirtualAccountIssued, PartialCancelled, Cancelled, Failed, PayPending, CancelPending, DisputeCreated, DisputeResolved`. 빌링키 이벤트: `BillingKey.Ready, Issued, Failed, Deleted, Updated` `[확인]`
- `Transaction.PartialCancelled`는 부분 환불이 일어났을 때 와요 `[확인]`
- 모르는 `type`이나 필드는 무시해야 해요 `[확인]`
- 5xx를 받으면 PortOne이 다시 보내요. 수동 재전송 API: `POST /payments/{id}/resend-webhook` `[확인]`
- 선택 사항: 발신 IP `52.78.5.241` 허용 목록 `[확인]`

### 브라우저 SDK · Browser SDK
- 브라우저에는 상점 아이디(`store-…`)와 채널 키(`channel-key-…`)만 보내요. API Secret과 웹훅 시크릿은 서버에만 둬요 `[확인]`
- 결제 요청 주요 값: `storeId, channelKey, paymentId, orderName, totalAmount, currency: "CURRENCY_KRW", payMethod, locale: "KO_KR"` `[확인]`
- 간편결제(카카오페이·네이버페이·토스페이 등)는 `payMethod: EASY_PAY` + `easyPay.easyPayProvider`로 골라요 `[확인]`

### 기타 엔드포인트 · Other endpoints
- 결제 조회 `GET /payments/{paymentId}`, 취소 `POST /payments/{paymentId}/cancel`, 빌링키 결제 `POST /payments/{paymentId}/billing-key`, 빌링키 발급·조회·삭제 `/billing-keys` `[확인]`
- 현금영수증 발급 `POST /cash-receipts`는 응답에 `status`가 없어요. `issueNumber, url, pgReceiptId`가 와요 `[확인]`
- 본인인증 `/identity-verifications/{id}/send · confirm · resend` `[확인]`

---

## 1b. 토스페이먼츠 결제 (직접 연동, 단독 또는 PortOne과 함께)

PortOne 과 토스페이먼츠는 둘 다 선택이에요. 콘솔 **결제 (토스페이먼츠)** 카드의 **사용: 토스페이먼츠로 결제 받기** 스위치는 기본으로 켜져 있어요. 켜져 있으면 PortOne 채널 키가 없는 켜진 결제수단이 토스페이먼츠로 가고, PortOne 이 카드를 저장할 수 없으면 새 카드도 토스페이먼츠에 저장돼요. 끄면 새 토스 결제와 새 토스 카드 저장만 멈춰요. 이미 시작한 결제, 환불, 저장된 카드는 처음 처리한 곳을 따라가요 ([설정 가이드 6절](payments-setup.md#6-토스페이먼츠로-결제받기-단독-또는-portone과-함께)).
PortOne and Toss Payments are both optional. The Toss Payments card's switch "Active: take payments with Toss Payments" is on by default: switched-on methods without a PortOne channel key go to Toss Payments, and new cards are saved there when PortOne cannot save cards. Off stops only new Toss payments and card saves; existing payments, refunds and saved cards stay on their gateway.

### 키와 인증 · Keys and authentication
- **API 개별 연동 키**만 써요: 클라이언트 키 `test_ck_` / `live_ck_`(브라우저 SDK), 시크릿 키 `test_sk_` / `live_sk_`(서버). 위젯 키(`gck`/`gsk`)는 주문번호 조회와 부가 API를 못 써서 받지 않아요 `[확인]`
- 키는 **MID마다** 따로 나와요. 결제창 MID와 자동결제(빌링) MID는 달라요 `[확인]`
- 인증: `Authorization: Basic base64("{시크릿키}:")` `[확인]`
- 테스트 키와 라이브 키는 섞을 수 없어요. 섞으면 `INVALID_API_KEY` / `UNAUTHORIZED_KEY` `[확인]`
- 보안 키(64자리 16진수)는 웹훅 서명 HMAC에 써요 `[확인]`

### 결제 흐름 · Payment flow (Dahae)
- 결제창: V2 SDK `TossPayments(clientKey).payment({customerKey: ANONYMOUS}).requestPayment({...})`. 간편결제는 `method: "CARD"` + `card.flowMode: "DIRECT"` + `card.easyPay`로 바로 열어요. 앱의 결제수단 선택 화면이 그대로 유지돼요 `[확인]`
- 승인: 성공 주소로 `paymentKey, orderId, amount`가 와요. 10분 안에 `POST /v1/payments/confirm {paymentKey, orderId, amount}`로 승인해야 돈이 움직여요 `[확인]`. Dahae는 `/pay/toss-success.html`이 `POST /v1/checkouts/{id}/toss-confirm`을 부르고, 서버가 저장된 금액으로 승인해요.
- `orderId`는 항상 우리 id(결제 id, `{kind}-{uuid}`, `subvisit-{visit}-{n}`)예요. 6–64자, `[A-Za-z0-9_=-]` `[확인]`. `paymentKey`는 저장하지 않고 `GET /v1/payments/orders/{orderId}`로 다시 찾아요.

### 연산 대응표 · Operations (PortOne → Toss Payments)
| 연산 | PortOne V2 | 토스페이먼츠 |
|---|---|---|
| 키 확인 (콘솔) | `GET /payments/malkkeum-setup-check` (404 = 정상) | `GET /v1/payments/orders/dahae-setup-check`, MID마다 (404 = 정상, 401 = 키 오류) |
| 결제창 | `PortOne.requestPayment` | `payment().requestPayment` (결제창, 간편결제는 자체창) |
| 승인 | 없음 (SDK가 끝냄) | `POST /v1/payments/confirm` (10분 안) |
| 조회 | `GET /payments/{paymentId}` | `GET /v1/payments/orders/{orderId}` (그 결제 MID의 sk) |
| 전액·부분 환불 | `POST /payments/{id}/cancel {amount, reason, currentCancellableAmount}` | 조회 → `POST /v1/payments/{paymentKey}/cancel {cancelReason, cancelAmount, refundableAmount}` |
| 카드 등록 화면 | `PortOne.requestIssueBillingKey` | `payment({customerKey}).requestBillingAuth({method: "CARD"})` → `authKey` |
| 빌링키 | `GET /billing-keys/{key}` (발급 확인) | `POST /v1/billing/authorizations/issue {authKey, customerKey}` → `billingKey` (다시 조회할 수 없음) |
| 저장 카드 결제 | `POST /payments/{pid}/billing-key` | `POST /v1/billing/{billingKey} {customerKey, amount, orderId, orderName}` (최대 60초, 웹훅 없음) |
| 저장 카드 삭제 | `DELETE /billing-keys/{key}` | `DELETE /v1/billing/{billingKey}` |
| 웹훅 | `/v1/webhooks/portone`, Standard Webhooks | `/v1/webhooks/tosspayments`: `PAYMENT_STATUS_CHANGED`, `DEPOSIT_CALLBACK` |
| 일일 대사 | `GET /payments?from&until&page` | `GET /v1/transactions?startDate&endDate&startingAfter&limit`, MID마다 |
| 현금영수증 | `POST /cash-receipts` + 취소 | 결제창의 `cashReceipt`(계좌이체·가상계좌). 결제를 취소하면 토스가 함께 취소 `[확인]` |
| 본인인증 (PASS) | `/identity-verifications/{id}` | 없음: 항상 PortOne |

### 결제 상태 · Payment status
- `READY, IN_PROGRESS, WAITING_FOR_DEPOSIT, DONE, CANCELED, PARTIAL_CANCELED, ABORTED, EXPIRED` `[확인]`
- 서버는 PortOne 상태로 바꿔 써요: DONE → `PAID`, WAITING_FOR_DEPOSIT → `VIRTUAL_ACCOUNT_ISSUED`, CANCELED → `CANCELLED`, PARTIAL_CANCELED → `PARTIAL_CANCELLED`, ABORTED·EXPIRED → `FAILED`, READY·IN_PROGRESS → `READY`.
- 금액: `totalAmount`(결제 금액), `balanceAmount`(남은 금액). 환불된 금액 = `totalAmount - balanceAmount` `[확인]`

### 오류 · Errors
- 본문: `{"code": "...", "message": "..."}` (v1). v2 API는 `{version, traceId, error: {code, message}}` `[확인]`

| 토스 | 서버 처리 |
|---|---|
| 401 `UNAUTHORIZED_KEY`, 400 `INVALID_API_KEY` | 설정 안 됨 (503 `provider_not_configured`) |
| 404 `NOT_FOUND_PAYMENT` (조회) | 결제 없음 |
| 404 `NOT_FOUND_PAYMENT_SESSION` (승인: 인증 10분 지남), 승인의 다른 4xx | 거절 (402), 돈은 안 움직임 |
| 400 `ALREADY_PROCESSED_PAYMENT` (승인) | 주문번호로 조회해서 그 상태를 씀 |
| 400 `DUPLICATED_ORDER_ID` / `ALREADY_PROCESSED_PAYMENT` (저장 카드 결제) | 조회만 하고 다시 청구하지 않음 |
| 409 `IDEMPOTENT_REQUEST_PROCESSING`, 400 `ALREADY_PROCESSING_REQUEST`, 429 | 나중에 다시 또는 조회 |
| 400 `NOT_MATCHES_REFUNDABLE_AMOUNT`, `ALREADY_CANCELED_PAYMENT` (환불) | 남은 금액을 다시 읽어 이미 환불됐으면 완료, 아니면 직원 확인으로 넘김 |
| 403 `REJECT_CARD_COMPANY` 등 다른 4xx | 거절 |
| 5xx, `FAILED_*_PROCESSING`, `COMMON_ERROR`, 연결 실패, 시간 초과 | 일시 오류 (503), 청구는 조회로 확인 |

### 멱등성 · Idempotency
- `Idempotency-Key` 헤더는 보내지 않아요. 오류 응답이 15일 동안 캐시되면 정당한 재시도가 막히기 때문이에요. 대신 결제마다 다른 `orderId`(토스가 같은 주문 두 번 승인을 거절), 환불의 `refundableAmount`, 결제 상태로 한 번만 처리해요.

### 웹훅 · Webhooks
- 개발자센터 › 웹훅에서 **MID마다** 등록해요. 로컬 주소는 등록할 수 없어요 `[확인]`
- `PAYMENT_STATUS_CHANGED`: `{eventType, createdAt, data: <Payment>}`. 저장 카드(빌링) 승인에는 오지 않아요 `[확인]`
- `DEPOSIT_CALLBACK`: `{createdAt, secret, status, transactionKey, orderId}` (eventType 없음) `[확인]`
- 헤더 `tosspayments-webhook-signature: v1:<base64>`: HMAC-SHA256 over `{원문 본문}:{tosspayments-webhook-transmission-time}`, 보안 키 `[확인]`. 결제 이벤트에 늘 오는지는 문서에 없어요 `[확인 필요]`
- `eventId`가 없어요. 서버는 본문을 힌트로만 쓰고, 시크릿 키로 다시 조회해서 처리해요 `[확인]`
- 10초 안에 200을 받지 못하면 1, 4, 16, 64, 256, 1024, 4096분 간격으로 최대 7번 다시 보내요 `[확인]`

### 테스트 환경 · Test environment
- 테스트 키에서는 돈이 움직이지 않아요. 가상계좌는 `X`로 시작해요 `[확인]`
- 카카오페이는 계약된 MID 키로만, 페이코는 라이브 키로만, 애플페이는 Safari에서만 테스트돼요 `[확인]`
- `TossPayments-Test-Code: <오류 코드>` 헤더로 테스트에서 오류를 일부러 낼 수 있어요 `[확인]`

---

## 2. 토스페이먼츠 지급대행 (판매자 정산 지급, 직접 호출)

지급대행은 PortOne 채널과 별개 상품이에요. `api.tosspayments.com`을 서버에서 직접 불러요. 고객 결제는 PortOne 또는 토스페이먼츠 결제(1b절)를 거치고, 정산 지급은 어느 쪽이든 토스 지급대행을 써요. 결제와 지급대행은 계약이 따로예요. 지급대행 웹훅은 `/v1/webhooks/toss`, 결제 웹훅은 `/v1/webhooks/tosspayments`예요.
Payouts are a separate Toss product, called directly from the server. Customer payments go through PortOne or Toss Payments (section 1b); seller payouts always go through Toss payouts. Payments and payouts are separate contracts with separate webhook URLs.

### 키 · Keys
- 시크릿 키 접두사: `test_sk_` (테스트), `live_sk_` (실거래). 클라이언트 키 `test_ck_` / `live_ck_`는 브라우저 SDK용이에요 `[확인]`
- 인증: `Authorization: Basic base64("{시크릿키}:")`. 콜론이 필요해요 `[확인]`
- 보안 키: 64자리 16진수, 시크릿 키와 별개예요. 암호화와 웹훅 서명에 써요 `[확인]`
- 키 값은 서버 설정에만 두고, 브라우저 번들이나 `NEXT_PUBLIC_` 같은 공개 접두사 변수에 넣지 않아요 `[확인]`

### 암호화 · Encryption
- 판매자 생성·수정, 지급 생성은 JWE(compact, `alg=dir`, `enc=A256GCM`)로 주고받아요. 헤더 `TossPayments-api-security-mode: ENCRYPTION`, 커스텀 보호 헤더 `iat`(ISO 8601)와 `nonce` `[확인]`
- 키는 보안 키를 16진수 디코딩한 32바이트예요 `[확인]`
- 지급 취소(POST, 본문 없음)와 잔액 조회(GET)는 암호화하지 않아요 `[확인]`
- 응답은 `{ version, traceId, entityType, entityBody: {...} }`로 감싸져 와요. 지급 생성 응답은 `entityBody.items[0]`에 지급 객체가 있어요 `[확인]`
- 멱등성 헤더: `Idempotency-Key`, 최대 300자, 첫 요청 후 15일 유효 `[확인]`

### 지급 · Payout create
- 요청 본문 예시는 배열 `[ {...} ]`이고 한 번에 최대 100건이에요. 한 건이 실패하면 전체가 실패해요 `[확인]`
- 항목: `refPayoutId`(최대 50자), `destination`(판매자 id), `scheduleType`(`EXPRESS` | `SCHEDULED`), `payoutDate`(SCHEDULED일 때 `yyyy-MM-dd`), `amount: {currency: "KRW", value}`, `transactionDescription`(최대 7자) `[확인]`
- 건당 10억 원 미만 `[확인]`
- EXPRESS(바로지급): 영업일 08:00–15:00에만 돼요. 공휴일에는 오류가 나요 `[확인]`
- SCHEDULED: `payoutDate`는 1년 안의 미래 영업일이어야 해요 `[확인]`
- 잔액이 부족하면 요청 시점에 실패해요 `[요약]`
- 지급 상태: `REQUESTED, IN_PROGRESS, COMPLETED, FAILED, CANCELED, DELETED, REJECTED` `[요약]`
- 취소는 `REQUESTED` 상태에서만 돼요. EXPRESS는 배치 때 `IN_PROGRESS`가 되어 사실상 취소할 수 없어요 `[요약]`

### 판매자 · Sellers
- 상태: `APPROVAL_REQUIRED`(지급 불가, 본인인증 필요) → `PARTIALLY_APPROVED`(주 1천만 원까지) → `KYC_REQUIRED`(한도 초과 시도, KYC 전까지 지급 차단) → `APPROVED`(한도 없음) `[확인]` (상세 수치는 `[요약]`)
- 법인 판매자는 등록 직후 `PARTIALLY_APPROVED` 수준으로 시작해요 `[확인]`
- 사업자등록번호는 바꿀 수 없어요. 바꾸려면 판매자를 새로 만들어요 `[확인]`
- 생성 필드: `refSellerId`(7–20자), `businessType`(`INDIVIDUAL` | `INDIVIDUAL_BUSINESS` | `CORPORATE`), `company` 또는 `individual`, `account`(은행 코드, 계좌번호 숫자 최대 14자, 예금주 최대 50자) `[요약]`

### 잔액 · Balance
- `GET /v2/balances`: `pendingAmount`(정산 주기 2일 전부터 갱신), `availableAmount`(지급 가능 금액) `[확인]`
- 테스트 환경은 `availableAmount`만 돌려줘요 `[확인]`

### 웹훅 · Webhooks
- 이벤트 이름: `payout.changed`, `seller.changed` `[확인]`
- 헤더: `tosspayments-webhook-signature`(`v1:<base64>`, 여러 개면 쉼표로 구분), `tosspayments-webhook-transmission-time`, `tosspayments-webhook-transmission-id`, `tosspayments-webhook-transmission-retried-count` `[요약]`
- 서명 방식: HMAC-SHA256 over `{원문 본문}:{transmission-time}`, 보안 키로 서명, base64 `[요약]`
- 본문: `eventType, createdAt, version, eventId, entityType, entityBody` `[요약]`
- `eventId`는 재시도에도 같아요. 중복 처리는 `eventId`로 막아요 `[요약]`
- COMPLETED와 FAILED에 `payout.changed`가 와요. DELETED는 알림이 없어요 `[요약]`

### 테스트 환경 · Test environment
- 테스트에서는 `CORPORATE` 판매자만 등록할 수 있고, 사업자등록번호가 유효해야 해요 `[확인]`
- 테스트에서는 KYC를 끝낼 수 없어서 `APPROVED`에 도달할 수 없어요 `[확인]`
- 실패 테스트 계좌: `77701777777`(우리종합금융 295), `3025353430761`(농협 011), `02004240994312`(산업은행 002) `[확인]`
- 테스트 환경 호출 제한: API당 분당 100회 `[확인]`
- 실거래 지급 테스트에는 수수료가 나가요 `[확인]`

---

## 3. 필요한 계약과 값 · Required contracts and values

| 항목 | 어디서 | 메모 |
|---|---|---|
| PortOne 상점 아이디 | PortOne 콘솔 › 상점 ID | 브라우저에 전달 가능 |
| PortOne V2 API Secret | 콘솔 › 연동 관리 › API Key | 서버 전용. PG 시크릿 아님 |
| PortOne 웹훅 시크릿 | 콘솔 › 연동 관리 › 웹훅 | `whsec_…`, 서버 전용 |
| PortOne 채널 키 | 콘솔 › 채널 관리 | 카드, 간편결제, 계좌이체, 가상계좌, 빌링, 본인인증, 현금영수증 |
| PortOne ↔ 토스 PG 계약 | 채널 관리 | 면세·복합과세·에스크로는 토스와 별도 계약이 필요 `[요약]` |
| 토스페이먼츠 전자결제 계약 | 토스페이먼츠 | 결제창 MID. 간편결제는 결제사마다 따로 승인 |
| 토스페이먼츠 자동결제 계약 (선택) | 토스페이먼츠 | 저장 카드·정기 방문용. 별도 심사, 별도 MID |
| 토스페이먼츠 결제 키 | 토스 개발자센터 › API 키 › API 개별 연동 키 | MID마다 `ck`(브라우저) + `sk`(서버 전용) |
| 토스 시크릿 키 · 보안 키 | 토스 개발자센터 › API 키 | 서버 전용 |
| 토스 지급대행 신청 | 지급대행 서비스 이용 신청 | 정산 문의 고객센터 1544-7772 (평일 08–19시) `[요약]` |
| 웹훅 주소 등록 | PortOne 웹훅 · 토스 개발자센터 웹훅 | `/v1/webhooks/portone`, `/v1/webhooks/tosspayments` (결제창 MID), `/v1/webhooks/toss` (지급대행) (토스는 MID별 등록) |
| 판매자 정보 | 판매자별 | 사업자등록번호(법인·개인사업자) 또는 이름·전화(개인, 본인인증), 검증된 계좌 |
| 사업자등록증, 통신판매업 신고 | 사장님 | 실거래 결제에 필요 |

- 토스 지급대행과 결제 MID가 같은 계약인지는 영업 담당에게 확인해요 `[확인 필요]`

---

## 4. 아직 확인하지 않은 항목 · To confirm in a sandbox

테스트 상점과 테스트 키로 아래를 먼저 확인하고, 그다음 실거래로 넘어가요.
Confirm these in a test store with test keys before any live use.

1. PortOne: 테스트와 실거래를 상점 ID나 채널 키 형식만으로 구분할 수 있는지. 확인 전까지 콘솔은 PortOne 카드에 모드를 표시하지 않아요.
2. PortOne: 웹훅 타임스탬프 허용 범위. Standard Webhooks 기본값(5분)을 가정했고, 문서에는 명시되지 않았어요.
3. PortOne: 카드 거절 코드, 빌링 청구 실패 코드. 명세에는 오류 이름만 있어요.
4. PortOne: 현금영수증 발급 응답 형식 (`cashReceipt.issueNumber`).
5. PortOne: 502 응답 뒤 결제 결과를 어떻게 확인하는지.
6. 토스: 웹훅 서명 키. 코드는 보안 키 문자열 그대로를 키로 써요. 디코딩한 바이트를 써야 하는지 테스트 웹훅 1건으로 확인해요.
7. 토스: 웹훅 재시도 간격과 필요한 응답 코드. 문서에 명시되지 않았어요.
8. 토스: 지급 생성 요청 본문을 배열로 보내야 하는지 객체로도 되는지. 예시는 배열이에요.
9. 토스: 오류 응답 객체의 구조. 가이드는 요청·응답 페이지로만 연결해요.
10. 토스: EXPRESS를 공휴일·영업시간 밖에 보냈을 때의 응답.
11. 토스: 주 1천만 원 한도에서 KYC_REQUIRED로 바뀌는 시점과 알림 방식.
12. 토스: 지급 상태 값 전체 목록과 `error` 필드의 사유 형식.
13. 토스: 개인 판매자(본인인증) 흐름은 테스트 환경에서 확인할 수 없어요. 사장님 확인이 필요해요.
14. 토스페이먼츠 결제: 결제수단별 결제창 (간편결제 자체창 코드, 계좌이체·가상계좌의 현금영수증 단계), 실패 주소에 `paymentId`가 남는지.
15. 토스페이먼츠 결제: 승인 10분 제한, 두 번 승인할 때 `ALREADY_PROCESSED_PAYMENT`, 금액을 바꿨을 때의 오류 코드, 주문번호 조회가 `IN_PROGRESS`/`ABORTED` 결제도 돌려주는지.
16. 토스페이먼츠 결제: `PAYMENT_STATUS_CHANGED`와 `DEPOSIT_CALLBACK`에 서명 헤더가 오는지, HMAC 키 바이트, 가상계좌 입금 시뮬레이션과 실제 `bankCode`, 예금주 표시.
17. 토스페이먼츠 자동결제: 테스트 인증번호 000000, 발급 응답 형식, 청구 응답 시간(서버는 15초 기다린 뒤 주문번호로 조회하고, 늦게 승인된 청구는 환불 대기열이 환불해요. 요청 전체가 30초에 끊기기 때문이에요), 삭제 동작, customerKey 길이 제한.
18. 토스페이먼츠: `/v1/transactions` 날짜 형식과 페이지. 개발자센터에서 API 버전을 2022-11-16(현재 최신)으로 정해 기록해요. 코드가 읽는 필드: `status`, `totalAmount`, `balanceAmount`, `orderId`, `paymentKey`, `virtualAccount.*`, `easyPay.provider`.
19. 토스페이먼츠: 결제 페이지의 CSP 위반이 없는지 (데스크톱, 모바일), Android·iOS에서 카드사 앱·토스·카카오페이로 넘어갔다가 `malkkeumpay://`로 돌아오는지.
20. 토스페이먼츠 결제: 승인 재요청에 오는 코드(`ALREADY_COMPLETED_PAYMENT`, `DUPLICATED_REQUEST`, `FORBIDDEN_CONSECUTIVE_REQUEST`). 서버는 이 코드를 거절이 아니라 "조회 후 판단"으로 다뤄요.
21. 토스페이먼츠 가상계좌: 입금 오류로 `DONE` 이 `WAITING_FOR_DEPOSIT` 으로 돌아갈 때 `DEPOSIT_CALLBACK` 이 오는지. 서버는 결제 분쟁 케이스를 열고 알림을 보내요.
22. Android 기기에서 iframe 안의 카드사 앱 링크(ISP/페이북, KB Pay, 신한, 삼성, 현대, NH, 롯데, 하나, 우리, 토스, 카카오페이, 백신 앱)가 열리는지. `webview_flutter_android` 4.x 는 iframe 이동을 앱에 넘기지 않아, 앱이 그 로드 오류에서 링크를 읽어 열어요. 개발자 출시 전 확인 항목이에요.

---

# Payment integration reference (English headings)

Section 1 covers PortOne V2 (customer payments, billing, cash receipts, identity). Section 1b covers Toss Payments payments (direct integration, alone or next to PortOne: each method goes to PortOne when it has that method's channel key, otherwise to Toss Payments; identity stays on PortOne). Section 2 covers Toss Payments payouts, called directly from the server. Section 3 lists the contracts and values the owner must provide. Section 4 is the sandbox checklist above.

Rules that never change: secret keys and security keys stay on the server; browsers only receive the PortOne store id and channel keys, or the Toss Payments client key; `test_` and `live_` keys are never mixed; changing a key means running the connection test again; webhooks never move money by themselves, the server re-reads the provider first.
