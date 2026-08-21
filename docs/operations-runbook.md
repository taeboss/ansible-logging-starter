# 구축 및 운영 절차

## 관리 구조

```text
Ansible 마스터의 ansible-admin
├── SSH 개인키 보관
└── 대상 서버 20대의 ansible-admin으로 공개키 접속
    └── sudo를 통해 정책 적용
```

개인키는 마스터에만 보관하고 공개키만 대상 서버의 `authorized_keys`에 둔다.

## 마스터 사전조건

- Git, Ansible, Python 3
- `ansible.posix` Collection
- `ansible-admin` 실행 계정
- 권한 `0600`의 SSH 개인키
- 검증된 대상 SSH 호스트 키
- 대상 TCP/22 네트워크 접근

## 대상 서버 사전조건

- `ansible-admin` 계정과 홈 디렉터리
- 마스터 공개키가 등록된 `authorized_keys`
- 비대화형 sudo 또는 안전하게 관리되는 become 자격증명
- SSH와 호환 가능한 Python
- 패키지 저장소 접근
- 외부 NTP UDP/123 및 DNS 접근

CentOS 7과 Ubuntu 16.04는 최신 Ansible의 대상 Python 지원 범위에서 벗어날
수 있으므로 Python 3 추가 또는 레거시 실행 환경 분리가 필요하다.

## 권장 실행 순서

```bash
ansible-galaxy collection install -r requirements.yml
ansible-inventory --graph
ansible-playbook playbooks/00-connectivity.yml
ansible-playbook playbooks/01-register-server.yml
```

최초 등록정보를 검토한 뒤 파일럿 한 대의 보안 사전점검을 수행한다.

```bash
ansible-playbook playbooks/02-security-preflight.yml \
  --limit server-01
```

사전점검 결과에서 SSH 구문, PAM 관리 방식, 계정 잠금·만료, 중앙인증,
audit immutable 상태와 로그 여유 공간을 확인한 뒤 파일럿에 적용한다.

```bash
ansible-playbook playbooks/10-apply-logging.yml \
  --limit server-01 --check --diff

ansible-playbook playbooks/10-apply-logging.yml \
  --limit server-01
```

새 SSH 세션, `sudo -n id`, `sshd -t`, `sshd -T`의 `banner`와 Ubuntu/Debian의
`debianbanner`, 새 일반·시리얼 콘솔의 호스트명 비노출, `chronyc tracking`,
`auditctl -s`, `auditctl -l`을 확인한 후 전체 적용한다. 전체 플레이북은 5대씩
처리한다. Role은 콘솔 사용자를 강제로 종료하지 않고 systemd 설정만 다시
읽으므로, 이미 대기 중인 콘솔은 로그아웃 후 새 getty가 시작되거나 재부팅된 뒤
변경된 로그인 프롬프트가 표시된다.

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
