# iOS 출시 순서 (App Store · TestFlight) / iOS release steps

고객 앱 `kr.co.dahae.customer`(다해)와 파트너 앱 `kr.co.dahae.partner`(다해 파트너)를 개발자가 GitHub Actions (`store-apps`, macOS 러너) 로 빌드하고 DODA 의 Apple 개발자 계정으로 올려요 ([guide 11](../guide/11-앱-직접-빌드.md)). Mac 은 필요 없어요. 개인정보 라벨·수출 규정 답은 `app-privacy.md`, 심사 메모는 `review-notes.md`, 키 목록은 `docs/DEPLOY.md` §3 예요.
The developer builds both apps with GitHub Actions (`store-apps`, macOS runner) and uploads them to DODA's own Apple developer account ([guide 11](../guide/11-앱-직접-빌드.md)). No Mac needed. Privacy label and export answers: `app-privacy.md`; review notes: `review-notes.md`; key list: `docs/DEPLOY.md` §3.

## 1. 준비할 것 (1번은 DODA, 2–11번은 개발자) / What to prepare (item 1 DODA, items 2–11 the developer)

| # | 준비물 / Item | 어디서 / Where | 비고 / Note |
|---|---|---|---|
| 1 | Apple Developer Program 가입 (연 99 USD, 법인은 D-U-N-S 번호) | developer.apple.com/programs | 개인 계정이면 판매자 이름이 개인 이름으로 보여요. Individual accounts show a personal seller name |
| 2 | App ID 2개 등록: `kr.co.dahae.customer`, `kr.co.dahae.partner` | Certificates, Identifiers & Profiles → Identifiers | 둘 다 **Push Notifications**, **Sign in with Apple**, **Associated Domains** 켜기. 하나라도 빠지면 서명이 실패해요. Enable all three on both IDs, or signing fails |
| 3 | App Store Connect 앱 2개 만들기 (언어 한국어, 번들 ID 위 2개, SKU 자유) | appstoreconnect.apple.com → Apps → + | 앱을 먼저 만들어야 TestFlight 업로드가 돼요. The app record must exist before the first upload |
| 4 | App Store Connect API 키 (역할 **Admin**. App Manager 키는 자동 서명 내보내기에서 "Cloud signing permission error" 로 실패해요. Role **Admin**: cloud signing needs it) | Users and Access → Integrations → App Store Connect API → Team Keys | `.p8`(한 번만 내려받기), Key ID, Issuer ID 3가지. 먼저 DODA 계정 소유자가 이 화면에서 API 액세스를 요청해야 해요. The Account Holder requests API access first |
| 5 | GitHub 저장소에 넣기 | 저장소 Settings → Environments → `store` → Secrets | `ASC_KEY_ID`, `ASC_ISSUER_ID`, `ASC_KEY_P8_B64` (.p8 을 base64 로). 값은 저장소에 올리지 않아요. Enter the three values; never commit them |
| 6 | APNs 인증 키 (`.p8`) | Apple Developer → Keys → Apple Push Notifications service | Firebase 콘솔 → 프로젝트 설정 → Cloud Messaging → Apple 앱 구성에 앱마다 업로드. Upload per iOS app in Firebase |
| 7 | Firebase iOS 앱 2개 (번들 ID 동일) → `GoogleService-Info.plist` | Firebase 콘솔 | `base64 -w0` 로 바꿔 `store` 환경의 secret `GOOGLE_SERVICE_INFO_PLIST_B64`(고객), `PARTNER_GOOGLE_SERVICE_INFO_PLIST_B64`(파트너). 없으면 푸시만 꺼진 채로 빌드돼요. Without it the build works, push is off |
| 8 | 로그인 키 | `docs/DEPLOY.md` §3 표 | `GOOGLE_IOS_CLIENT_ID`(번들 ID마다 iOS OAuth 클라이언트), `KAKAO_NATIVE_KEY`(카카오 개발자 콘솔 → 플랫폼 → iOS 번들 ID 등록), 네이버 개발자센터에서 iOS 번들 ID 등록 |
| 9 | `APP_LINK_DOMAIN` (서버 도메인) + 서버 콘솔 **설정 → 앱 링크** 카드의 Apple Team ID | Apple Developer 우측 상단 Membership → Team ID ([guide 11](../guide/11-앱-직접-빌드.md) 6번 6항) | 서버가 `/.well-known/apple-app-site-association` 을 내줘야 네이버 로그인이 iOS 17.4+ 에서 돼요. 아래 4항 확인 |
| 10 | Apple 로그인 서버 키 (`.p8`) | 콘솔 Apple 카드, `review-notes.md` §3 | 탈퇴 때 Apple 토큰 해지에 쓰여요. 키 하나 (`Eric Sign in with Apple`) 로 두 앱에 모두 써요. 서버가 이 키로 서명하고 앱마다 번들 ID (`kr.co.dahae.customer`, `kr.co.dahae.partner`) 를 클라이언트 ID 로 써요. One key, `Eric Sign in with Apple`, covers both apps: the server signs with it and uses each app's bundle id as its client id |
| 11 | 스크린샷, 설명, 개인정보처리방침 URL | App Store Connect 앱 페이지 | 아이폰 6.9인치 스크린샷 필수 (`docs/store/screenshots`, 문구 `listing-*.md`). 아이패드용은 필요 없어요 (앱이 아이폰 전용) |

**이름 규칙 / Naming**: DODA 가 개발자를 초대할 때 사용자 이름은 **Eric Wijesinghe** (이름 Eric), 개발자를 위해 만드는 키는 이름에 Eric 을 넣어요 (API 키 `Eric CI`, APNs 키 `Eric APNs`, Apple 로그인 키 `Eric Sign in with Apple`). 회사 안에서 누구의 계정·키인지 헷갈리지 않게 하려는 거예요. The developer's user is invited as **Eric Wijesinghe**, and every key made for him carries "Eric" in its name, so nobody at the company mistakes whose it is.

`store` 환경의 값은 secret (키) 과 variable (`API_BASE`, `APPLE_TEAM_ID`, `APP_LINK_DOMAIN`) 로 나눠 넣어요. 키는 저장소 파일에 쓰지 않아요. Keys go into environment secrets only.

## 2. 첫 빌드 / First build

1. 저장소에서 버전 태그를 만들고 (예: `v1.6.0`, [guide 11](../guide/11-앱-직접-빌드.md) 8번) Actions → `store-apps` → Run workflow 에서 `app` = customer 또는 both, `platform` = ios 또는 both, `upload` = true 로 실행해요 (태그가 없으면 pubspec 의 버전을 써요). store 환경에 승인자를 넣었다면 승인 뒤 실행돼요. If the store environment has reviewers, one approves the run.
2. 워크플로가 하는 일 / The workflow does: `check-native.py`(Info.plist·권한·개인정보 매니페스트 검사) → `flutter build ios --config-only`(난독화, 빌드 번호는 `scripts/ci/build-number.py`) → `xcodebuild archive` 와 `-exportArchive`(자동 서명, App Store Connect 키로 프로필 생성, `-allowProvisioningUpdates`) → `check-native.py --ipa`(서명된 ipa 검사) → `upload` 가 켜져 있으면 `xcrun altool --upload-app` 로 TestFlight 업로드.
3. 빌드 번호는 시간(분) 기준 증가 값이라 같은 번호가 두 번 나오지 않아요. 직접 올리지 마세요. Build numbers rise by the minute; never set one by hand.
4. 업로드 후 Apple 처리에 10~30분. TestFlight 내부 테스터로 먼저 실기기 확인: 로그인(전화, 카카오, 네이버, 구글, Apple), 사진 고르기, 결제창(PortOne, 카드 앱 호출), 푸시, 위치.
5. 심사 제출은 **사람이 해요** (워크플로는 심사에 자동 제출하지 않아요): App Store Connect → 버전에 빌드 추가 → `review-notes.md` 내용을 심사 메모에 붙여 넣기 → 개인정보 라벨은 `app-privacy.md` 대로 입력 → 제출. Submission is manual by design.

## 3. 앱 안에서 이미 해 둔 것 / Already in the app

- 최소 iOS 15.0 (Firebase, 구글 로그인, image_picker 가 15 이상 필요). 아이폰 전용.
- Apple 로 로그인 (지침 4.8), 앱 안 계정 삭제 (5.1.1(v)), 후기 신고 (1.2), 개인정보 매니페스트 `PrivacyInfo.xcprivacy`, `ITSAppUsesNonExemptEncryption = false`.
- 권한 문구는 한국어·영어 2개 언어 (나머지 12개는 `archive/i18n-extra-languages`에 보관). 위치는 "사용 중에만"이고 고객 앱만 (geolocator 는 `BYPASS_PERMISSION_LOCATION_ALWAYS=1` 로 빌드해 항상 허용 요청을 뺐어요). 카메라 문구는 image_picker 가 카메라 API 를 링크하므로 남겨 둔 것이에요 (`review-notes.md` §5).
- 결제는 PortOne 웹 화면이고 인앱 결제(IAP)가 아니에요: 방문 청소는 앱 밖에서 쓰는 실물 서비스 (지침 3.1.3(e)).
- 키는 저장소에 없어요. `scripts/ci/native-keys.sh` 가 빌드 때 `keys.json`, `Keys.xcconfig`, `GoogleService-Info.plist` 를 만들고 모두 `.gitignore` 예요.

## 4. 확인 명령 / Checks

```
curl -s https://<도메인>/.well-known/apple-app-site-association     # applinks 와 webcredentials 둘 다 보여야 해요
python -I scripts/ci/check-native.py                                # 저장소 검사 (CI 가 매번 실행)
python -I scripts/ci/check-native.py --ipa <Runner.ipa>             # 빌드된 ipa 검사 (store-apps 가 실행)
```

AASA 는 기기가 Apple CDN 을 거쳐 받아서 바꾼 뒤 반영까지 시간이 걸려요. 테스트 중에는 entitlement 값 끝에 `?mode=developer` 를 붙이고 기기 설정 > 개발자 > Associated Domains Development 를 켜는 방법이 있어요 (출시 빌드에는 붙이지 마세요). AASA changes reach devices through Apple's CDN with a delay.

## 5. 이 환경에서 확인하지 못한 것 / Not verifiable without a real build

Windows 에서는 Xcode 가 없어서 아래는 첫 store-apps 실행과 TestFlight 에서만 알 수 있어요. None of this can be tested without Xcode.

- SwiftPM 플러그인 해석 (Firebase 등을 GitHub 에서 받아요), 서명, 용량.
- iOS 에서 카드 앱(ISP, KB, 토스 등) 호출과 복귀: 호스팅 결제 페이지(`website/pay/index.html`)는 `redirectUrl` 만 써요. PortOne/PG 가 iOS 에서 앱 복귀용 `appScheme` 을 요구하면 앱 URL 스킴 추가와 페이지 수정이 필요해요. TestFlight 에서 카드 결제를 꼭 한 번 해 보세요.
- 네이버 로그인 (iOS 17.4+ https 콜백): AASA + `webcredentials` 가 Apple 쪽에서 확인되어야 해요. 17.4 미만은 `dahaenaver://` 스킴을 써요.
- `GoogleService-Info.plist` 없이 빌드하면 푸시는 꺼져요. 푸시를 쓰려면 7번을 먼저 하세요.
