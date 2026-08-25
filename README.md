# Ansible Linux 로깅 기준선 스타터

CentOS/RHEL 계열과 Ubuntu 서버의 로그인 경고, 시간 동기화, auditd,
journald, rsyslog 및 정기 점검을 단계적으로 관리하기 위한 시작점입니다.

## 문서

### 자산 진단 체크 취약점별 조치 방법

- [취약점별 조치 현황과 작성 기준](docs/remediation/README.md)
- [U0308 - Session Timeout 설정](docs/remediation/U0308.md)
- [U0309 - root 계정 Telnet·SSH 접근 제한](docs/remediation/U0309.md)
- [U0510 - OpenSSL 버전 취약성](docs/remediation/U0510.md)

### 공통 운영과 잔여 요구사항

- [결정사항과 구현 현황](docs/decisions-and-status.md)
- [공통 구축 및 운영 절차](docs/operations-runbook.md)
- [로깅 정책 잔여 요구사항 매핑](docs/logging-policy-mapping.md)
- [audit 정책 잔여 요구사항 설계](docs/audit-policy-design.md)
- [로그 보존정책 잔여 요구사항 설계](docs/log-retention-design.md)
- [로그인 정책 잔여 요구사항 설계](docs/login-policy-design.md)

## Ansible 마스터노드 최초 구성

아래 작업은 Ansible을 실행할 Rocky Linux 9.8 마스터노드에서 수행합니다. 별도의
로컬 `ansible-admin` 계정은 필요하지 않으며 저장소와 SSH 개인키를 소유한 현재
실행 계정으로 Ansible을 실행합니다.

### 1. Rocky Linux 9.8 기본 패키지 설치

마스터노드 버전을 확인하고 필요한 패키지를 설치합니다.

```bash
cat /etc/rocky-release
sudo dnf repolist
sudo dnf install -y git python3.12 python3.12-pip openssh-clients
python3.12 --version
```

### 2. 저장소 clone

```bash
git clone https://github.com/taeboss/ansible-logging-starter.git
cd ansible-logging-starter
git switch main
git pull --ff-only
```

이미 clone한 저장소라면 `cd ansible-logging-starter` 이후 `git pull --ff-only`만
실행합니다. 운영 중에는 검토되지 않은 로컬 수정이 섞이지 않았는지
`git status --short`로 확인합니다.

### 3. Python 가상환경과 Ansible 설치

시스템 기본 Python은 변경하지 않습니다. 저장소 내부가 아닌 마스터노드 실행
계정의 `~/.venvs/`에 Python 3.12 가상환경을 만들고 현재 검증 버전인
`ansible-core 2.21.3`과 필요한 Collection을 설치합니다.

```bash
mkdir -p ~/.venvs
python3.12 -m venv ~/.venvs/ansible-core-2.21.3
source ~/.venvs/ansible-core-2.21.3/bin/activate
python -m pip install --upgrade pip
python -m pip install 'ansible-core==2.21.3'
ansible-galaxy collection install -r requirements.yml
```

새 터미널에서는 마스터노드의 가상환경을 활성화한 뒤 저장소로 이동합니다.

```bash
source ~/.venvs/ansible-core-2.21.3/bin/activate
cd ansible-logging-starter
```

설치 결과를 확인합니다.

```bash
ansible --version
ansible-galaxy collection list ansible.posix
```

### 4. Ansible 전용 SSH 키 생성

기존 키가 있는지 먼저 확인합니다. 파일이 이미 있으면 `ssh-keygen`으로
덮어쓰지 말고 기존 키의 관리 주체와 사용 여부를 확인합니다.

```bash
ls -l ~/.ssh/ansible_ed25519 ~/.ssh/ansible_ed25519.pub
```

키가 없을 때만 생성합니다.

```bash
mkdir -p ~/.ssh
chmod 700 ~/.ssh
ssh-keygen -t ed25519 -f ~/.ssh/ansible_ed25519 -C ansible-admin
chmod 600 ~/.ssh/ansible_ed25519
chmod 644 ~/.ssh/ansible_ed25519.pub
ssh-keygen -lf ~/.ssh/ansible_ed25519.pub
```

개인키는 마스터노드에만 보관합니다. 대상 서버에는 bootstrap 플레이북이
공개키만 배포합니다.

## 정책 적용 현황 요약

이 표의 상태는 정책상 가능한 기능이 아니라 **현재 저장소의 소스코드 기준**이다.

- ✅ 적용 가능: 현재 플레이북 또는 Role에 구현되어 있다.
- 🟡 부분 적용: 일부 구현됐지만 추가 정보, 스케줄 또는 외부 시스템이 필요하다.
- ⚪ 미구현: Ansible로 구현할 수 있지만 현재 실행 코드에는 없다.
- 🚫 외부 절차: Ansible만으로 보장할 수 없는 사람·IAM·SIEM 영역이다.

### 자산 진단 체크 취약점별 조치 현황

자산 진단 체크 항목은 취약점 코드와 OS별 플레이북으로 분리한다. 각 설정은
하나의 취약점 플레이북만 변경하며 기존 통합 플레이북에서는 중복 적용하지
않는다. 사전점검 플레이북의 읽기 전용 수집은 중복 적용이 아니므로 유지한다.

| 조치 방법 | 상태 | Red Hat 계열 | Ubuntu | 자동 조치 | 직접 조치 |
|---|---:|---|---|---|---|
| [U0308 - Session Timeout 설정](docs/remediation/U0308.md) | ✅ | `playbooks/controls/U0308-redhat.yml` | `playbooks/controls/U0308-ubuntu.yml` | `/etc/profile`의 비정상 `TMOUT` 지시문을 정리한다. 유효값이 없거나 0·비숫자·300초 초과이면 300초로 설정하고, 기존 1~300초는 유지한다. | 없음. 새 로그인 세션에서 적용 여부 확인 |
| [U0309 - root 계정 Telnet·SSH 접근 제한](docs/remediation/U0309.md) | 🟡 | `playbooks/controls/U0309-redhat.yml` | `playbooks/controls/U0309-ubuntu.yml` | SSH: `PermitRootLogin no`, `PermitEmptyPasswords no`, 구문·유효 설정·서비스 상태 검증 | Telnet: 직접 조치 필요. 플레이북은 TCP/23, `/etc/pam.d/login`, `/etc/securetty` 현황만 확인한다. |
| [U0510 - OpenSSL 버전 취약성](docs/remediation/U0510.md) | 🟡 | `playbooks/controls/U0510-redhat.yml` | `playbooks/controls/U0510-ubuntu.yml` | 설치된 OpenSSL 실행 패키지와 라이브러리를 OS 공식 저장소의 최신 버전으로 업데이트하고 추가 업데이트 여부를 검증한다. | 업무 서비스 영향 확인 후 필요한 서비스 재시작 또는 서버 재부팅 |

취약점별 조치가 추가되어 기존 로깅·로그인 정책의 세부 내용을 완전히 대체하면
아래 잔여 요구사항 표에서 해당 행을 삭제한다. 일부만 대체한 경우에는 아직
구현하지 않은 잔여 범위만 남긴다.

U0308은 `/etc/profile` 끝의 전용 관리 블록에서 새 로그인 셸의 `TMOUT`을
1~300초로 제한한다. 미설정·0·비숫자·300초 초과 값은 300초로 보정하고,
300초를 초과하거나 비정상인 기존 활성 지시문은 관리 블록 밖에서 제거한다.
기존 1~300초 값은 유지한다. 서비스 재시작과 재부팅은 필요하지 않다.

```bash
ansible-playbook playbooks/controls/U0308-redhat.yml \
  --limit REDHAT_TARGET --check --diff

ansible-playbook playbooks/controls/U0308-ubuntu.yml \
  --limit UBUNTU_TARGET --check --diff
```

U0309는 접속 장애 위험을 줄이기 위해 한 대씩 실행한다. Red Hat 계열은
`sshd`를 reload하고 Ubuntu는 `ssh`를 restart한다. 변경 전 SSH 구문이나
서비스 상태가 정상이 아니거나, 변경 후 유효 설정이 기준과 다르면 서비스에
반영하지 않고 중단한다.

먼저 시험 서버 한 대에서 점검 모드로 변경 예정 내용을 확인한다.

```bash
ansible-playbook playbooks/controls/U0309-redhat.yml \
  --limit rocky-server-01 --check --diff

ansible-playbook playbooks/controls/U0309-ubuntu.yml \
  --limit ubuntu-server-01 --check --diff
```

점검 결과를 검토한 후 `--check --diff`를 제거하여 실제 적용한다. Telnet이
TCP/23에서 수신 중인 서버는 플레이 결과에 `LISTENING`으로 표시되며 담당자
협의 후 PAM과 `/etc/securetty`를 별도로 조치한다.

U0510은 전체 OS를 업그레이드하지 않고 현재 설치된 OpenSSL 관련 패키지만
최신화한다. 배포판의 보안 백포트를 인정하므로 upstream 버전 숫자만으로
양호·취약을 판단하지 않고 OS 패키지 전체 버전과 추가 업데이트 여부를 확인한다.
서비스 재시작과 재부팅은 자동 수행하지 않는다.

```bash
ansible-playbook playbooks/controls/U0510-redhat.yml \
  --limit REDHAT_TARGET --check --diff

ansible-playbook playbooks/controls/U0510-ubuntu.yml \
  --limit UBUNTU_TARGET --check --diff
```

### 로깅 정책 잔여 요구사항

`10-99-apply-all-logging.yml`은 아래의 `10-01`~`10-05` 적용 항목 전체를
일괄 실행한다. 표의 점검 플레이북은 설정을 변경하지 않고 증적만 수집한다.

| 정책 항목 | 상태 | 관련 플레이북 | Ansible 실행 성격 | 현재 소스 기준 설명 |
|---|---:|---|---|---|
| 로그인 전 시스템 정보 비노출 및 인가 사용자 경고 | ✅ | `10-01-apply-login-warning.yml` | 최초 1회 | 고정 문구를 `/etc/issue`, `/etc/issue.net`에 배포하고 SSH Banner를 설정한다. 일반·시리얼 콘솔은 `agetty --nohostname`으로 호스트명을 숨기고 Ubuntu/Debian SSH는 `DebianBanner no`로 배포판 버전 정보를 숨긴다. OpenSSH 프로토콜의 최소 식별정보는 남는다. |
| 불필요한 로그인 안내문 제거 | 🟡 | `10-01-apply-login-warning.yml` | 최초 1회 | 사전 인증 Banner는 통제하지만 Ubuntu 동적 MOTD와 기타 프로그램별 안내문 제거는 구현하지 않았다. |
| 로그 및 감사 기록의 관리자 전용 관리 | 🟡 | `10-03-apply-local-logging.yml`, `10-04-apply-audit.yml` | 최초 1회 | `logadmin` 그룹, `/var/log/compliance`, audit 규칙 권한만 구현했다. 관리자 명단, 전체 로그 권한과 sudo 정책이 필요하다. |
| chrony 기반 외부 NTP 동기화 | ✅ | `10-02-apply-ntp.yml` | 최초 1회 | `time.kaist.ac.kr`, `time.nist.gov`을 사용하도록 chrony를 설치·설정한다. Ubuntu/Debian은 apt 캐시를 갱신하고 기존 timesyncd/ntpd를 중지·비활성화·마스킹한다. 적용 전에 DNS와 UDP/123 응답을 확인해야 한다. |
| 월 1회 시간 오차 점검 | 🟡 | `30-monthly-time-check.yml` (점검) | 정기 반복 | 오차 점검 플레이북은 있지만 AWX/AAP·CI·cron 스케줄은 등록되지 않았다. |
| 반기 네트워크 서비스 현황 점검 | 🟡 | `40-semiannual-network-review.yml` (점검) | 정기 반복 | LISTEN·연결 소켓 보고서는 생성하지만 정기 스케줄, 승인 목록 비교와 특이사항 판단은 없다. |
| 관리 서버 원시 로그, 기타 서버 요약 로그 | ⚪ | 미구현 | 최초 1회 | 모든 서버를 단일 정책과 `raw` 변수로 통일했다. `log_profile`에 따른 실제 필터링은 구현되지 않았다. |
| 가장 최근에 로그인한 사용자 | ✅ | `50-account-access-review.yml` (점검) | 정기 반복 | 계정 접근 정기점검에서 `lastlog` 결과를 날짜별 JSON으로 수집한다. |
| 사용자별 로그인 시간 | ✅ | `10-04-apply-audit.yml`, `50-account-access-review.yml` (점검) | 정기 반복 | `lastlog`, `last -F` 결과와 wtmp/lastlog 감사 규칙을 사용한다. |
| 사용자별 로그인·로그아웃 | ✅ | `10-04-apply-audit.yml`, `50-account-access-review.yml` (점검) | 최초 1회 | utmp, wtmp, btmp, lastlog 변경을 auditd가 기록하도록 설정한다. |
| 접속 터미널 ID와 위치 | 🟡 | `10-01-apply-login-warning.yml`, `10-04-apply-audit.yml`, `50-account-access-review.yml` (점검) | 최초 1회 | 세션 파일과 SSH `LogLevel VERBOSE`로 TTY·원격 IP를 확보할 수 있다. 물리 위치는 VPN/NAC/CMDB 연계가 필요하다. |
| 서버 접근 성공 및 실패 | 🟡 | `10-01-apply-login-warning.yml`, `10-04-apply-audit.yml`, `50-account-access-review.yml` (점검) | 최초 1회 | SSH 상세 로그와 로그인 세션·실패 파일은 설정한다. 중앙 상관분석과 서비스별 접근 감사는 없다. |
| 데이터 및 다른 자원 접근 성공·실패 | 🟡 | `10-04-apply-audit.yml` | 최초 1회 | 선택한 경로 변경과 일반 사용자의 `EACCES/EPERM` 파일 접근 실패를 기록한다. 업무 데이터·DB·앱 감사 대상은 미정이다. |
| 시스템·커널·부팅 이벤트 | ✅ | `10-03-apply-local-logging.yml` | 최초 1회 | journald를 영구 저장하고 rsyslog로 전달하도록 설정한다. |
| 주요 시스템 및 네트워크 설정파일 | ✅ | `10-04-apply-audit.yml` | 최초 1회 | 계정, sudo, SSH, PAM, hosts, resolv, sysctl 등 존재하는 경로를 audit watch로 설정한다. |
| cron 실행 및 패키지 업데이트 이력 | 🟡 | `10-04-apply-audit.yml` | 최초 1회 | cron 설정파일 변경은 감사하지만 cron의 실제 실행과 yum/dnf/apt/dpkg 이력 규칙은 없다. |
| 사용자 명령어 실행 이력 | ✅ | `10-04-apply-audit.yml` | 최초 1회 | 로그인 사용자의 `auid`를 기준으로 `execve`를 감사한다. 명령 인자의 민감정보 보호정책은 추가로 필요하다. |
| 로그 1년 이상 보관 | 🟡 | `10-03-apply-local-logging.yml`, `10-05-apply-central-logging.yml` | 최초 1회 | journald에 365일을 설정했지만 4GB 제한으로 조기 삭제될 수 있다. audit/logrotate 및 중앙 저장소 보존정책이 필요하다. |
| 주 1회 무결성·용량 점검 | 🟡 | `20-weekly-check.yml` (점검) | 정기 반복 | 용량 임계치, audit 상태와 규칙 파일 SHA-256을 점검한다. 승인 체크섬 비교와 실행 스케줄은 없다. |
| 지속적인 로그인 실패 알림 | 🚫 | `10-04-apply-audit.yml`, `10-05-apply-central-logging.yml`; SIEM 필요 | 상시 운영(외부) | 감사 이벤트 생성은 가능하지만 실시간 분석과 알림은 SIEM 영역이다. |
| 관리자 권한 파일·서비스 사용 시도 | 🟡 | `10-04-apply-audit.yml` | 최초 1회 | sudoers 변경과 일반 접근 거부는 일부 기록한다. sudo/su/systemctl/SUID 실행 감사 규칙은 미구현이다. |
| 비인가 파일·자원·서비스 사용 시도 | 🟡 | `10-04-apply-audit.yml`, `40-semiannual-network-review.yml` (점검) | 최초 1회 | 파일 접근 실패는 기록한다. 승인 파일·포트·서비스 목록 및 실시간 비교·알림은 없다. |
| 행위별 성공·실패 구분과 유형별 보존 | 🟡 | `10-03-apply-local-logging.yml`, `10-04-apply-audit.yml`, `10-05-apply-central-logging.yml` | 최초 1회 | 접근 실패 errno 규칙은 있다. 전체 이벤트의 성공·실패 규칙과 로그 유형별 보존기간은 미정이다. |

### 로그인 정책 잔여 요구사항

> 현재 저장소에는 통합 `login_policy` Role과 `11-apply-login-policy.yml`이 없다.
> 아래 표에는 자산 진단 체크 취약점별 조치로 아직 완전히 대체되지 않은
> 요구사항만
> 남긴다.

`03-security-preflight.yml`과 `50-account-access-review.yml`은 로그인 정책을
변경하지 않고 현재 설정과 계정 상태만 수집한다.

| 정책 항목 | 상태 | 관련 플레이북 | Ansible 실행 성격 | Ansible 적용 가능 범위 또는 미구현 사유 |
|---|---:|---|---|---|
| 1인 1계정 발급 | 🚫 | `03-security-preflight.yml`, `50-account-access-review.yml` (점검만) | 조건부 반복 | 계정 생성·중복 UID 점검은 자동화할 수 있지만 실제 사용자와 계정의 1:1 관계는 HR/IAM·발급 절차가 보장해야 한다. |
| 패스워드는 사용자 본인이 관리 | 🚫 | 외부 절차 | 상시 운영(외부) | Ansible은 비밀번호 자체가 아닌 복잡도·만료·이력 정책만 관리해야 한다. |
| 2종류 10자 또는 3종류 8자, 권고 12~64자 | ⚪ | `03-security-preflight.yml` (점검만), 적용 플레이북 미구현 | 최초 1회 | `pam_pwquality`로 구현 가능하지만 조건부 OR를 배포판 공통으로 정확히 표현하기 어려워 최소 12자·2종류의 단일 기준을 설계했다. 아직 코드화하지 않았다. |
| 실제 Null 패스워드 계정 금지 | ⚪ | `03-security-preflight.yml`, `50-account-access-review.yml` (점검만), 적용 플레이북 미구현 | 최초 1회 | `/etc/shadow`의 실제 빈 암호 계정 조치는 아직 구현하지 않았다. SSH의 빈 암호 접속 제한은 U0309에서 별도로 관리한다. |
| 사용자 ID와 동일한 패스워드 금지 | ⚪ | 적용 플레이북 미구현 | 최초 1회 | `pam_pwquality usercheck`로 구현 가능하나 구형 OS 지원 확인과 Role 작성이 필요하다. |
| 예측 가능한 패스워드 금지 | ⚪ | 적용 플레이북 미구현 | 최초 1회 | 사전검사, 반복·연속문자 제한은 가능하다. 조직명·연도 등 모든 패턴의 완전한 차단은 별도 사전/IAM이 필요하다. |
| 주기성 패스워드 재사용 금지 | ⚪ | `03-security-preflight.yml` (점검만), 적용 플레이북 미구현 | 최초 1회 | 동일한 최근 비밀번호는 `pam_pwhistory`로 차단 가능하다. 숫자·계절명 변형 패턴은 표준 PAM만으로 완전히 막기 어렵다. |
| 패스워드 힌트 금지 | 🚫 | 외부 절차 | 상시 운영(외부) | 일반 Linux 로컬계정의 표준 기능이 아니며 포털·메일·헬프데스크 절차에서 통제해야 한다. |
| 초기 패스워드 최초 접속 시 변경 | ⚪ | 적용 플레이북 미구현 | 조건부 반복 | 계정 발급 후 `chage -d 0`으로 강제 가능하지만 사용자 목록과 발급 흐름이 정해지지 않았다. |
| 로그인 실패 10회 이하·로그·연결 해제 | ⚪ | `03-security-preflight.yml`, `50-account-access-review.yml` (점검만), 적용 플레이북 미구현 | 최초 1회 | `pam_faillock`, SSH `MaxAuthTries`, audit/PAM 로그로 가능하다. OS별 PAM 방식과 잠금시간이 확정되지 않았다. |
| 비밀번호 6개월 만료, MFA 사용자는 1년 | ⚪ | `03-security-preflight.yml`, `50-account-access-review.yml` (점검만), 적용 플레이북 미구현 | 최초 1회 | `chage -M 180/365`로 가능하다. 사람·서비스 계정 구분과 MFA 대상 원천정보가 필요하다. |
| 직전 2개 비밀번호 재사용 금지 | ⚪ | `03-security-preflight.yml` (점검만), 적용 플레이북 미구현 | 최초 1회 | `pam_pwhistory remember=2`로 가능하지만 OS별 PAM Role이 아직 없다. |
| 일방향 암호화 저장 | ⚪ | 적용·검증 플레이북 미구현 | 최초 1회 | OS별 SHA-512 또는 yescrypt 정책 적용이 가능하다. 기존 해시는 다음 비밀번호 변경 때 전환된다. 아직 코드화하지 않았다. |
| 분실·도난 시 본인 확인 후 재발급 | 🚫 | 외부 절차 | 조건부 반복 | 휴대폰·이메일 본인 확인과 승인은 IAM·헬프데스크 영역이다. 승인 후 임시 해시 적용과 즉시 변경만 자동화할 수 있다. |

### 현재 실행 가능한 범위

현재 실제 설정 적용 진입점은 기능별 로깅 플레이북, 전체 적용 플레이북과
자산 진단 체크 취약점별 플레이북이다.

```bash
ansible-playbook playbooks/10-01-apply-login-warning.yml
ansible-playbook playbooks/10-02-apply-ntp.yml
ansible-playbook playbooks/10-03-apply-local-logging.yml
ansible-playbook playbooks/10-04-apply-audit.yml
ansible-playbook playbooks/10-05-apply-central-logging.yml
ansible-playbook playbooks/10-99-apply-all-logging.yml
ansible-playbook playbooks/controls/U0308-redhat.yml
ansible-playbook playbooks/controls/U0308-ubuntu.yml
ansible-playbook playbooks/controls/U0309-redhat.yml
ansible-playbook playbooks/controls/U0309-ubuntu.yml
ansible-playbook playbooks/controls/U0510-redhat.yml
ansible-playbook playbooks/controls/U0510-ubuntu.yml
```

U0308·U0309·U0510을 제외한 통합 보안 정책은 별도 Role을 구현하기 전까지 실행 명령이
존재하지 않는다. audit 규칙 보완과 로그 보존정책 역시 상세 설계만 추가했으며
아직 실행 Role에는 반영하지 않았다.

## 적용 전 수정할 값

1. 최초 관리계정 구성은 `inventory/bootstrap/hosts.yml`·Vault 일괄 방식과
   IP 직접 지정 방식 중에서 선택합니다. 일괄 방식의 호스트·자격증명 파일은
   Git에서 제외됩니다.
2. `inventory/hosts.yml`의 예시 IP(`192.0.2.x`)를 실제 IP로 교체하고 모든
   대상 서버를 단일 `linux_servers` 그룹에 넣습니다.
3. `inventory/group_vars/all.yml`에서 SSH 개인키 경로, 경고문,
   NTP 서버, 보존 기준을 확정합니다.
4. 중앙 로그/SIEM이 준비되기 전에는 `central_logging_enabled: false`를
   유지합니다. 현재 TLS 템플릿은 암호화만 제공하고 CA 기반 서버 인증을
   수행하지 않습니다.

## 실행 성격 구분

아래 분류는 정책 행위 자체가 아니라 Ansible 적용·점검 작업의 실행 성격이다.

- **최초 1회**: 신규 서버를 Ansible 관리 대상으로 등록하거나 기준 설정을
  처음 적용할 때 실행한다.
- **조건부 반복**: 계정 발급·재발급, 접속 장애 점검처럼 해당 사건이 발생했을
  때만 실행한다.
- **정기 반복**: 로그와 운영 상태의 증적을 남기기 위해 정해진 주기로 실행한다.
- **상시 운영(외부)**: 실시간 분석·알림처럼 Ansible 정기 실행이 아닌
  SIEM·IAM 등 외부 시스템에서 지속적으로 운영한다.

Ansible 플레이북은 같은 상태를 다시 적용해도 불필요한 변경을 만들지 않도록
멱등성을 기준으로 작성한다. `최초 1회` 설정은 적용 후 개별 서버에서 수동으로
변경하지 않는다. 중앙 Ansible 정책 개정이나 서버 재구축 시에는 새 기준을 다시
배포할 수 있지만 이를 평상시 조건부 반복 작업으로 분류하지 않는다.

## 기존 서버 bootstrap 실행

기존 서버를 처음 Ansible 관리 대상으로 전환할 때는 기존 관리자 계정으로
접속하여 `ansible-admin`, 전용 공개키와 비대화형 sudo를 구성합니다. Rocky
마스터노드에도 같은 이름의 로컬 계정을 만들 필요는 없습니다.
먼저 위의 마스터노드 최초 구성을 완료합니다.

bootstrap 진입 방법은 다음 두 가지 중에서 선택합니다.

### 방식 1: hosts.yml과 서버별 Vault로 일괄 실행

서버 IP와 기존 관리자 계정명은 `hosts.yml`에, 서로 다른 SSH·sudo 암호는
서버별 Ansible Vault 파일에 저장합니다. `hosts.yml`과 `host_vars/`는 모두
Git에서 제외됩니다.

```bash
cp inventory/bootstrap/hosts.example.yml inventory/bootstrap/hosts.yml
# hosts.yml에 서버 IP와 bootstrap_initial_user 입력

mkdir -p inventory/bootstrap/host_vars/server-001
ansible-vault create inventory/bootstrap/host_vars/server-001/vault.yml
```

Vault 파일에는 해당 서버의 기존 관리자 자격증명만 입력합니다.

```yaml
ansible_password: "기존 관리자 SSH 암호"
ansible_become_password: "기존 관리자 sudo 암호"
```

모든 서버의 Vault 파일을 같은 Vault 암호로 만들면 다음 명령 한 번으로
서버마다 서로 다른 자격증명을 사용하여 기본 10대씩 처리할 수 있습니다.

```bash
ansible-playbook -i inventory/bootstrap/hosts.yml \
  playbooks/00-bootstrap-access.yml --ask-vault-pass
```

### 방식 2: hosts.yml 없이 IP를 한 대씩 직접 지정

서버 암호를 파일에 보관하지 않을 때 사용합니다. 마지막 쉼표가 있어야 IP를
인라인 인벤토리로 인식합니다. 암호는 명령 인자에 넣지 않고 프롬프트에 직접
입력합니다. 먼저 기존 관리자 계정으로 대상 IP에 일반 SSH 접속을 한 번 완료해
호스트 키를 `known_hosts`에 등록한 뒤 실행합니다.

```bash
ansible-playbook -i '192.0.2.101,' \
  playbooks/00-bootstrap-access.yml \
  -e bootstrap_hosts=all \
  -e bootstrap_initial_user=기존관리자계정 \
  --ask-pass --ask-become-pass
```

이 방식은 서버마다 반복 실행합니다. 성공한 서버는 이후 운영용
`inventory/hosts.yml`에 추가하고, 그때부터 `ansible-admin`과 전용 공개키를
사용합니다.

플레이북은 새 `ansible-admin` 공개키 접속과 `sudo`까지 자동 검증합니다. 별도로
다시 확인하려면 다음 명령을 실행합니다.

```bash
ssh -i ~/.ssh/ansible_ed25519 ansible-admin@192.0.2.101 \
  'id; sudo -n id'
```

첫 번째 `id`는 `ansible-admin`, 두 번째 `sudo -n id`는 `uid=0(root)`가
출력되어야 합니다. 생성된 보고서도 확인합니다.

```bash
find reports/bootstrap -type f -print
```

bootstrap 완료 후 공통 운영 흐름을 실행합니다.

```bash
ansible-inventory --graph
ansible-playbook playbooks/01-connectivity.yml
ansible-playbook playbooks/02-register-server.yml
```

기존 관리자 계정이 이미 SSH 키와 비대화형 sudo를 사용한다면 암호 옵션을
생략할 수 있습니다. 비밀번호·개인키와 Vault 암호는 인벤토리나 Git에 기록하지
않습니다.

bootstrap은 기본 10대 단위로 실행합니다. 현재 배치에 아래 항목을 구성한 뒤
같은 배치에서 새 계정의 공개키 전용 SSH 접속·Python·비대화형 sudo를 즉시
검증합니다. 한 대라도 실패하면 다음 배치를 시작하지 않습니다. 결과는
`reports/bootstrap/<날짜>/`에 저장됩니다.

- 시스템 계정·그룹 `ansible-admin`
- 컨트롤 노드의 승인된 SSH 공개키
- `/etc/sudoers.d/ansible-admin`의 `NOPASSWD` sudo 권한

최초 등록정보는 `reports/inventory/<서버명>.json`에 최신본으로 저장됩니다.
정책을 적용하기 전에는 읽기 전용 사전점검을 별도로 실행합니다.

```bash
ansible-playbook playbooks/03-security-preflight.yml --limit server-01
```

사전점검 보고서는 `reports/preflight/<날짜>/`에 실행 시각별로 보존됩니다.
사전점검에는 계정명과 보안 설정 메타데이터가, 네트워크·계정 정기점검에는
접속 IP와 로그인 이력이 포함될 수 있으므로 `reports/`는 Git에 커밋하지
않습니다.

## 파일럿 적용

기능별 플레이북을 한 대씩 순서대로 적용하는 방식을 권장합니다. 먼저 로그인 전
경고와 시스템 정보 비노출 설정을 check mode로 확인합니다.

```bash
ansible-playbook playbooks/10-01-apply-login-warning.yml \
  --limit server-01 --check --diff
```

주의: 서비스 시작과 command handler 등은 check mode만으로 완전히
검증되지 않습니다. 유지보수 시간에 파일럿 한 대를 실제 적용하고 새 SSH 세션,
`sudo`, `sshd -t`를 확인합니다.

```bash
ansible-playbook playbooks/10-01-apply-login-warning.yml --limit server-01
```

같은 방식으로 NTP, 로컬 로그와 audit를 각각 check mode로 검토한 뒤 실제
적용합니다. 중앙 로그 전송은 SIEM 주소와 TLS 검증 방식을 확정하고
`central_logging_enabled: true`로 변경한 뒤 실행합니다.

```bash
ansible-playbook playbooks/10-02-apply-ntp.yml --limit server-01 --check --diff
ansible-playbook playbooks/10-02-apply-ntp.yml --limit server-01

ansible-playbook playbooks/10-03-apply-local-logging.yml --limit server-01 --check --diff
ansible-playbook playbooks/10-03-apply-local-logging.yml --limit server-01

ansible-playbook playbooks/10-04-apply-audit.yml --limit server-01 --check --diff
ansible-playbook playbooks/10-04-apply-audit.yml --limit server-01
```

각 파일럿 검증 후 `--limit`을 제거하면 모든 서버에 5대씩 적용합니다. 전체
항목을 한 번에 적용해야 할 때만 `10-99-apply-all-logging.yml`을 사용합니다.

## 정기 점검

아래 플레이북은 AWX/AAP 또는 CI 스케줄에 등록합니다.

- 매주: `playbooks/20-weekly-check.yml`
- 매월: `playbooks/30-monthly-time-check.yml`
- 반기: `playbooks/40-semiannual-network-review.yml`
- 분기 또는 반기: `playbooks/50-account-access-review.yml`

최초 등록, 환경설정 변경과 정기 로그·상태 점검의 역할은 다음과 같이 분리되어
있습니다.

| 단계 | 플레이북 | 실행 성격 | 권장 실행 시점 | 보고서 |
|---|---|---|---|---|
| 관리계정 bootstrap | `00-bootstrap-access.yml` | 최초 1회 | 기존 서버를 Ansible 관리 대상으로 전환할 때 | `reports/bootstrap/<날짜>/` |
| 접속 확인 | `01-connectivity.yml` | 조건부 반복 | 신규 등록, SSH·sudo·Python 변경, 접속 장애 시 | 없음 |
| 최초 등록 | `02-register-server.yml` | 최초 1회 | 신규 서버 등록 시 | `reports/inventory/` 최신본 |
| 정책 적용 전 | `03-security-preflight.yml` | 최초 1회 | 최초 로깅·로그인 정책 적용 전 | `reports/preflight/<날짜>/` |
| 로그인 전 경고·정보 비노출 | `10-01-apply-login-warning.yml` | 최초 1회 | 최초 구축 시 | 없음 |
| NTP 시간 동기화 | `10-02-apply-ntp.yml` | 최초 1회 | 최초 구축 시 | 없음 |
| 로컬 로그 보존 | `10-03-apply-local-logging.yml` | 최초 1회 | 최초 구축 시 | 없음 |
| audit 감사 규칙 | `10-04-apply-audit.yml` | 최초 1회 | 최초 구축 시 | 없음 |
| 중앙 로그 전송 | `10-05-apply-central-logging.yml` | 최초 1회 | 중앙 로그 준비 후 | 없음 |
| 전체 로깅 기준 일괄 적용 | `10-99-apply-all-logging.yml` | 최초 1회 | 전체 항목을 함께 적용할 때 | 없음 |
| U0308 Session Timeout(Red Hat 계열) | `controls/U0308-redhat.yml` | 최초 1회 | 자산 진단 체크 U0308 조치 시 | 실행 출력 |
| U0308 Session Timeout(Ubuntu) | `controls/U0308-ubuntu.yml` | 최초 1회 | 자산 진단 체크 U0308 조치 시 | 실행 출력 |
| U0309 root SSH 제한(Red Hat 계열) | `controls/U0309-redhat.yml` | 최초 1회 | 자산 진단 체크 U0309 조치 시 | 실행 출력 |
| U0309 root SSH 제한(Ubuntu) | `controls/U0309-ubuntu.yml` | 최초 1회 | 자산 진단 체크 U0309 조치 시 | 실행 출력 |
| U0510 OpenSSL 업데이트(Red Hat 계열) | `controls/U0510-redhat.yml` | 조건부 반복 | 자산 진단 체크 U0510 조치 또는 신규 보안 업데이트 시 | 실행 출력 |
| U0510 OpenSSL 업데이트(Ubuntu) | `controls/U0510-ubuntu.yml` | 조건부 반복 | 자산 진단 체크 U0510 조치 또는 신규 보안 업데이트 시 | 실행 출력 |
| 로그인 정책 적용(예정) | `11-apply-login-policy.yml` **미구현** | 최초 1회 | Role 구현 후 최초 구축 시 | 없음 |
| 주간 로그 점검 | `20-weekly-check.yml` | 정기 반복 | 매주 1회 이상 | `reports/weekly/<날짜>/` |
| 월간 시간 점검 | `30-monthly-time-check.yml` | 정기 반복 | 매월 1회 이상 | `reports/monthly-time/<날짜>/` |
| 반기 네트워크 점검 | `40-semiannual-network-review.yml` | 정기 반복 | 반기 1회 이상 | `reports/network/<날짜>/` |
| 계정 접근 검토 | `50-account-access-review.yml` | 정기 반복 | 분기 또는 반기 | `reports/account-access/<날짜>/` |

환경설정 적용 플레이북(`10-01`~`10-05`, `10-99`, `controls/`, 향후 `11`)은
최초 구축 또는 취약점 조치 시 실행하고 스케줄에 등록하지 않는다. 중앙 Git
정책이 개정되거나 서버가 재구축되면 새 기준 배포로 다시 실행할 수 있다.
반대로 `20`~`50` 점검 플레이북은 설정을 변경하지 않고 결과를 누적하는 정기
실행 대상이다.

실시간 비정상행위 알림은 Ansible이 아니라 중앙 로그/SIEM에서 구현해야
합니다. Ansible은 audit/SSH/rsyslog 설정을 배포하고 정상 상태를 검증합니다.

NTP는 `time.kaist.ac.kr`, `time.nist.gov`으로 설정되어 있습니다. 실제 적용
전에 대상 망에서 두 이름의 DNS 해석과 UDP/123 응답을 확인해야 합니다.
CentOS/RHEL과 Ubuntu/Debian 모두 chrony를 사용하며, 기존
`systemd-timesyncd`, `ntp`, `ntpd` 서비스가 있으면 중지·비활성화·마스킹합니다.

## 아직 결정해야 하는 항목

- 중앙 로그 서버 주소, 인증서와 CA 검증 방식
- 원시 로그와 요약 로그의 정확한 이벤트 범위
- 서버 역할별 승인 포트 및 서비스 목록
- 애플리케이션/DB 데이터 접근 감사 대상
- 로그 유형별 중앙 보존기간과 저장 용량
- 명령 인자에 포함될 수 있는 비밀번호·토큰 보호 방안
- Ubuntu 16.04 및 CentOS 7용 Ansible/Python 실행 환경

모든 서버는 동일한 정책과 단일 `log_profile`을 사용합니다. 요약 로그의
정의가 확정되기 전에는 원시 로그를 삭제하거나 전송에서 제외하지 않습니다.
