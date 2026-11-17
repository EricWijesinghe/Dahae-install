# 10. AWS 에 다해 서버 올리고 자동 배포하기

[English](#english) · 한국어

이 안내서는 **AWS 를 쓰는 회사(DODA)** 가 다해 서버를 AWS 에 올리고, GitHub 에서 새 버전을 승인하면 자동으로 서버에 반영되도록 설정하는 전 과정입니다. 클라우드 전문가가 아니어도 **화면의 버튼을 누르는 순서대로** 따라 하면 됩니다. 명령을 직접 입력하는 곳은 서버 안의 `dahae setup-code` 한 줄뿐이고, 그것도 AWS 웹 화면에서 엽니다.

> 이 안내서가 가리키는 파일: [deploy/aws/dahae.cfn.yaml](../../deploy/aws/dahae.cfn.yaml) (서버 한 대를 만드는 AWS 템플릿), [deploy/aws/deployer-iam-policy.json](../../deploy/aws/deployer-iam-policy.json) (담당 계정의 권한), [.github/workflows/deploy-aws.yml](../../.github/workflows/deploy-aws.yml) (자동 배포). 템플릿 파일은 각 릴리스의 Assets 에 `dahae-aws.yaml` 이름으로 올라가요.

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
| IAM 사용자 1명 (담당 계정) | DODA 가 만들고 개발사에 전달 | 아래 A 단계 |
| 도메인 (예: `app.example.co.kr`) | DODA | 고객이 들어갈 주소 |
| 도메인 DNS 관리 권한 또는 Route 53 호스팅 영역 | DODA | 있으면 자동 연결, 없으면 A 레코드 한 줄 |
| GitHub 저장소 접근 권한 | 개발사 (EricWijesinghe/Dahae) | 승인자(reviewer)도 이 저장소 접근이 있어야 해요 |
| 인증 앱 (Google Authenticator 등) | 사장님 | 사장님 계정 로그인과 승인 코드에 써요 |
| 비밀번호 관리 앱 | DODA | 계정 정보는 채팅에 쓰지 마세요 |

**보안 원칙:** 비밀번호와 MFA 설정 키는 채팅, 메일 본문, 문서에 적지 않습니다. 비밀번호 관리 앱에 저장하고, 필요하면 전화 등 다른 경로로 전달하세요. AWS 액세스 키(장기 키)는 만들지 않습니다. GitHub 배포는 키 없이 임시 권한으로 돌아갑니다.

---

## A. 담당 계정 만들기 (DODA 가 할 일)

AWS 콘솔(https://aws.amazon.com/console/)에 회사 계정으로 로그인한 뒤 진행하세요.

1. 오른쪽 위 지역을 **아시아 태평양(서울) ap-northeast-2** 로 바꿉니다.
2. 검색창에 **IAM** 을 입력해 엽니다.
3. 왼쪽 **사용자 > 사용자 생성** 을 누릅니다.
4. 사용자 이름: `dahae-deployer` (또는 원하는 이름). **AWS Management Console 에 대한 사용자 액세스 권한 제공** 을 체크합니다.
5. 암호는 **사용자 지정 암호** 를 고르고 강한 암호를 만듭니다. 비밀번호 관리 앱에 저장하세요. "다음 로그인 시 암호 재설정 필요" 는 체크를 해제해도 되고 켜도 됩니다.
6. **권한 설정** 에서 **직접 정책 연결** 을 고르고 **정책 생성** 을 누릅니다 (새 탭이 열려요).
7. **JSON** 탭을 누르고, 개발사가 보내 준 `deployer-iam-policy.json` 내용을 전부 붙여 넣습니다. **다음**, 정책 이름: `dahae-deployer-policy`, **정책 생성**.
8. 이전 탭으로 돌아와 목록을 새로 고침하고 `dahae-deployer-policy` 를 체크한 뒤 **다음 > 사용자 생성**.
9. 사용자 목록에서 방금 만든 사용자를 눌러 **보안 자격 증명 > 다단계 인증(MFA) 할당** 을 누르고 **인증 앱** 을 고릅니다. QR 코드를 인증 앱으로 읽고, 코드 두 개를 입력합니다.
   - MFA 를 켜기 전에는 이 계정으로 서버 관련 작업이 거부돼요. 정책이 MFA 없이는 막도록 되어 있어요. 이건 의도된 동작입니다.
   - 액세스 키는 **만들지 마세요.**
10. 담당자에게 전달할 것: 계정 로그인 주소 (`https://<12자리 계정 번호>.signin.aws.amazon.com/console`), 사용자 이름, 비밀번호 (별도 경로), MFA 는 담당자가 직접 설정합니다.

**English:** Create one IAM user for the deployer, enable console access, paste the policy JSON as a customer-managed policy, attach it, and turn on an authenticator-app MFA. Do not create access keys. Send the sign-in URL and user name through a channel other than chat, and send the password separately.

---

## B. 서버 만들기 (CloudFormation)

담당자가 위 계정으로 로그인하고 MFA 코드를 넣습니다. 지역이 **서울(ap-northeast-2)** 인지 다시 확인하세요.

### B-1. 템플릿 파일 준비

개발사가 보내 준 **`dahae-aws.yaml`** 파일을 컴퓨터에 저장합니다. (이 파일은 다해의 특정 릴리스에만 맞춰져 있어요. 파일 이름을 바꾸지 마세요.)

### B-2. 스택 만들기

1. 검색창에 **CloudFormation** 을 입력해 엽니다.
2. **스택 > 스택 생성 > 새 리소스 사용 (표준)** 을 누릅니다.
3. **템플릿 준비 > 템플릿 파일 업로드 > 파일 선택** 에서 `dahae-aws.yaml` 을 고르고 **다음**.
4. **스택 이름: 반드시 `dahae-` 로 시작** (예: `dahae-doda`). 권한 정책이 `dahae-*` 이름만 허용하므로 다른 이름이면 만들 수 없어요.
5. 매개변수를 채웁니다.

| 항목 | 넣을 값 |
|---|---|
| Domain | 고객이 쓸 주소 (예: `app.example.co.kr`). `https://` 없이 |
| Email | 인증서 안내를 받을 이메일 |
| HostedZoneId | Route 53 에 도메인을 두었다면 호스팅 영역 ID (Z 로 시작). 아니면 **비워 두기** |
| InstanceType | `t3.medium` (x86, 기본값). 처음에는 기본값을 쓰세요. `t4g` 는 ARM 이라 바꾸면 서버가 새로 만들어져요 |
| DiskSizeGiB | `50` (기본값). 사진이 많으면 늘리세요 |
| VpcId | 기본 VPC (default) 를 고르면 됩니다 |
| SubnetId | 그 VPC 의 **인터넷으로 나갈 수 있는 서브넷** 하나. 모르면 기본 서브넷 중 하나 |
| SshCidr | **비워 두기** (SSH 를 열지 않아요. 접속은 AWS 웹 화면으로 합니다) |
| GitHubRepo | `EricWijesinghe/Dahae` (기본값 그대로) |
| GitHubEnvironment | `doda-aws` (기본값 그대로, C 단계와 같은 이름이어야 해요) |
| CreateGitHubOidcProvider | 이 계정에 GitHub 연결이 처음이면 `true`. 이미 있다면 `false` |
| AmdAmiId, ArmAmiId | 바꾸지 마세요 (기본값이 Ubuntu 24.04 를 찾아요) |

6. **다음 > 다음**. 맨 아래 **AWS CloudFormation 이 IAM 리소스를 생성할 수 있음을 승인합니다** 를 체크하고 **제출**.
7. 상태가 `CREATE_IN_PROGRESS` → `CREATE_COMPLETE` 로 바뀔 때까지 기다립니다 (약 3–5분).

> 서버가 `CREATE_COMPLETE` 가 되어도 **설치는 아직 진행 중**이에요. 서버 안에서 설치가 약 15–20분 걸립니다.

### B-3. 도메인 연결

- **Route 53 을 쓴 경우 (HostedZoneId 입력):** 자동으로 끝났어요. 할 일 없음.
- **도메인 등록 업체(가비아, 후이즈, Cloudflare 등)를 쓴 경우:** 스택의 **출력(Outputs)** 탭에서 `PublicIp` 값을 복사합니다. DNS 관리 화면에서 **A 레코드** 를 추가합니다.
  - 호스트(이름): 도메인 앞부분 (예: `app`)
  - 형식: `A`
  - 값: `PublicIp` 에 적힌 IP
  - TTL: 300 (또는 기본값)
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

1. 브라우저에서 `https://<도메인>/admin` 을 엽니다.
2. 설치 코드를 입력하고 **사장님 계정**(이메일, 비밀번호)을 만듭니다.
3. 인증 앱에 2단계 인증을 등록합니다. 복구 코드는 종이에 적어 서버 밖에 보관하세요.
4. 설정 화면에서 회사 정보, 결제, 문자 연결을 순서대로 진행합니다 ([02번 안내서](02-설정-마법사.md)).

---

## C. 자동 배포 연결하기 (GitHub)

새 버전은 **GitHub 에서 승인한 뒤에만** 서버에 들어갑니다. 승인은 GitHub 의 승인자(reviewer)가 합니다.

### C-1. GitHub 환경(Environment) 만들기 (개발사 저장소)

이 단계는 저장소 관리자(개발사 담당)가 GitHub 웹 화면에서 합니다.

1. https://github.com/EricWijesinghe/Dahae 를 엽니다 (접근 권한이 있는 계정으로).
2. **Settings > Environments > New environment** 를 누르고 이름을 `doda-aws` 로 입력 (B-2 의 GitHubEnvironment 와 같아야 해요).
3. **Required reviewers** 를 켜고 승인할 사람(DODA 담당자 또는 개발사 담당자)을 추가합니다. 승인자는 이 저장소에 접근 권한이 있어야 해요.
   - **Deployment branches** 를 **Selected branches** 로 바꾸고 `master` 하나만 추가합니다. 다른 브랜치에서는 이 환경을 쓸 수 없어요. (승인자를 빼면 승인 없이 배포가 돌아요. 꼭 넣으세요.)
4. **Environment variables** 에 아래 네 개를 **Add variable** 로 넣습니다. 값은 B-2 스택의 **출력(Outputs)** 탭에 있어요.

| 변수 이름 (Name) | 출력 탭의 값 | 예시 |
|---|---|---|
| `AWS_REGION` | `AwsRegion` | `ap-northeast-2` |
| `AWS_DEPLOY_ROLE_ARN` | `DeployRoleArn` | `arn:aws:iam::123456789012:role/dahae-…` |
| `AWS_INSTANCE_ID` | `InstanceId` | `i-0123456789abcdef0` |
| `DAHAE_DOMAIN` | `DahaeDomain` | `app.example.co.kr` |

- 이 값들은 **Secrets 가 아니라 Variables** 에 넣습니다. 비밀번호나 키가 아니에요.
- 변수 이름은 대소문자까지 똑같이 쓰세요.

### C-2. 자동으로 무엇이 일어나나

1. 개발사가 안정 버전 태그(예: `v1.5.0`)를 만들면 릴리스 작업이 끝나요.
2. **deploy-aws** 작업이 자동으로 시작하고, `doda-aws` 환경의 승인을 기다립니다 (Actions 탭에 "Waiting" 으로 보여요).
3. 승인자가 **Review deployments > doda-aws > Approve and deploy** 를 누르면 서버에서 `dahae update v1.5.0` 이 실행됩니다.
4. 서버는 백업을 먼저 하고, 새 버전을 받고, 응답을 확인합니다. 새 버전이 응답하지 않으면 **이전 버전으로 자동으로 되돌립니다.**
5. 마지막으로 `https://<도메인>/health` 와 서버가 보고하는 버전을 확인합니다. 모두 통과하면 작업이 초록색이 됩니다.

> **베타 버전(`v1.5.0-beta.1` 같은 태그)은 자동 배포되지 않아요.** 고객 서버에는 안정 버전만 갑니다.
>
> **승인 방식 차이:** 콘솔에서 사장님이 직접 누르는 업데이트는 기존처럼 사장님의 인증 앱 코드로 승인해요. GitHub 에서 실행되는 배포는 **GitHub 승인자의 승인**으로 대신해요.
>
> **자동 업데이트 주의:** 서버에는 기본으로 **매주 월요일 04:00 자동 업데이트** 가 켜져 있어요 (안정 채널, 새 버전을 본 뒤 24시간 대기). GitHub 승인 없이도 서버가 스스로 올라갈 수 있어요. GitHub 승인으로만 올리고 싶으면 서버에서 `sudo dahae update-schedule off` 를 실행하세요 (되돌리려면 `sudo dahae update-schedule mon@04:00`).

### C-3. 수동으로 배포하거나 되돌리기

- **특정 버전 배포:** GitHub 저장소의 **Actions** 탭 > 왼쪽 **deploy-aws** > **Run workflow** > `version` 칸에 `v1.5.0` 입력 > **Run workflow**. 승인자가 승인하면 실행돼요.
- **이전 버전으로 되돌리기:** 같은 화면에서 **rollback** 을 체크하고 **Run workflow**. 서버에서 `dahae rollback` 이 실행돼요. 마지막 업데이트 바로 전 버전으로 돌아가요.
- 되돌리기는 정산이 진행 중이면 멈춰요. 그때는 콘솔 **정산** 화면에서 상태를 먼저 정리하세요.

### C-4. 실패했을 때

- 작업이 빨간색이면 **Actions > 그 실행 > server output** 을 엽니다. 서버가 보낸 출력이 그대로 있어요.
- 서버가 이전 버전으로 되돌렸다면 출력에 "이전 버전으로 되돌렸어요" 라고 나와요. 서비스는 이전 버전에서 계속 돌아가요.
- 서버 자체가 응답하지 않으면 B-5 로 들어가 `sudo dahae logs` 를 확인하세요.

---

## D. 백업, 모니터링, 비용, 삭제

### D-1. 백업 (두 겹)

| 무엇 | 어디에 | 언제 | 확인 방법 |
|---|---|---|---|
| 데이터베이스와 사진 (서버 안) | 서버 디스크 `backups/` (봉인된 파일) | 매일 03:30 (서버 시간) | `sudo dahae status` |
| 외부 백업 (권장) | 콘솔 **설정 > 외부 백업** 에서 연결한 S3 호환 저장소 | 매일 | 콘솔 카드 상태 |
| 디스크 스냅샷 | AWS **EBS 스냅샷** (이 템플릿이 매일 만들고 7개를 보관) | 매일 03:00 한국 시간 | EC2 > 스냅샷 |
| 비밀 키 사본 | 서버에서 `sudo dahae keys-backup` 으로 만든 파일 | 한 번, 바뀔 때마다 | 서버 밖(비밀번호 관리 앱 또는 USB)에 보관 |

- **외부 백업 연결과 비밀 키 사본은 꼭 하세요.** 외부 백업은 비밀 키로 잠겨 있어서, 키 사본이 없으면 서버를 잃었을 때 풀 수 없어요.
- 스냅샷만으로는 충분하지 않아요. 서버 밖에 복사본을 하나 더 두세요 (외부 백업).

### D-2. 모니터링

- **가동 확인 (heartbeat):** 서버에서 `sudo dahae heartbeat <주소>` 로 외부 가동 확인 서비스(예: healthchecks.io)에 연결하세요. 서버가 멈추면 외부 서비스가 알려줘요.
- **하드웨어 장애 자동 복구:** 템플릿이 AWS 경보를 만들어 두었어요. 서버 하드웨어가 고장 나면 AWS 가 같은 서버를 다른 장비에서 자동으로 되살려요. 디스크 데이터는 그대로 남아요.
- **운영 명령:** 로그와 상태 확인 명령은 [07번 안내서](07-운영-명령어.md) 에 있어요.

### D-3. 비용

- 비용은 서버 종류, 디스크 크기, 데이터 전송량, 스냅샷 양에 따라 달라요.
- 정확한 금액은 AWS 요금 계산기(https://calculator.aws/)에서 **서울(ap-northeast-2)** 과 위 설정으로 직접 계산하세요. 이 안내서에는 가격을 적지 않았어요.
- 고정 IP(Elastic IP)는 서버에 연결되어 있을 때는 따로 비용이 없고, 연결이 끊기면 요금이 붙어요.

### D-4. 안전하게 삭제하기

서버를 없애기 전에 **반드시** 다음 순서로 하세요. 삭제하면 서비스가 멈춰요.

1. 마지막 백업: 서버에서 `sudo dahae backup` 과 외부 백업 확인.
2. EC2 > 인스턴스 > 해당 서버 > **작업 > 인스턴스 설정 > 종료 방지 변경** 에서 종료 방지를 **끕니다** (켜져 있으면 스택 삭제가 실패해요).
3. CloudFormation > 스택 > `dahae-…` 선택 > **삭제**.
4. 삭제 후에도 남는 것:
   - **디스크(볼륨):** 서버를 종료해도 디스크는 남아요 (데이터 보호). 더 필요 없다고 확인한 뒤 EC2 > 볼륨에서 직접 지우세요.
   - **고정 IP:** EC2 > 탄력적 IP 에서 확인하고, 필요 없으면 해제하세요.
   - **OIDC 공급자:** 계정 전체에서 쓰이므로 스택이 지우지 않아요.
5. GitHub 환경 `doda-aws` 의 변수를 지우거나 환경을 삭제하세요.
6. 담당 IAM 사용자는 더 쓸 일이 없으면 **IAM > 사용자 > 삭제** 로 지우세요.

> **스택 업데이트 주의:** CloudFormation 에서 스택을 **업데이트** 할 때 변경 세트에 `ServerInstance` 가 `Replacement: True` 로 나오면 **취소**하세요. 서버가 새로 만들어지는 변경이에요. 새 서버로 옮기려면 개발사와 먼저 상의하세요.

---

## English

This guide sets up one Dahae server on AWS (Seoul) and connects GitHub Actions so that a stable release, once approved by a reviewer, is installed on the server with `dahae update`. A non-specialist can finish it from the AWS and GitHub web consoles.

1. **Deployer account (DODA):** create an IAM user with console access, attach `deploy/aws/deployer-iam-policy.json` as a customer-managed policy, enable MFA (authenticator app), and create no access keys. Send the sign-in URL and user name through a separate channel from the password.
2. **Launch:** in CloudFormation (region ap-northeast-2), create a stack from `dahae-aws.yaml`. The stack name must start with `dahae-`. Fill Domain, Email, and optionally HostedZoneId (Route 53). Leave SshCidr empty. Acknowledge IAM and create. Wait for CREATE_COMPLETE, then about 20 minutes for the installer.
3. **DNS:** with Route 53 it is automatic; otherwise add an A record for the domain pointing to the `PublicIp` output.
4. **Setup code without SSH:** EC2 > Instances > Connect > Session Manager > Connect, then `sudo dahae setup-code` (valid 2 hours; `--renew` for a new code). Open `https://<domain>/admin`, enter the code, create the owner account, enable two-step sign-in.
5. **CD:** the repo admin creates the GitHub environment `doda-aws` with required reviewers, deployment branches limited to `master`, and four environment variables from the stack outputs: `AWS_REGION`, `AWS_DEPLOY_ROLE_ARN`, `AWS_INSTANCE_ID`, `DAHAE_DOMAIN`. Each stable tag (vX.Y.Z) waits for approval, then runs `dahae update <tag>` through SSM and checks health and version. Manual deploy and rollback are in the Actions tab (deploy-aws, Run workflow). Beta tags are never deployed.
6. **Note:** the server's weekly auto-update is on by default. To make GitHub approval the only path, run `sudo dahae update-schedule off`.
7. **Backups:** keep the offsite backup and the keys backup (`sudo dahae keys-backup`) outside the server. EBS snapshots run daily at 03:00 Seoul time, keep 7.
8. **Costs:** use the AWS pricing calculator (https://calculator.aws/) with Seoul pricing. This guide lists no prices.
9. **Deleting:** back up first, turn off termination protection, then delete the stack. The disk and the Elastic IP stay until you delete them yourself. The GitHub OIDC provider is account-wide and stays.

[한국어로 돌아가기](#10-aws-에-다해-서버-올리고-자동-배포하기)
