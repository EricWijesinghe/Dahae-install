# 10. AWS 에 다해 서버 올리고 자동 배포하기

[English](#english) · 한국어

이 안내서는 다해 서버를 **DODA 의 AWS 계정**에 올리고, 개발자의 GitHub 저장소(`EricWijesinghe/Dahae`)에서 새 버전이 배포되도록 연결하는 전 과정입니다. AWS 계정과 비용은 DODA 명의이고, 배포와 유지보수는 개발자가 해요. DODA 가 원하면 배포 승인자를 한 명 넣을 수 있어요 (C 단계). DODA 가 할 일(A 단계)은 **화면의 버튼을 누르는 순서대로** 따라 하면 되고, 명령은 입력하지 않아요. 명령은 개발자가 B-1 (템플릿 만들기) 과 서버 안 (`dahae setup-code`) 에서만 씁니다.

> 이 안내서가 가리키는 파일: [deploy/aws/dahae.cfn.yaml](../../deploy/aws/dahae.cfn.yaml) (서버 한 대를 만드는 AWS 템플릿), [deploy/aws/deployer-iam-policy.json](../../deploy/aws/deployer-iam-policy.json) (담당 계정의 권한), [deploy/aws/dahae-boundary-policy.json](../../deploy/aws/dahae-boundary-policy.json) (서버 역할의 권한 한도), [.github/workflows/deploy-aws.yml](../../.github/workflows/deploy-aws.yml) (자동 배포). 채워진 템플릿 `dahae-aws.yaml` 은 B-1 에서 만들어요.

## 목차

- 준비물과 역할 나누기
- A. 담당 계정 만들기 (DODA 가 할 일)
- B. 서버 만들기 (CloudFormation)
- C. 자동 배포 연결하기 (GitHub)
- D. 백업, 모니터링, 비용, 삭제
- English

---

## 준비물과 역할 나누기

| 준비물 | 누가 | 메모 |
|---|---|---|
| AWS 계정 (결제 수단 등록 완료) | DODA | 회사 이름으로 만드세요 |
| IAM 사용자 1명 (담당 계정) | DODA 가 만들어 개발자에게 줘요 (개발자가 이 계정으로 B 단계를 해요. 새 버전 배포는 GitHub 로 해요) | 아래 A 단계 |
| 권한 한도 정책 `dahae-boundary` | DODA 가 한 번 만들어요 (누구에게도 붙이지 않아요) | 아래 A 단계 6번 |
| 도메인 (예: `app.example.co.kr`) | 개발자 (Cloudflare 계정으로 구매) | 고객이 들어갈 주소. DODA 와 정해요 |
| 도메인 DNS (A 레코드) | 개발자 (Cloudflare) | A 레코드 한 줄. 앞부분 없는 도메인(예: `example.co.kr`)이면 `@` 와 `www` 두 줄. Cloudflare 는 회색 구름 (DNS only) |
| GitHub 저장소 `EricWijesinghe/Dahae` | 개발자 | 개발자가 관리해요. DODA 가 배포 승인자를 원하면 그 사람도 이 저장소 접근 권한이 있어야 해요 |
| 인증 앱 (Google Authenticator 등) | 사장님 | 사장님 계정 로그인과 승인 코드에 써요 |
| 인증 앱 | 개발자 | 담당 계정(IAM 사용자)의 MFA. A 단계 10번에서 통화하며 설정해요 |
| 비밀번호 관리 앱 | DODA | 계정 정보는 채팅에 쓰지 마세요 |

**보안 원칙:** 비밀번호와 MFA 설정 키는 채팅, 메일 본문, 문서에 적지 않습니다. 비밀번호 관리 앱에 저장하고, 필요하면 전화 등 다른 경로로 전달하세요. AWS 액세스 키(장기 키)는 만들지 않습니다. GitHub 배포는 키 없이 임시 권한으로 돌아갑니다.

---

## A. 담당 계정 만들기 (DODA 가 할 일)

AWS 콘솔(https://aws.amazon.com/console/)에 회사 계정으로 로그인한 뒤 진행하세요.

> **빠른 방법 (3·7번을 명령 한 줄로):** 지역을 서울로 바꾼 뒤 콘솔 위쪽의 **CloudShell** 아이콘(`>_`)을 누르고 아래 한 줄을 붙여 넣으면 정책 두 개(`dahae-boundary`, `dahae-deployer-policy`)가 만들어져요. 그다음 4·5·8–11번만 하면 돼요.
> ```
> curl -fsSL https://raw.githubusercontent.com/EricWijesinghe/Dahae/master/deploy/aws/launch.sh | bash -s -- --admin
> ```
> 예전에 정책을 만들어 둔 계정도 이 줄을 **한 번 다시** 실행하세요: 있는 정책은 새 판으로 바뀌어요 (v1.6.0 보다 새 템플릿은 외부 백업 저장소와 알림 메일을 만들어서 권한이 조금 늘었어요).

1. 오른쪽 위 지역을 **아시아 태평양(서울) ap-northeast-2** 로 바꿉니다.
2. 검색창에 **IAM** 을 입력해 엽니다.
3. **권한 한도 정책 먼저 (한 번만):** 왼쪽 **정책 > 정책 생성 > JSON** 을 누르고, 다해 저장소의 `deploy/aws/dahae-boundary-policy.json` 내용을 전부 붙여 넣습니다. **다음**, 정책 이름: **정확히 `dahae-boundary`**, **정책 생성**. 이 정책은 사용자에게 붙이지 **않아요.** 템플릿이 만드는 서버 역할들이 이 한도 안에서만 움직이고, 담당 계정은 이 한도가 없는 역할을 만들 수 없고 이 정책을 고칠 수도 없어요. 이름이 다르면 B 단계의 스택 만들기가 실패해요.
4. 왼쪽 **사용자 > 사용자 생성** 을 누릅니다. 사용자 이름: `dahae-deployer` (또는 원하는 이름). **AWS Management Console 에 대한 사용자 액세스 권한 제공** 을 체크합니다.
5. 암호는 **사용자 지정 암호** 를 고르고 강한 암호를 만듭니다. 비밀번호 관리 앱에 저장하세요. "다음 로그인 시 암호 재설정 필요" 는 체크를 해제해도 되고 켜도 됩니다.
6. **권한 설정** 에서 **직접 정책 연결** 을 고르고 **정책 생성** 을 누릅니다 (새 탭이 열려요).
7. **JSON** 탭을 누르고, 다해 저장소의 `deploy/aws/deployer-iam-policy.json` 내용을 전부 붙여 넣습니다. **다음**, 정책 이름: `dahae-deployer-policy`, **정책 생성**.
8. 이전 탭으로 돌아와 목록을 새로 고침하고 `dahae-deployer-policy` 와 AWS 관리형 정책 **`AWSCloudShellFullAccess`** (B 단계의 명령 한 줄에 필요) 를 체크한 뒤 **다음 > 사용자 생성**. (`dahae-boundary` 는 체크하지 마세요.)
9. 사용자 목록에서 방금 만든 사용자를 눌러 **보안 자격 증명 > 다단계 인증(MFA) 할당** 을 누르고 **인증 앱** 을 고릅니다.
10. 이 단계는 **개발자와 통화하며** 해요: 개발자가 화면의 QR 코드를 자기 휴대폰 인증 앱으로 읽고, 나온 코드 두 개를 불러 주면 입력합니다. (담당 계정의 정책은 스스로 MFA 를 등록할 권한이 없어서 DODA 가 등록해 줘야 해요. 인증 앱은 개발자 휴대폰에 있어요.)
   - MFA 를 켜기 전에는 이 계정으로 서버 관련 작업이 거부돼요. 정책이 MFA 없이는 막도록 되어 있어요. 이건 의도된 동작입니다.
   - 액세스 키는 **만들지 마세요.**
11. 개발자에게 전달할 것: 계정 로그인 주소 (`https://<12자리 계정 번호>.signin.aws.amazon.com/console`), 사용자 이름, 비밀번호 (별도 경로). MFA 는 10번에서 개발자 휴대폰에 이미 설정됐어요.

**English:** First create the customer-managed policy `dahae-boundary` (exact name) from `deploy/aws/dahae-boundary-policy.json` and attach it to nobody: it is the permissions boundary the template puts on its roles, and the deployer can neither create a role without it nor edit it. Then create one IAM user for the developer, enable console access, paste `deploy/aws/deployer-iam-policy.json` as a second customer-managed policy, and attach only that one. Assign the authenticator-app MFA on a call with the developer, who scans the QR code with the developer's own phone and reads back the two codes (the policy does not let the user register MFA alone). Do not create access keys. Send the sign-in URL and user name through a channel other than chat, and send the password separately.

---

## B. 서버 만들기 (CloudFormation)

개발자가 위 계정으로 로그인하고 MFA 코드를 넣습니다. 지역이 **서울(ap-northeast-2)** 인지 다시 확인하세요.

> **한 번에 만들기 (권장):** 콘솔 위쪽의 **CloudShell** 아이콘(`>_`)을 누르고, 도메인과 이메일만 바꿔서 아래 한 줄을 붙여 넣어요. 최신 릴리스의 설치 스크립트를 릴리스의 `SHA256SUMS` 와 대조한 뒤 템플릿에 고정하고, 기본 VPC 의 공개 서브넷을 찾아 스택 `dahae-main` 을 만들고, 끝나면 서버 IP 와 다음 할 일을 보여 줘요 (약 5–10분). 그다음 B-3 (도메인) 부터 하면 돼요. 아래 B-1·B-2 는 같은 일을 손으로 하는 방법이에요.
> ```
> curl -fsSL https://raw.githubusercontent.com/EricWijesinghe/Dahae/master/deploy/aws/launch.sh | bash -s -- --domain example.co.kr --email owner@example.co.kr
> ```
> - 아직 도메인이 없으면 `--domain auto`: `<서버 IP>.sslip.io` 임시 주소로 열고 DNS 설정은 건너뛰어요. 진짜 도메인은 나중에 콘솔에서 옮겨요.
> - Route 53 에 도메인이 있으면 `--hosted-zone-id <ID>` 를 붙이면 A 레코드도 만들어요. 사양은 `--instance-type t3.large` 처럼 바꿔요. 실행 전 확인만 하려면 `--dry-run`.
> - 이미 있는 스택은 건드리지 않아요. 새 버전 설치는 C 단계의 GitHub 배포로 해요.
> - CloudShell 대신 **Windows 의 Git Bash** 에서도 돼요 (AWS CLI 를 설치하고 `aws configure` 로 담당 계정에 로그인한 뒤). 파일 경로와 한글 템플릿은 스크립트가 알아서 Windows 용으로 넘겨요.

### B-1. 템플릿 파일 준비 (개발자)

가장 쉬운 길은 위의 `launch.sh` 예요. 템플릿 파일을 직접 올리려면 GitHub 저장소의 **Releases → 최신 버전(Latest) → Assets** 에서 `dahae-aws.yaml` 을 내려받아요. 릴리스 작업이 `master` 의 템플릿에 그 릴리스의 설치 스크립트 주소와 지문을 채워 만든 파일이라 그대로 써요.

(선택) 템플릿을 직접 만들 수도 있어요 (Git Bash 나 Linux 셸, `gh` 로그인 상태).

```bash
git clone https://github.com/EricWijesinghe/Dahae.git && cd Dahae   # master
gh release download v1.6.1 -R EricWijesinghe/Dahae -p install.sh -p SHA256SUMS -p SHA256SUMS.bundle -D rel-v1.6.1
bash scripts/release/verify-blob.sh rel-v1.6.1/SHA256SUMS EricWijesinghe/Dahae   # cosign 이 있으면: 릴리스 서명 확인
(cd rel-v1.6.1 && sha256sum -c --ignore-missing SHA256SUMS)                       # install.sh: OK
dist=https://raw.githubusercontent.com/EricWijesinghe/Dahae-install           # 공개 배포 저장소 (DIST_REPO)
bash scripts/release/build-aws-template.sh rel-v1.6.1/install.sh "$dist/v1.6.1/install.sh" out
```

- 마지막 줄이 `aws template written: out/dahae-aws.yaml (install.sh sha256 <지문>)` 로 나와요. 이 지문이 `SHA256SUMS` 의 `install.sh` 줄과 같아야 해요.
- 주소는 반드시 태그(`/v1.6.1/`)가 들어간 주소여야 해요. 스크립트가 `main` 같은 브랜치 주소는 거부해요.
- 만든 `out/dahae-aws.yaml` 을 B-2 에서 올립니다.
- (저장소 안의 `deploy/aws/dahae.cfn.yaml` 은 빈칸이 있는 원본이라 그대로는 못 써요.)

### B-2. 스택 만들기

1. 검색창에 **CloudFormation** 을 입력해 엽니다.
2. **스택 > 스택 생성 > 새 리소스 사용 (표준)** 을 누릅니다.
3. **템플릿 준비 > 템플릿 파일 업로드 > 파일 선택** 에서 `dahae-aws.yaml` 을 고르고 **다음**.
4. **스택 이름: 반드시 `dahae-` 로 시작** (예: `dahae-doda`). 권한 정책이 `dahae-*` 이름만 허용하므로 다른 이름이면 만들 수 없어요.
5. 매개변수를 채웁니다.

| 항목 | 넣을 값 |
|---|---|
| Domain | 고객이 쓸 주소 (예: `app.example.co.kr`). `https://` 없이 |
| Email | 인증서 안내와 **서버 이상 알림 메일**을 받을 이메일 (아래 8번에서 한 번 확인해요) |
| HostedZoneId | Route 53 에 도메인을 두었다면 호스팅 영역 ID (Z 로 시작). 아니면 **비워 두기** |
| InstanceType | `t3.medium` (x86, 기본값). 처음에는 기본값을 쓰세요. `t4g` 는 ARM 이라 바꾸면 서버가 새로 만들어져요 |
| DiskSizeGiB | `50` (기본값). 사진이 많으면 늘리세요 |
| VpcId | 기본 VPC (default) 를 고르면 됩니다 |
| SubnetId | 그 VPC 의 **인터넷으로 나갈 수 있는 서브넷** 하나. 모르면 기본 서브넷 중 하나 |
| SshCidr | **비워 두기** (SSH 를 열지 않아요. 접속은 AWS 웹 화면으로 합니다) |
| ExistingBackupBucket | **비워 두기.** 서버를 잃고 새로 만들 때만 예전 스택 출력의 `BackupBucket` 이름 (D-1) |
| GitHubRepo | **기본값 `EricWijesinghe/Dahae` 를 그대로 두세요.** 이 저장소의 `deploy-aws` 워크플로만 이 서버에 배포할 권한을 얻어요. 다른 주소를 넣지 마세요 |
| GitHubRepoIds | **기본값 `EricWijesinghe@48539716/Dahae@1452913067` 그대로.** GitHub 가 배포 로그인에 쓰는 고정 번호예요 (저장소 이름이 바뀌어도 그대로). 개발자는 `gh api repos/EricWijesinghe/Dahae --jq '.owner.id,.id'` 로 확인할 수 있어요 |
| GitHubEnvironment | `doda-aws` (기본값 그대로, C 단계와 같은 이름이어야 해요) |
| CreateGitHubOidcProvider | 이 계정에 GitHub 연결이 처음이면 `true`. 이미 있다면 `false` |
| AmdAmiId, ArmAmiId | 바꾸지 마세요 (기본값이 Ubuntu 24.04 를 찾아요) |

6. **다음 > 다음**. 맨 아래 **AWS CloudFormation 이 IAM 리소스를 생성할 수 있음을 승인합니다** 를 체크하고 **제출**.
7. 상태가 `CREATE_IN_PROGRESS` → `CREATE_COMPLETE` 로 바뀔 때까지 기다립니다 (약 3–5분).
8. **알림 메일 확인 (클릭 한 번):** Email 로 **AWS Notification - Subscription Confirmation** 메일이 와요. 메일의 **Confirm subscription** 을 한 번 누르세요. 누르지 않으면 서버 이상 알림(하드웨어 점검 실패와 자동 복구)이 오지 않아요. 메일이 안 보이면 스팸함을 보세요. (CloudShell 의 한 줄로 만들었을 때도 같아요.)

> 서버가 `CREATE_COMPLETE` 가 되어도 **설치는 아직 진행 중**이에요. 서버 안에서 설치가 약 15–20분 걸립니다.

### B-3. 도메인 연결

도메인은 개발자의 **Cloudflare** 계정에 있어요. 그래서 보통은 아래 두 번째 경우예요.

- **Route 53 을 쓴 경우 (HostedZoneId 입력):** `Domain` 의 A 레코드는 자동으로 만들어져요. 앞부분 없는 도메인이면 `www` A 레코드 한 줄은 직접 추가하세요 (같은 IP).
- **Cloudflare (또는 가비아, 후이즈 등) 를 쓴 경우:** 스택의 **출력(Outputs)** 탭에서 `PublicIp` 값을 복사합니다. DNS 관리 화면에서 **A 레코드** 를 추가합니다.
  - 호스트(이름): 도메인 앞부분 (예: `app`). 도메인에 앞부분이 없으면 (예: `example.co.kr`) `@` 와 `www` 두 줄을 만들어요.
  - 형식: `A`
  - 값: `PublicIp` 에 적힌 IP
  - TTL: 300 (또는 기본값)
  - Cloudflare 는 **프록시 상태를 회색 구름 (DNS only)** 으로 두세요. 서버가 직접 인증서를 받고 고객 연결을 받아요.
- DNS 반영까지 몇 분~1시간 걸릴 수 있어요. 인증서는 자동으로 발급됩니다.

### B-4. 설치 끝나는지 확인

약 20분 뒤 브라우저에서 `https://<도메인>/admin` 을 엽니다. 사장님 계정 설정 화면이 나오면 설치가 끝난 것입니다. 화면이 안 나오면 아래 **B-5** 로 서버 상태를 봅니다.

### B-5. 서버에 웹 화면으로 접속하기 (SSH 없이)

1. AWS 콘솔에서 **EC2 > 인스턴스** 로 갑니다. 이름이 `dahae-dahae-…` 인 서버를 체크합니다.
2. 위의 **연결** 버튼 > **세션 관리자(Session Manager)** 탭 > **연결**.
3. 검은 화면이 열리면 아래를 입력합니다 (`sudo` 는 필요해요).

```
sudo dahae status
```

- `외부 백업`, `비밀 키 사본` 같은 줄에 주의 표시가 있으면 D 단계의 설명대로 처리하세요.
- 연결 버튼이 안 보이면 인스턴스 상태가 `실행 중` 인지, 2–3분 뒤 다시 확인하세요.

### B-6. 사장님 계정 설치 코드 받기

서버 화면(B-5)에서:

```
sudo dahae setup-code
```

12자리 코드가 `XXXX-XXXX-XXXX` 형식으로 나옵니다. 이 코드는 **처음 보인 뒤 2시간 동안만** 씁니다. 지났으면 `sudo dahae setup-code --renew` 로 새로 받으세요.

아래 1–4번은 **DODA 사장님이** 해요. 통화 중에 개발자가 코드를 알려 주면 사장님이 바로 진행해요 (인증 앱은 DODA 휴대폰). 개발자가 이 계정을 만들면 결제 카드를 지키는 소유자 코드가 개발자 휴대폰에 생겨요.

1. 브라우저에서 `https://<도메인>/admin` 을 엽니다.
2. 설치 코드를 입력하고 **사장님 계정**(이메일, 비밀번호)을 만듭니다.
3. 인증 앱에 2단계 인증을 등록합니다. 복구 코드는 종이에 적어 서버 밖에 보관하세요.
4. 설정 화면에서 회사 정보, 결제, 문자 연결을 순서대로 진행합니다 ([02번 안내서](02-설정-마법사.md)).

---

## C. 자동 배포 연결하기 (GitHub)

새 버전은 개발자 저장소의 `deploy-aws` 워크플로로 서버에 들어가요. `doda-aws` 환경에 승인자를 넣었다면 승인 후에만 들어가요. 승인자는 DODA 가 원할 때 넣어요.

### C-1. GitHub 환경(Environment) 만들기 (개발자 저장소)

이 단계는 개발자가 `EricWijesinghe/Dahae` 의 GitHub 웹 화면에서 합니다.

1. https://github.com/EricWijesinghe/Dahae 를 엽니다 (개발자 계정으로).
2. **Settings > Environments > New environment** 를 누르고 이름을 `doda-aws` 로 입력 (B-2 의 GitHubEnvironment 와 같아야 해요).
3. **Deployment branches** 를 **Selected branches** 로 바꾸고 `master` 하나만 추가합니다 (꼭 하세요). 다른 브랜치에서는 이 환경을 쓸 수 없어요.
4. (선택) DODA 가 승인하기를 원하면 **Required reviewers** 를 켜고 DODA 담당자를 추가합니다. (승인자가 없으면 개발자가 실행하는 즉시 배포돼요.)
5. **Environment variables** 에 아래 네 개를 **Add variable** 로 넣습니다. 값은 B-2 스택의 **출력(Outputs)** 탭에 있어요.

| 변수 이름 (Name) | 출력 탭의 값 | 예시 |
|---|---|---|
| `AWS_REGION` | `AwsRegion` | `ap-northeast-2` |
| `AWS_DEPLOY_ROLE_ARN` | `DeployRoleArn` | `arn:aws:iam::123456789012:role/dahae-…` |
| `AWS_INSTANCE_ID` | `InstanceId` | `i-0123456789abcdef0` |
| `DAHAE_DOMAIN` | `DahaeDomain` | `app.example.co.kr` |

- 이 값들은 **Secrets 가 아니라 Variables** 에 넣습니다. 비밀번호나 키가 아니에요.
- 변수 이름은 대소문자까지 똑같이 쓰세요.
- **CD 워크플로(`deploy-aws.yml`)는 `EricWijesinghe/Dahae` 에서 돌아요.** 환경과 변수 네 개도 이 저장소에 있어야 해요. 승인 신뢰(OIDC)는 B-2 의 `GitHubRepoIds`, `GitHubRepo`, `GitHubEnvironment` 로 정해져요. 다른 저장소는 이 서버에 배포할 수 없어요.

### C-2. 자동으로 무엇이 일어나나

1. 개발자가 안정 버전 태그(`vX.Y.Z`)를 만들고, 릴리스 작업(`release`)이 끝나요.
2. **deploy-aws** 작업이 자동으로 시작해요. `doda-aws` 환경에 승인자가 있으면 승인을 기다리고 (없으면 바로 배포돼요)
3. 서버에서 `dahae update v1.6.1` 이 실행됩니다 (승인자를 넣었다면 승인자가 **Review deployments > doda-aws > Approve and deploy** 를 누른 뒤). 업데이트는 최대 6시간까지 기다려요.
4. 서버는 백업을 먼저 하고, 새 버전을 받고, 응답을 확인합니다. 새 버전이 응답하지 않으면 **이전 버전으로 자동으로 되돌립니다.**
5. 마지막으로 `https://<도메인>/health` 와 서버가 보고하는 버전을 확인합니다. 모두 통과하면 작업이 초록색이 됩니다.

> **베타 버전(`v1.5.0-beta.1` 같은 태그)은 자동 배포되지 않아요.** 고객 서버에는 안정 버전만 갑니다.
>
> **승인 방식 차이:** 콘솔에서 사장님이 직접 누르는 업데이트는 기존처럼 사장님의 인증 앱 코드로 승인해요. GitHub 에서 실행되는 배포는 승인자를 넣었다면 **GitHub 승인자의 승인**으로, 아니면 개발자의 실행으로 대신해요.
>
> **릴리스와 배포:** 새 버전은 `EricWijesinghe/Dahae` 에서 릴리스돼요 (서명된 서버 이미지와 설치 스크립트가 공개돼 있어요). 릴리스 작업이 끝나면 `deploy-aws` 가 자동으로 시작해요. 특정 버전 배포와 되돌리기는 C-3 을 봐요.
>
> **자동 업데이트 주의:** 서버에는 기본으로 **매주 월요일 04:00 자동 업데이트** 가 켜져 있어요 (안정 채널, 새 버전을 본 뒤 24시간 대기). GitHub 배포 없이도 서버가 스스로 올라갈 수 있어요. GitHub 배포로만 올리고 싶으면 서버에서 `sudo dahae update-schedule off` 를 실행하세요 (되돌리려면 `sudo dahae update-schedule mon@04:00`).

### C-3. 수동으로 배포하거나 되돌리기

- **특정 버전 배포:** GitHub 저장소의 **Actions** 탭 > 왼쪽 **deploy-aws** > **Run workflow** > `version` 칸에 `v1.6.1` 입력 > **Run workflow**. 승인자를 넣었다면 승인 뒤 실행돼요.
- **이전 버전으로 되돌리기:** 같은 화면에서 **rollback** 을 체크하고 **Run workflow**. 서버에서 `dahae rollback` 이 실행돼요. 마지막 업데이트 바로 전 버전으로 돌아가요.
- **콘솔에서 되돌리기 (GitHub 없이):** 콘솔 **시스템 상태 > 서버 업데이트** 카드의 **복구 > 이전 버전으로 되돌리기**. 사장님 코드와 서버 승인 코드를 차례로 넣어요. 승인 창에 **되돌아갈 버전**(예: `v1.6.1`)이 나와요. 먼저 백업하고, 관리 도구도 그 버전으로 맞춰요.
- **되돌린 뒤 자동 업데이트:** 되돌린 버전을 지켜요. 채널(또는 고정한 태그)이 아직 되돌리기 전 버전을 가리키면 매주 자동 업데이트가 그 버전을 다시 올리지 않고, **새 버전이 나오면** 다시 자동으로 업데이트해요. 직접 업데이트(GitHub 배포, 콘솔 업데이트, `dahae update`)를 하면 이 지킴은 끝나요. 단, v1.6.x 이하로 되돌리면 그 버전의 관리 도구는 이 지킴을 몰라요: 콘솔에서 자동 업데이트를 꺼 두세요.
- 되돌릴 이전 버전이 없으면 (설치 뒤 아직 업데이트한 적이 없으면) 콘솔이 그렇게 알려 주고 아무것도 바꾸지 않아요.
- 되돌리기는 정산이 진행 중이면 멈춰요. 그때는 콘솔 **정산** 화면에서 상태를 먼저 정리하세요 (콘솔은 그 이유를 그대로 보여 줘요).

### C-4. 실패했을 때

- 작업이 빨간색이면 **Actions > 그 실행 > server output** 을 엽니다. 서버가 보낸 출력이 그대로 있어요.
- 서버가 이전 버전으로 되돌렸다면 출력에 "이전 버전으로 되돌렸어요" 라고 나와요. 서비스는 이전 버전에서 계속 돌아가요.
- 서버 자체가 응답하지 않으면 B-5 로 들어가 `sudo dahae logs` 를 확인하세요.

### C-5. 관리 도구(작업 실행기·dahae 명령) 새로 받기

v1.6.0 보다 새 버전부터는 `dahae update <태그>` (GitHub 배포, 콘솔 업데이트, 자동 업데이트 모두)가 앱 이미지와 함께 **그 버전의 관리 도구**
(`dahae` 명령, 서버 작업 실행기, systemd 단위, 예약표, compose 설정)도 바꿔요. 배포 저장소(`EricWijesinghe/Dahae-install`)에서 그 버전의 `install.sh` 와 `agent.sh` 를 받아,
**바로 그 태그에서** 릴리스가 서명한 `SHA256SUMS` (서명은 서버 이미지와 같은 방법으로 확인해요)와 지문이 같을 때만 쓰고, 하나라도 다르면 아무것도 바꾸지 않고 멈춰요.
채널(stable·beta)은 그 채널이 지금 가리키는 태그로 바꿔 받고, 그 태그가 이 서버에 올렸던 버전보다 오래됐으면 멈춰요 (예전 서명 파일을 다시 내밀어도 쓰지 않아요).
`.env` 와 데이터는 그대로예요. `dahae rollback` 도 관리 도구와 작업 실행기를 되돌린 버전의 것으로 맞춰요 (보관해 둔 파일을 써서 다시 내려받지 않아요).
관리 도구 기록은 앱 이미지가 실제로 바뀔 때만 한 칸 밀려요: 새 버전이 없는 주의 자동 업데이트는 되돌리기용 기록을 바꾸지 않아요.

**새 버전은 정상인데 관리 도구만 바꾸지 못했다면** 콘솔 작업에 "관리 도구만 바꾸지 못했어요: 설치 줄을 한 번 다시 실행해 주세요" 가 나와요.
서비스는 새 버전으로 잘 돌아가요. 아래처럼 설치 줄을 한 번 다시 실행하면 맞춰져요. 되돌리기도 같아요.

**v1.6.0 이하로 만든 서버는 한 번만** 설치 줄을 다시 실행해야 해요 (그 버전의 `dahae update` 는 아직 앱 이미지만 바꿔요):
AWS 콘솔 → **Systems Manager > Run Command > AWS-RunShellScript** → 대상 인스턴스를 고르고, 명령에 설치 줄을 그대로 넣어 실행해요
(같은 도메인, 데이터는 그대로예요). B-5 의 웹 SSH 로 실행해도 돼요.
도메인을 옮긴 뒤에는 `MALKKEUM_DOMAIN` 을 빼고 실행하세요 (.env 의 도메인을 써요). 넣으면 그 도메인으로 다시 옮겨 가요.

---

## D. 백업, 모니터링, 비용, 삭제

### D-1. 백업 (두 겹)

| 무엇 | 어디에 | 언제 | 확인 방법 |
|---|---|---|---|
| 데이터베이스와 사진 (서버 안) | 서버 디스크 `backups/` (봉인된 파일) | 매일 03:30 (서버 시간) | `sudo dahae status` |
| 외부 백업 | 이 템플릿이 만든 **S3 저장소** (자동, 키 입력 없음). 콘솔 **설정 > 외부 백업** 에서 다른 S3 호환 저장소의 키를 넣으면 그쪽을 써요 | 매일, 업데이트·되돌리기 전에도 (사진 파일도) | 콘솔 카드: **AWS S3 (자동)** |
| 디스크 스냅샷 | AWS **EBS 스냅샷** (이 템플릿이 매일 만들고 7개를 보관) | 매일 03:00 한국 시간 | EC2 > 스냅샷 |
| 비밀 키 사본 | 콘솔 **시스템 상태 > 서버 업데이트 > 복구 > 비밀 키 사본 내려받기** (또는 서버에서 `sudo dahae keys-backup`) | 한 번, 바뀔 때마다 | 서버 밖(비밀번호 관리 앱 또는 USB)에 보관. 내려받은 뒤 콘솔에서 **서버 밖에 보관했어요** 를 눌러야 보관 확인이 기록돼요 |

- **비밀 키 사본은 꼭 하세요.** 외부 백업은 비밀 키로 잠겨 있어서, 키 사본이 없으면 서버를 잃었을 때 풀 수 없어요.
- **비밀 키 사본에 든 것:** 외부 백업을 푸는 두 키(`DATA_KEY`, `JWT_SECRET`)만 들어 있어요. DB 비밀번호 같은 다른 값은 없어요.
- **자동 외부 백업 (S3):** 서버 역할은 이 저장소에 **새 파일을 만들고**, 읽고, 목록을 볼 수만 있어요. **지우거나 덮어쓸 수 없어요**: 올릴 때마다 "같은 이름이 없을 때만" 조건을 붙여야 해서, 서버를 빼앗겨도 이미 있는 사본을 지우거나 바꾸지 못해요 (새 파일은 더할 수 있어요). 버전 관리가 켜져 있고, 저장소가 암호화하고(SSE-S3), 공개 접근은 막혀 있어요. 오래된 백업은 저장소 규칙이 지워요: 35일 뒤 만료, 만료된 판은 35일 더 보관. 백업이 멈추면 마지막 사본도 약 70일 뒤에 없어지니 `sudo dahae status` 의 "외부 백업 정상" 을 가끔 보세요. 스택을 지워도 저장소는 남아요.
- **이 버전보다 먼저 만든 스택에는 S3 저장소가 없어요.** `launch.sh` 는 있는 스택을 바꾸지 않아요. 그런 서버는 예전처럼 콘솔 **설정 > 외부 백업** 에 다른 S3 호환 저장소의 키를 넣어 연결하세요 (꼭 하세요). 템플릿을 새로 올려 저장소를 더하는 일은 개발사와 상의하세요 (D-4 의 스택 업데이트 주의).
- **서버를 잃고 새로 만들 때:** 예전 스택의 출력 **BackupBucket** 이름을 새 스택에 넘기면 새 서버가 그 저장소를 읽고 이어서 써요. CloudShell 한 줄이면 끝에 `--backup-bucket <BackupBucket 이름>` 을 붙이고, 손으로 만들면 B-2 의 **ExistingBackupBucket** 칸에 넣어요 (같은 리전). 설치가 끝나면 서버에서 `sudo dahae keys-restore <비밀 키 사본>` → `sudo dahae restore-offsite` 순서로 되돌려요.
- 스냅샷만으로는 충분하지 않아요. 서버 밖에 복사본을 하나 더 두세요 (외부 백업).

### D-2. 모니터링

- **가동 확인 (heartbeat):** 서버에서 `sudo dahae heartbeat <주소>` 로 외부 가동 확인 서비스(예: healthchecks.io)에 연결하세요. 서버가 멈추면 외부 서비스가 알려줘요.
- **하드웨어 장애 자동 복구와 알림 메일:** 템플릿이 AWS 경보를 만들어 두었어요. 서버 하드웨어 점검이 실패하면 AWS 가 같은 서버를 다른 장비에서 자동으로 되살리고, Email 로 알림 메일을 보내요 (정상으로 돌아오면 한 번 더). B-2 8번의 **Confirm subscription** 을 눌러야 메일이 와요.
- **디스크 공간 경보는 없어요:** 서버에 CloudWatch 에이전트가 없어 AWS 가 디스크 사용량을 모르기 때문이에요. 디스크는 `sudo dahae status` 와 콘솔 시스템 상태에서 보세요.
- **운영 명령:** 로그와 상태 확인 명령은 [07번 안내서](07-운영-명령어.md) 에 있어요.

### D-3. 비용

- 비용은 서버 종류, 디스크 크기, 데이터 전송량, 스냅샷 양에 따라 달라요.
- 정확한 금액은 AWS 요금 계산기(https://calculator.aws/)에서 **서울(ap-northeast-2)** 과 위 설정으로 직접 계산하세요. 이 안내서에는 가격을 적지 않았어요.
- 고정 IP(Elastic IP)는 서버에 연결되어 있을 때는 따로 비용이 없고, 연결이 끊기면 요금이 붙어요.
- **S3 백업:** 저장 용량은 대략 (DB 백업 + 사진 전체) × 약 70일분이에요. 매일, 그리고 업데이트·되돌리기 전마다 사진 전체를 한 번씩 올리고, 각 사본이 35일 + 35일 남아요. 사진이 많으면 요금 계산기에 S3 Standard 용량도 넣어 보세요.

### D-4. 안전하게 삭제하기

서버를 없애기 전에 **반드시** 다음 순서로 하세요. 삭제하면 서비스가 멈춰요.

1. 마지막 백업: 서버에서 `sudo dahae backup` 과 외부 백업 확인.
2. EC2 > 인스턴스 > 해당 서버 > **작업 > 인스턴스 설정 > 종료 방지 변경** 에서 종료 방지를 **끕니다** (켜져 있으면 스택 삭제가 실패해요).
3. CloudFormation > 스택 > `dahae-…` 선택 > **삭제**.
4. 삭제 후에도 남는 것:
   - **디스크(볼륨):** 서버를 종료해도 디스크는 남아요 (데이터 보호). 더 필요 없다고 확인한 뒤 EC2 > 볼륨에서 직접 지우세요.
   - **고정 IP:** EC2 > 탄력적 IP 에서 확인하고, 필요 없으면 해제하세요 (담당 계정으로 안 되면 DODA 의 관리자 계정으로).
   - **OIDC 공급자:** 계정 전체에서 쓰이므로 스택이 지우지 않아요.
   - **백업 S3 저장소 (`dahae-…-backupbucket-…`):** 데이터 보호를 위해 남아요. 필요 없다고 확인한 뒤 S3 콘솔에서 비우고 지우세요 (DODA 관리자 계정).
5. GitHub 환경 `doda-aws` 의 변수를 지우거나 환경을 삭제하세요.
6. 담당 IAM 사용자는 더 쓸 일이 없으면 **IAM > 사용자 > 삭제** 로 지우세요. 다해 서버가 하나도 남지 않으면 정책 `dahae-deployer-policy` 와 `dahae-boundary` 도 **IAM > 정책** 에서 지워요 (DODA 관리자 계정).

> **스택 업데이트 주의:** CloudFormation 에서 스택을 **업데이트** 할 때 변경 세트에 `ServerInstance` 가 `Replacement: True` 로 나오면 **취소**하세요. 서버가 새로 만들어지는 변경이에요. 새 서버로 옮기려면 개발사와 먼저 상의하세요.

---

## English

This guide sets up one Dahae server on AWS (Seoul) and connects the developer's GitHub Actions (github.com/EricWijesinghe/Dahae), so that a stable release is installed on the server with `dahae update`, optionally after a DODA reviewer approves. A non-specialist can finish the server part from the AWS console.

0. **Fastest path (AWS CloudShell, the `>_` icon in the console, region Seoul):** DODA's admin runs `curl -fsSL https://raw.githubusercontent.com/EricWijesinghe/Dahae/master/deploy/aws/launch.sh | bash -s -- --admin` once (creates both policies), then creates the user below with `dahae-deployer-policy` and `AWSCloudShellFullAccess` attached. The developer, signed in as that user, runs `curl -fsSL https://raw.githubusercontent.com/EricWijesinghe/Dahae/master/deploy/aws/launch.sh | bash -s -- --domain example.co.kr --email owner@example.co.kr`: it checks the latest release's install.sh against the release SHA256SUMS, pins it into the template, finds the default VPC's public subnet, creates stack `dahae-main` and prints the outputs (options: `--domain auto` for a temporary `<ip>.sslip.io` address until the real domain is moved in from the console, `--hosted-zone-id`, `--instance-type`, `--dry-run`). It never touches an existing stack; new versions go through C. Steps 1–2 below are the manual equivalent.
1. **Deployer account (DODA, for the developer):** first create the customer-managed policy `dahae-boundary` (exact name) from `deploy/aws/dahae-boundary-policy.json` and attach it to nobody; it is the permissions boundary of the template's roles. Then create an IAM user with console access for the developer and attach `deploy/aws/deployer-iam-policy.json` (as a second customer-managed policy) and the AWS managed policy `AWSCloudShellFullAccess`. Assign its MFA on a call: the developer scans the QR code with the developer's own phone and reads back the two codes. Create no access keys. Send the sign-in URL and user name through a separate channel from the password.
2. **Template (developer):** the simplest path is `launch.sh` (step 0). To upload a template by hand, download `dahae-aws.yaml` from the latest release (Releases → Latest → Assets); the release builds it from `master`'s template with that release's install.sh URL and SHA-256 filled in. Optional fallback, building it locally: on a `master` checkout, download `install.sh`, `SHA256SUMS` and `SHA256SUMS.bundle` of v1.6.1 (`gh release download v1.6.1 -R EricWijesinghe/Dahae -p install.sh -p SHA256SUMS -p SHA256SUMS.bundle -D rel-v1.6.1`), check them (`scripts/release/verify-blob.sh rel-v1.6.1/SHA256SUMS EricWijesinghe/Dahae` when cosign is installed, then `sha256sum -c --ignore-missing SHA256SUMS`), and, with `dist=https://raw.githubusercontent.com/EricWijesinghe/Dahae-install` (the public delivery repo), run `bash scripts/release/build-aws-template.sh rel-v1.6.1/install.sh "$dist/v1.6.1/install.sh" out`. It writes `out/dahae-aws.yaml` and prints the install.sh SHA-256, which must match the `install.sh` line in `SHA256SUMS`.
3. **Launch (developer, signed in as the deployer user):** in CloudFormation (region ap-northeast-2), create a stack from that `dahae-aws.yaml`. Keep `GitHubRepo` (`EricWijesinghe/Dahae`) and `GitHubRepoIds` (`EricWijesinghe@48539716/Dahae@1452913067`) at their defaults. The stack name must start with `dahae-`. Fill Domain, Email, and optionally HostedZoneId (Route 53). Leave SshCidr empty. Acknowledge IAM and create. Wait for CREATE_COMPLETE, then about 20 minutes for the installer.
4. **DNS (developer, Cloudflare):** add an A record for the domain pointing to the `PublicIp` output, DNS only (grey cloud). For a bare domain such as `example.co.kr`, add two: `@` and `www`. With Route 53 (HostedZoneId) the stack writes the `Domain` record only.
5. **Setup code without SSH:** EC2 > Instances > Connect > Session Manager > Connect, then `sudo dahae setup-code` (valid 2 hours; `--renew` for a new code). DODA's owner, on a call with the developer, opens `https://<domain>/admin`, enters the code, creates the owner account and enables two-step sign-in on DODA's phone (the owner code guards the payment cards, so the developer never creates this account).
6. **CD:** the developer's repository `EricWijesinghe/Dahae` holds the GitHub environment `doda-aws` (deployment branches limited to `master`; required reviewers from DODA are optional, and without them a run deploys as soon as the developer starts it) and four environment variables from the stack outputs: `AWS_REGION`, `AWS_DEPLOY_ROLE_ARN`, `AWS_INSTANCE_ID`, `DAHAE_DOMAIN`. The AWS role trusts that repository (by its immutable owner and repository ids) and environment, so no other repository can deploy here. The workflow `deploy-aws.yml` runs there. Each stable tag (vX.Y.Z) runs `dahae update <tag>` through SSM and checks health and version (after approval, when reviewers are set). Manual deploy and rollback are in the Actions tab (deploy-aws, Run workflow). Beta tags are never deployed.
7. **Note:** the server's weekly auto-update is on by default. To make the GitHub deploy the only path, run `sudo dahae update-schedule off`.
8. **Backups:** the template makes a private S3 bucket (block public access, versioning, SSE-S3, lifecycle: 35 days, then 35 more for old versions; kept when the stack is deleted) and the server's role may only create new objects (every PUT must carry `If-None-Match: *`), get and list: it can never delete or overwrite a copy. Backups stop for good about 70 days after uploads stop, so check `dahae status`. Stacks made before this release have no bucket (launch.sh never updates a stack): connect the console backup card with keys for another S3-compatible store. A replacement stack takes the old stack's `BackupBucket` output as `ExistingBackupBucket` (`launch.sh --backup-bucket NAME`), then `dahae keys-restore` and `dahae restore-offsite`. S3 storage is roughly (DB + all photos) x about 70 days of copies. Offsite backups and photo backups go there with no keys to type (console backup card: "AWS S3 (automatic)"); keys saved on that card for another S3-compatible store win. Keep the keys backup outside the server: console System status > Server update > Recovery > download the secret key copy (owner code, one download; recorded as kept only after the owner presses "Kept off the server"), or `sudo dahae keys-backup`. The copy holds only DATA_KEY and JWT_SECRET. EBS snapshots run daily at 03:00 Seoul time, keep 7.
8a. **Alarm e-mails:** the system-check alarm (auto-recover) also mails the stack's Email through SNS, and again when it is OK. AWS first sends "Subscription Confirmation": click **Confirm subscription** once. No disk-space alarm: the server has no CloudWatch agent.
8b. **Host tools and rollback:** after v1.6.0, `dahae update <tag>` also replaces the host tools (CLI, agent, units, cron, compose) with that release's install.sh from the delivery repo, and agent.sh, only when they match the SHA256SUMS that the release workflow signed at exactly that tag (a channel is read as the tag it points at and refused if older than the newest version this server ran); otherwise nothing changes. The rollback copy of the tools moves only when the images really change. If only the tools could not be replaced, the job says so (exit 5): re-run the install line once. After a rollback, the weekly auto-update holds the rolled-back version until a newer release appears (any manual update ends the hold); a rollback with no previous version says so (exit 6). `dahae rollback` (also in the console: Recovery > Roll back, owner code plus server approval code) puts the previous version's tools back too, and still stops while a Toss payout is in doubt. Servers made with v1.6.0 or older re-run the install line once.
8c. **Policies and Windows:** accounts that ran `--admin` before run it again once (existing policies get the new default version: the template now adds the bucket and the alarm topic). `launch.sh` also runs from Git Bash on Windows with the AWS CLI for Windows.
9. **Costs:** use the AWS pricing calculator (https://calculator.aws/) with Seoul pricing. This guide lists no prices.
10. **Deleting:** back up first, turn off termination protection, then delete the stack. The disk and the Elastic IP stay until you delete them yourself. The GitHub OIDC provider is account-wide and stays. With no Dahae server left, DODA's admin deletes the deployer user and both policies.

[한국어로 돌아가기](#10-aws-에-다해-서버-올리고-자동-배포하기)
