# Ansible Linux 로깅 기준선 스타터

CentOS/RHEL 계열과 Ubuntu 서버의 로그인 경고, 시간 동기화, auditd,
journald, rsyslog 및 정기 점검을 단계적으로 관리하기 위한 시작점입니다.

## 설계 문서

- [결정사항과 구현 현황](docs/decisions-and-status.md)
- [로깅 정책 매핑](docs/logging-policy-mapping.md)
- [로그인 정책 설계 초안](docs/login-policy-design.md)
- [구축 및 운영 절차](docs/operations-runbook.md)

## 정책 적용 현황 요약

이 표의 상태는 정책상 가능한 기능이 아니라 **현재 저장소의 소스코드 기준**이다.

- ✅ 적용 가능: 현재 플레이북 또는 Role에 구현되어 있다.
- 🟡 부분 적용: 일부 구현됐지만 추가 정보, 스케줄 또는 외부 시스템이 필요하다.
- ⚪ 미구현: Ansible로 구현할 수 있지만 현재 실행 코드에는 없다.
- 🚫 외부 절차: Ansible만으로 보장할 수 없는 사람·IAM·SIEM 영역이다.

### 로깅 정책

| 정책 항목 | 상태 | 현재 소스 기준 설명 |
|---|---:|---|
| 로그인 전 시스템 정보 비노출 및 인가 사용자 경고 | ✅ | 고정 문구를 `/etc/issue`, `/etc/issue.net`에 배포하고 SSH Banner를 설정한다. 문구에 호스트명·OS·버전 변수를 사용하지 않는다. |
| 불필요한 로그인 안내문 제거 | 🟡 | 사전 인증 Banner는 통제하지만 Ubuntu 동적 MOTD와 기타 프로그램별 안내문 제거는 구현하지 않았다. |
| 로그 및 감사 기록의 관리자 전용 관리 | 🟡 | `logadmin` 그룹, `/var/log/compliance`, audit 규칙 권한만 구현했다. 관리자 명단, 전체 로그 권한과 sudo 정책이 필요하다. |
| chrony 기반 외부 NTP 동기화 | ✅ | `time.kaist.ac.kr`, `time.bora.net`을 사용하도록 chrony를 설치·설정한다. 적용 전에 DNS와 UDP/123 응답을 확인해야 한다. |
| 월 1회 시간 오차 점검 | 🟡 | 오차 점검 플레이북은 있지만 AWX/AAP·CI·cron 스케줄은 등록되지 않았다. |
| 반기 네트워크 서비스 현황 점검 | 🟡 | LISTEN·연결 소켓 보고서는 생성하지만 정기 스케줄, 승인 목록 비교와 특이사항 판단은 없다. |
| 관리 서버 원시 로그, 기타 서버 요약 로그 | ⚪ | 모든 서버를 단일 정책과 `raw` 변수로 통일했다. `log_profile`에 따른 실제 필터링은 구현되지 않았다. |
| 가장 최근에 로그인한 사용자 | ✅ | discovery에서 `lastlog` 결과를 서버별 JSON으로 수집한다. |
| 사용자별 로그인 시간 | ✅ | `lastlog`, `last -F` 결과와 wtmp/lastlog 감사 규칙을 사용한다. |
| 사용자별 로그인·로그아웃 | ✅ | utmp, wtmp, btmp, lastlog 변경을 auditd가 기록하도록 설정한다. |
| 접속 터미널 ID와 위치 | 🟡 | 세션 파일과 SSH `LogLevel VERBOSE`로 TTY·원격 IP를 확보할 수 있다. 물리 위치는 VPN/NAC/CMDB 연계가 필요하다. |
| 서버 접근 성공 및 실패 | 🟡 | SSH 상세 로그와 로그인 세션·실패 파일은 설정한다. 중앙 상관분석과 서비스별 접근 감사는 없다. |
| 데이터 및 다른 자원 접근 성공·실패 | 🟡 | 선택한 경로 변경과 일반 사용자의 `EACCES/EPERM` 파일 접근 실패를 기록한다. 업무 데이터·DB·앱 감사 대상은 미정이다. |
| 시스템·커널·부팅 이벤트 | ✅ | journald를 영구 저장하고 rsyslog로 전달하도록 설정한다. |
| 주요 시스템 및 네트워크 설정파일 | ✅ | 계정, sudo, SSH, PAM, hosts, resolv, sysctl 등 존재하는 경로를 audit watch로 설정한다. |
| cron 실행 및 패키지 업데이트 이력 | 🟡 | cron 설정파일 변경은 감사하지만 cron의 실제 실행과 yum/dnf/apt/dpkg 이력 규칙은 없다. |
| 사용자 명령어 실행 이력 | ✅ | 로그인 사용자의 `auid`를 기준으로 `execve`를 감사한다. 명령 인자의 민감정보 보호정책은 추가로 필요하다. |
| 로그 1년 이상 보관 | 🟡 | journald에 365일을 설정했지만 4GB 제한으로 조기 삭제될 수 있다. audit/logrotate 및 중앙 저장소 보존정책이 필요하다. |
| 주 1회 무결성·용량 점검 | 🟡 | 용량 임계치, audit 상태와 규칙 파일 SHA-256을 점검한다. 승인 체크섬 비교와 실행 스케줄은 없다. |
| 지속적인 로그인 실패 알림 | 🚫 | 감사 이벤트 생성은 가능하지만 실시간 분석과 알림은 SIEM 영역이다. |
| 관리자 권한 파일·서비스 사용 시도 | 🟡 | sudoers 변경과 일반 접근 거부는 일부 기록한다. sudo/su/systemctl/SUID 실행 감사 규칙은 미구현이다. |
| 비인가 파일·자원·서비스 사용 시도 | 🟡 | 파일 접근 실패는 기록한다. 승인 파일·포트·서비스 목록 및 실시간 비교·알림은 없다. |
| 행위별 성공·실패 구분과 유형별 보존 | 🟡 | 접근 실패 errno 규칙은 있다. 전체 이벤트의 성공·실패 규칙과 로그 유형별 보존기간은 미정이다. |

### 로그인 정책

> 현재 저장소에는 `login_policy` Role과 `11-apply-login-policy.yml`이 없다.
> 따라서 아래 항목은 설계 문서에만 있고 어떤 로그인 정책도 아직 서버에
> 적용되지 않는다.

| 정책 항목 | 상태 | Ansible 적용 가능 범위 또는 미구현 사유 |
|---|---:|---|
| 1인 1계정 발급 | 🚫 | 계정 생성·중복 UID 점검은 자동화할 수 있지만 실제 사용자와 계정의 1:1 관계는 HR/IAM·발급 절차가 보장해야 한다. |
| 패스워드는 사용자 본인이 관리 | 🚫 | Ansible은 비밀번호 자체가 아닌 복잡도·만료·이력 정책만 관리해야 한다. |
| 2종류 10자 또는 3종류 8자, 권고 12~64자 | ⚪ | `pam_pwquality`로 구현 가능하지만 조건부 OR를 배포판 공통으로 정확히 표현하기 어려워 최소 12자·2종류의 단일 기준을 설계했다. 아직 코드화하지 않았다. |
| Null 패스워드 금지 | ⚪ | `PermitEmptyPasswords no`와 `/etc/shadow` 점검으로 구현 가능하나 미구현이다. |
| 사용자 ID와 동일한 패스워드 금지 | ⚪ | `pam_pwquality usercheck`로 구현 가능하나 구형 OS 지원 확인과 Role 작성이 필요하다. |
| 예측 가능한 패스워드 금지 | ⚪ | 사전검사, 반복·연속문자 제한은 가능하다. 조직명·연도 등 모든 패턴의 완전한 차단은 별도 사전/IAM이 필요하다. |
| 주기성 패스워드 재사용 금지 | ⚪ | 동일한 최근 비밀번호는 `pam_pwhistory`로 차단 가능하다. 숫자·계절명 변형 패턴은 표준 PAM만으로 완전히 막기 어렵다. |
| 패스워드 힌트 금지 | 🚫 | 일반 Linux 로컬계정의 표준 기능이 아니며 포털·메일·헬프데스크 절차에서 통제해야 한다. |
| 초기 패스워드 최초 접속 시 변경 | ⚪ | 계정 발급 후 `chage -d 0`으로 강제 가능하지만 사용자 목록과 발급 흐름이 정해지지 않았다. |
| 로그인 실패 10회 이하·로그·연결 해제 | ⚪ | `pam_faillock`, SSH `MaxAuthTries`, audit/PAM 로그로 가능하다. OS별 PAM 방식과 잠금시간이 확정되지 않았다. |
| 비밀번호 6개월 만료, MFA 사용자는 1년 | ⚪ | `chage -M 180/365`로 가능하다. 사람·서비스 계정 구분과 MFA 대상 원천정보가 필요하다. |
| 직전 2개 비밀번호 재사용 금지 | ⚪ | `pam_pwhistory remember=2`로 가능하지만 OS별 PAM Role이 아직 없다. |
| 일방향 암호화 저장 | ⚪ | OS별 SHA-512 또는 yescrypt 정책 적용이 가능하다. 기존 해시는 다음 비밀번호 변경 때 전환된다. 아직 코드화하지 않았다. |
| 분실·도난 시 본인 확인 후 재발급 | 🚫 | 휴대폰·이메일 본인 확인과 승인은 IAM·헬프데스크 영역이다. 승인 후 임시 해시 적용과 즉시 변경만 자동화할 수 있다. |

### 현재 실행 가능한 범위

현재 실제 설정 적용 진입점은 로깅 전용 플레이북뿐이다.

```bash
ansible-playbook playbooks/10-apply-logging.yml
```

로그인 정책은 별도 Role을 구현하기 전까지 실행 명령이 존재하지 않는다.

## 적용 전 수정할 값

1. `inventory/hosts.yml`의 예시 IP(`192.0.2.x`)를 실제 IP로 교체합니다.
2. 모든 대상 서버를 단일 `linux_servers` 그룹에 넣습니다.
3. `inventory/group_vars/all.yml`에서 SSH 개인키 경로, 경고문,
   NTP 서버, 보존 기준을 확정합니다.
4. 중앙 로그/SIEM이 준비되기 전에는 `central_logging_enabled: false`를
   유지합니다.

## 최초 실행

```bash
cd ansible-logging-starter
ansible-galaxy collection install -r requirements.yml
ansible-inventory --graph
ansible-playbook playbooks/00-connectivity.yml
ansible-playbook playbooks/01-discovery.yml
```

현황 보고서는 `reports/<서버명>.json`에 저장됩니다.

## 파일럿 적용

먼저 한 대에 check mode를 실행합니다.

```bash
ansible-playbook playbooks/10-apply-logging.yml \
  --limit server-01 --check --diff
```

주의: 서비스 시작과 command handler 등은 check mode만으로 완전히
검증되지 않습니다. 유지보수 시간에 파일럿 한 대를 실제 적용하고 새 SSH
세션, `sudo`, `sshd -t`, `chronyc tracking`, `auditctl -s`를 확인합니다.

```bash
ansible-playbook playbooks/10-apply-logging.yml --limit server-01
```

파일럿 확인 후 5대 단위로 실행합니다. 플레이북에 `serial: 5`가 설정되어
있습니다.

```bash
ansible-playbook playbooks/10-apply-logging.yml
```

## 정기 점검

아래 플레이북은 AWX/AAP 또는 CI 스케줄에 등록합니다.

- 매주: `playbooks/20-weekly-check.yml`
- 매월: `playbooks/30-monthly-time-check.yml`
- 반기: `playbooks/40-semiannual-network-review.yml`

실시간 비정상행위 알림은 Ansible이 아니라 중앙 로그/SIEM에서 구현해야
합니다. Ansible은 audit/SSH/rsyslog 설정을 배포하고 정상 상태를 검증합니다.

NTP는 `time.kaist.ac.kr`, `time.bora.net`으로 설정되어 있습니다. 실제 적용
전에 대상 망에서 두 이름의 DNS 해석과 UDP/123 응답을 확인해야 합니다.

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
