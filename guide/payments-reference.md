# 결제 연동 참고 (PortOne V2 · 토스페이먼츠 지급대행)

# Payment integration reference (PortOne V2 · Toss Payments payouts)

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

## 2. 토스페이먼츠 지급대행 (판매자 정산 지급, 직접 호출)

지급대행은 PortOne 채널과 별개 상품이에요. `api.tosspayments.com`을 서버에서 직접 불러요. 고객 결제는 PortOne을 거치고, 정산 지급만 토스 지급대행을 써요. 두 계약이 모두 있어야 해요.
Payouts are a separate Toss product, called directly from the server. Customer payments go through PortOne; seller payouts go through Toss payouts. Both contracts are needed.

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
| 토스 시크릿 키 · 보안 키 | 토스 개발자센터 › API 키 | 서버 전용 |
| 토스 지급대행 신청 | 지급대행 서비스 이용 신청 | 정산 문의 고객센터 1544-7772 (평일 08–19시) `[요약]` |
| 웹훅 주소 등록 | PortOne 웹훅 · 토스 개발자센터 웹훅 | `/v1/webhooks/portone`, `/v1/webhooks/toss` (토스는 MID별 등록) |
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

---

# Payment integration reference (English headings)

Section 1 covers PortOne V2 (customer payments, billing, cash receipts, identity). Section 2 covers Toss Payments payouts, called directly from the server. Section 3 lists the contracts and values the owner must provide. Section 4 is the sandbox checklist above.

Rules that never change: secret keys and security keys stay on the server; browsers only receive the PortOne store id and channel keys; `test_` and `live_` keys are never mixed; changing a key means running the connection test again; webhooks never move money by themselves, the server re-reads the provider first.
