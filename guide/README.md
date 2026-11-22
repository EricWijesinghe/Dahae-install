# 다해 안내서 · Dahae guides

[English](#english) · 한국어

처음이라면 **0번 한 장**만 따라 하면 설치가 끝나요. Start with guide 0 if this is your first setup.

출시 계획 (안드로이드 · iOS · 서버 배포) · [Launch plan](../LAUNCH-PLAN.md)

| 번호 | 안내서 · Guide | 한 줄 설명 · What it covers | 누가 · Who | 언제 · When |
|---|---|---|---|---|
| 0 | [한 번에 설치하기](00-한번에-설치.md) · *Install in one go* | 서버 빌리기, 도메인 연결, 설치 명령 한 줄, 사장님 계정까지 | 사장님 또는 설치 담당 | 처음 한 번 (약 30분) |
| 1 | [서버 설치 (자세히)](01-서버-설치.md) · *Server setup in detail* | 서버 사양, 방화벽, 설치가 막힐 때의 자세한 순서 | 설치 담당 | 0번에서 막혔을 때 |
| 2 | [설정 마법사](02-설정-마법사.md) · *Setup wizard* | 사업장 정보, 결제, 문자, 간편 로그인, 외부 백업 연결 | 설치 담당 + 사장님 | 설치 직후 |
| 3 | [앱 출시](03-앱-출시.md) · *App store release* | App Store, Google Play에 고객·파트너 앱을 올리는 순서 | 설치 담당 → 개발자 | 스토어에 앱을 올릴 때 |
| 4 | [사장님 가이드](04-사장님-가이드.md) · *Owner's daily guide* | 콘솔 하루 사용법: 견적, 파트너, 직원, 개인정보 요청 | 사장님 | 매일 5분 |
| 6 | [알림톡 템플릿 등록](06-알림톡-템플릿-등록.md) · *KakaoTalk AlimTalk templates* | 카카오 알림톡 템플릿 7종 등록과 설정 칸 입력 | 설치 담당 | 알림톡을 쓸 때 |
| 7 | [운영 명령어](07-운영-명령어.md) · *Operations commands* | `dahae` 명령: 상태, 로그, 백업, 복구, 업데이트, 되돌리기 | 설치 담당 | 백업, 업데이트, 문제 확인 |
| 8 | [내 클라우드 계정에 올리기](08-클라우드-배포.md) · *Deploying on your own cloud account* | 클라우드 서버 만들기, 도메인과 DNS, 설치 명령, 백업 저장소 연결까지 전 과정 | 사장님 또는 설치 담당 | 서버를 직접 빌려 처음 설치할 때 (약 1시간) |
| 9 | [클라우드별 설치](09-클라우드별-설치.md) · *Install by cloud provider* | 업체별로 붙여넣는 칸, 방화벽, 고정 IP, 브라우저 SSH 한 줄 | 사장님 또는 설치 담당 | 서버를 만들 때 |
| 10 | [AWS 에 올리고 자동 배포하기](10-AWS-배포.md) · *Deploying on AWS with automatic deploys* | 담당 IAM 계정, CloudFormation 으로 서버 만들기, GitHub 승인 배포와 되돌리기 | 사장님, 담당자, 개발자 | AWS 를 쓰는 회사가 한 번 (약 2시간) |
| 11 | [스토어 앱 빌드 (개발자)](11-앱-직접-빌드.md) · *Store app builds (developer, GitHub Actions)* | 저장소 `store` 환경 설정, 스토어 계정과 키, `store-apps` 실행, Play·TestFlight 업로드, 심사 제출 | 개발자 (DODA 계정 권한으로) | 앱을 처음 스토어에 올릴 때와 업데이트할 때 |
| - | [결제 연동 설정](payments-setup.md) · *Payment setup (PortOne · Toss Payments)* | 결제와 지급대행 키 발급 순서, 웹훅 점검, 테스트 결제 | 설치 담당 | 결제 카드를 채울 때 |
| - | [결제 연동 참고](payments-reference.md) · *Payment integration reference* | 결제·지급대행 API 연동 사양과 확인 필요 항목 | 개발자 | 결제 로직을 점검할 때 |

## 급할 때 · When something is wrong

| 상황 · Situation | 볼 곳 · Where to look |
|---|---|
| 사이트가 안 열려요 · The site does not open | [7. 운영 명령어](07-운영-명령어.md) 1번 `status`, 2번 `logs` |
| 업데이트 뒤에 이상해요 · Something broke after an update | [7. 운영 명령어](07-운영-명령어.md) 6번 `rollback` |
| 데이터를 되살려야 해요 · Restore data from a backup | [7. 운영 명령어](07-운영-명령어.md) 4번 `restore` |
| 설치 코드를 잃어버렸어요 · Lost the setup code | 서버에서 `sudo dahae setup-code` · run on the server |
| 결제 키 값이 맞는지 모르겠어요 · Not sure the payment keys are right | [결제 연동 설정](payments-setup.md) 3번 웹훅 점검 · section 3, webhook check |

## 엔지니어링 문서 · Engineering docs

설치와 운영을 넘어서 구조와 근거가 궁금하다면 저장소의 문서를 보세요. 한국어 요약을 먼저 두었어요.

| 문서 | 내용 |
|---|---|
| [기능 카탈로그](../FEATURES.md) | 기능별 설명과 코드 위치, 알려진 한계 |
| [아키텍처](../ARCHITECTURE.md) | 구성 요소, 데이터 흐름, 배포 구조 |
| [보안과 개인정보](../SECURITY-AND-PRIVACY.md) · [보안 정책](../../SECURITY.md) | 보안 통제, 개인정보 흐름, 취약점 신고 |
| [운영 가이드](../OPERATIONS.md) · [장애 대응 런북](../RUNBOOK.md) | 설치 뒤 운영과 증상별 복구 |
| [품질과 검사](../QUALITY.md) · [API 색인](../api/README.md) | 테스트와 CI, 경로 목록 |

---

<a id="english"></a>

## English

Guides are written in Korean for the business owner. The two payment guides are bilingual.
Start with **guide 0** (install in one go), then **guide 2** (setup wizard) and **guide 4** (daily console use).

| No. | Guide | What it covers | Who | When |
|---|---|---|---|---|
| 0 | [Install in one go](00-한번에-설치.md) (Korean) | Renting a server, connecting a domain, the one-line install command, creating the owner account | Owner or installer | Once, about 30 minutes |
| 1 | [Server setup in detail](01-서버-설치.md) (Korean) | Server size, firewall, step-by-step fallback when guide 0 gets stuck | Installer | When guide 0 is blocked |
| 2 | [Setup wizard](02-설정-마법사.md) (Korean) | Business details, payments, SMS, social sign-in, offsite backup | Installer and owner | Right after install |
| 3 | [App store release](03-앱-출시.md) (Korean) | Publishing the customer and partner apps to the App Store and Google Play | Installer, then developer | When the apps go to the stores |
| 4 | [Owner's daily guide](04-사장님-가이드.md) (Korean) | Daily console use: quotes, partners, staff, privacy requests | Owner | Five minutes a day |
| 6 | [KakaoTalk AlimTalk templates](06-알림톡-템플릿-등록.md) (Korean) | Registering the seven AlimTalk templates and entering them in the wizard | Installer | When AlimTalk is used |
| 7 | [Operations commands](07-운영-명령어.md) (Korean) | The `dahae` command: status, logs, backup, restore, update, rollback | Installer | Backups, updates, troubleshooting |
| 8 | [Deploying on your own cloud account](08-클라우드-배포.md) (Korean) | Creating the cloud server, domain and DNS, the install command, offsite backup | Owner or installer | Once, when you set up your own server (about an hour) |
| 9 | [Install by cloud provider](09-클라우드별-설치.md) (Korean) | Which field to paste into for each provider, firewall, static IP, one-line browser SSH fallback | Owner or installer | When creating the server |
| 10 | [Deploying on AWS with automatic deploys](10-AWS-배포.md) (Korean) | The deployer IAM account, launching the server with CloudFormation, approved GitHub deploys and rollback | Owner, installer, developer | Once, for a company that runs on AWS (about two hours) |
| 11 | [Store app builds (developer, GitHub Actions)](11-앱-직접-빌드.md) (Korean, English section included) | Setting up the `store` environment, store accounts and keys, the `store-apps` runs, Play and TestFlight uploads, review submission | Developer, using DODA's store accounts | First store release and each update |
| - | [Payment setup (PortOne · Toss Payments)](payments-setup.md) (bilingual) | Key order, webhook check, test payments | Installer | When filling the payment card |
| - | [Payment integration reference](payments-reference.md) (bilingual) | PortOne V2 and Toss payout API notes, items still to confirm in a sandbox | Developer | When reviewing payment code |

## Engineering docs (English)

- [Feature catalogue](../FEATURES.md) (Korean and English), [Architecture](../ARCHITECTURE.md), [Security and privacy](../SECURITY-AND-PRIVACY.md), [Security policy](../../SECURITY.md)
- [Operations guide](../OPERATIONS.md), [Incident runbook](../RUNBOOK.md), [Quality and checks](../QUALITY.md), [API index](../api/README.md)

[한국어로 돌아가기](#다해-안내서--dahae-guides)
