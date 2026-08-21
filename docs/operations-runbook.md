# 구축 및 운영 절차

## 관리 구조

```text
Ansible 컨트롤 노드의 실행 사용자
├── SSH 개인키 보관
└── 대상 서버의 ansible-admin으로 공개키 접속
    └── sudo를 통해 정책 적용
```

개인키는 마스터에만 보관하고 공개키만 대상 서버의 `authorized_keys`에 둔다.
컨트롤 노드에 `ansible-admin` 로컬 계정을 별도로 만들 필요는 없다.

## 마스터 사전조건

- 지원 중인 최신 Rocky Linux 9 마이너 릴리스
- Git, Ansible, Python 3.12 이상
- `ansible-core 2.21.3`
- `ansible.posix 2.2.2` Collection
- 권한 `0600`의 SSH 개인키
- 검증된 대상 SSH 호스트 키
- 대상 TCP/22 네트워크 접근

## bootstrap 전 대상 서버 사전조건

- SSH 접속 가능한 기존 관리자 계정
- 해당 계정의 sudo/root 권한과 안전하게 관리되는 자격증명
- Python 3
- 삭제·잠금하지 않은 콘솔 또는 별도 복구 계정

bootstrap 완료 후에는 다음 항목이 구성된다.

- `ansible-admin` 시스템 계정과 홈 디렉터리
- 컨트롤 노드 공개키가 등록된 `authorized_keys`
- 검증된 `/etc/sudoers.d/ansible-admin` 비대화형 sudo

정책 적용 단계에는 다음 접근 조건도 필요하다.

- 패키지 저장소 접근
- 외부 NTP UDP/123 및 DNS 접근

CentOS 7과 Ubuntu 16.04는 최신 Ansible의 대상 Python 지원 범위에서 벗어날
수 있으므로 Python 3 추가 또는 레거시 실행 환경 분리가 필요하다.

## 권장 실행 순서

```bash
ansible-galaxy collection install -r requirements.yml
```

### 방식 1: hosts.yml과 서버별 Vault

서버 수가 많고 서버별 자격증명을 안전하게 준비할 수 있을 때 사용한다.

```bash
cp inventory/bootstrap/hosts.example.yml inventory/bootstrap/hosts.yml
# 서버별 IP와 bootstrap_initial_user 입력

mkdir -p inventory/bootstrap/host_vars/server-001
ansible-vault create inventory/bootstrap/host_vars/server-001/vault.yml

ansible-playbook -i inventory/bootstrap/hosts.yml \
  playbooks/00-bootstrap-access.yml --ask-vault-pass
```

각 `vault.yml`에는 `ansible_password`, `ansible_become_password`만 저장한다.
모든 파일은 동일한 Vault 암호로 암호화하며 Vault 암호와 파일의 평문 복사본은
Git에 저장하지 않는다.

### 방식 2: 인벤토리 파일 없이 IP 한 대씩 실행

자격증명을 파일에 보관하지 않을 때 사용한다. IP 뒤의 쉼표는 필수다.

```bash
ansible-playbook -i '192.0.2.101,' \
  playbooks/00-bootstrap-access.yml \
  -e bootstrap_hosts=all \
  -e bootstrap_initial_user=기존관리자계정 \
  --ask-pass --ask-become-pass
```

한 서버의 새 `ansible-admin` 재접속 검증까지 성공한 뒤 해당 서버를 운영용
`inventory/hosts.yml`에 추가한다. 이후 다음 서버를 같은 방식으로 처리한다.

### bootstrap 이후 공통 순서

```bash
ansible-inventory --graph
ansible-playbook playbooks/01-connectivity.yml
ansible-playbook playbooks/02-register-server.yml
```

bootstrap은 기존 관리자 접속을 `ansible-admin` 전용 공개키 접속으로 전환하는
최초 1회 작업이다. 기본 10대 배치마다 계정·키·sudoers 구성 직후 기존 SSH
연결을 종료하고 새 계정의 공개키 전용 접속, Python 모듈과 `sudo id`를 모두
검증한다. 한 대라도 실패하면 다음 배치를 시작하지 않는다. 검증 결과는
`reports/bootstrap/<날짜>/`에 남는다.

hosts.yml 방식은 서버별 Vault 또는 AWX·AAP 자격증명을 사용한다. IP 직접 지정
방식은 암호를 프롬프트에 입력하므로 파일에 남기지 않지만 서버마다 반복해야
한다. 기존 관리자 계정은 전체 정책 적용과 복구경로 검증이 끝나기 전에 삭제하지
않는다.

최초 등록정보를 검토한 뒤 파일럿 한 대의 보안 사전점검을 수행한다.

```bash
ansible-playbook playbooks/03-security-preflight.yml \
  --limit server-01
```

사전점검 결과에서 SSH 구문, PAM 관리 방식, 계정 잠금·만료, 중앙인증,
audit immutable 상태와 로그 여유 공간을 확인한 뒤 파일럿에 적용한다.

```bash
ansible-playbook playbooks/10-01-apply-login-warning.yml \
  --limit server-01 --check --diff

ansible-playbook playbooks/10-01-apply-login-warning.yml \
  --limit server-01
```

새 SSH 세션, `sudo -n id`, `sshd -t`, `sshd -T`의 `banner`와 Ubuntu/Debian의
`debianbanner`, 새 일반·시리얼 콘솔의 호스트명 비노출을 확인한다. Role은 콘솔
사용자를 강제로 종료하지 않고 systemd 설정만 다시 읽으므로, 이미 대기 중인
콘솔은 로그아웃 후 새 getty가 시작되거나 재부팅된 뒤 변경된 로그인 프롬프트가
표시된다.

이후 NTP, 로컬 로그와 audit를 각각 check mode로 검토하고 실제 적용한다.

```bash
ansible-playbook playbooks/10-02-apply-ntp.yml \
  --limit server-01 --check --diff
ansible-playbook playbooks/10-02-apply-ntp.yml \
  --limit server-01

ansible-playbook playbooks/10-03-apply-local-logging.yml \
  --limit server-01 --check --diff
ansible-playbook playbooks/10-03-apply-local-logging.yml \
  --limit server-01

ansible-playbook playbooks/10-04-apply-audit.yml \
  --limit server-01 --check --diff
ansible-playbook playbooks/10-04-apply-audit.yml \
  --limit server-01
```

`chronyc tracking`, `systemctl status systemd-journald rsyslog`, `auditctl -s`와
`auditctl -l`을 확인한 후 각 플레이북에서 `--limit`을 제거하여 전체 적용한다.
모든 적용 플레이북은 5대씩 처리한다. 전체 항목을 한 번에 적용해야 할 때만
`10-99-apply-all-logging.yml`을 사용한다. 중앙 로그는 SIEM 주소와 TLS 검증
방식을 확정하고 `central_logging_enabled: true`로 변경한 뒤
`10-05-apply-central-logging.yml`로 적용한다.

시간 동기화는 모든 배포판에서 chrony로 통일한다. Role은 Ubuntu/Debian의
apt 캐시를 갱신하고, 기존 `systemd-timesyncd`, `ntp`, `ntpd` unit이 있으면
중지·비활성화·마스킹한 뒤 chrony 실행 상태와 단독 실행 여부를 검증한다.

## 정기 운영

| 주기 | 플레이북 | 목적 |
|---|---|---|
| 매주 | `20-weekly-check.yml` | 로그 용량과 audit 상태 |
| 매월 | `30-monthly-time-check.yml` | 시간 오차 기준 확인 |
| 반기 | `40-semiannual-network-review.yml` | 네트워크 서비스 현황수집 |
| 분기 또는 반기 | `50-account-access-review.yml` | 계정·관리자·로그인·잠금·만료 현황 |

최초 등록 보고서는 서버별 최신본으로 관리한다. 사전점검과 정기점검 보고서는
`reports/<점검유형>/<날짜>/`에 실행 시각이 포함된 파일로 저장하여 이전 결과를
덮어쓰지 않는다. 이 보고서에는 사용자 ID와 접속 IP가 포함될 수 있으므로 Git에
커밋하지 않고 접근권한 `0600`을 유지한다.

운영 단계에서는 AWX/AAP 또는 CI 스케줄을 권장한다. 단순 cron도 가능하지만
실행 승인, RBAC, 자격증명 보호, 감사 이력과 실패 알림이 부족하다.

## 안전 원칙

- 실제 IP, 비밀번호, 개인키와 Vault 비밀번호를 Git에 커밋하지 않는다.
- bootstrap 전 기존 관리자 sudo와 별도 콘솔 복구경로를 확인한다.
- bootstrap 검증 성공 전 기존 관리자 계정을 삭제하거나 잠그지 않는다.
- `--check --diff` 후에도 파일럿 실제 적용을 별도로 수행한다.
- SSH/PAM 변경 시 기존 관리자 세션과 콘솔 복구 경로를 유지한다.
- root SSH 및 비밀번호 인증 차단은 공개키·sudo·새 SSH 세션 검증 후 시행한다.
- PAM 정책 적용 자체를 위해 서버를 재부팅하지 않는다.
- 승인 목록 없이 발견한 포트나 서비스를 자동 종료하지 않는다.

## 로그인 정책 무재부팅 적용 절차

로그인 정책 Role이 구현되면 로깅 Role과 분리하여 다음 순서로 적용한다.

1. `serial: 1`, `max_fail_percentage: 0`으로 한 번에 한 서버만 처리한다.
2. 기존 관리자 SSH 세션을 닫지 않고 PAM과 SSH 원본 설정을 백업한다.
3. 해당 OS의 `authselect`, `pam-auth-update` 또는 검증된 레거시 방식으로
   PAM 설정을 적용한다.
4. `sshd -t` 성공 후 SSH 서비스를 reload한다. restart는 사용하지 않는다.
5. 새 SSH 세션에서 공개키 인증, 비밀번호 인증, `sudo`, 잠금과 해제를
   검증한다.
6. 모든 검증이 성공한 경우에만 다음 서버로 진행한다.

PAM 설정은 다음 인증부터 적용되므로 재부팅할 필요가 없다. SSSD 설정까지
변경하는 경우 해당 서비스 재시작이 필요할 수 있으나 서버 재부팅은 하지 않는다.
audit 규칙에 immutable 모드(`-e 2`)가 이미 활성화된 경우 그 규칙을 변경하려면
재부팅이 필요할 수 있으므로 초기 설계에서는 기본값을 비활성화한다.
