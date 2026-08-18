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
ansible-playbook playbooks/01-discovery.yml
```

현황 결과를 검토한 뒤 파일럿 한 대에 적용한다.

```bash
ansible-playbook playbooks/10-apply-logging.yml \
  --limit server-01 --check --diff

ansible-playbook playbooks/10-apply-logging.yml \
  --limit server-01
```

새 SSH 세션, `sudo -n id`, `sshd -t`, `chronyc tracking`, `auditctl -s`,
`auditctl -l`을 확인한 후 전체 적용한다. 전체 플레이북은 5대씩 처리한다.

## 정기 운영

| 주기 | 플레이북 | 목적 |
|---|---|---|
| 매주 | `20-weekly-check.yml` | 로그 용량과 audit 상태 |
| 매월 | `30-monthly-time-check.yml` | 시간 오차 기준 확인 |
| 반기 | `40-semiannual-network-review.yml` | 네트워크 서비스 현황수집 |

운영 단계에서는 AWX/AAP 또는 CI 스케줄을 권장한다. 단순 cron도 가능하지만
실행 승인, RBAC, 자격증명 보호, 감사 이력과 실패 알림이 부족하다.

## 안전 원칙

- 실제 IP, 비밀번호, 개인키와 Vault 비밀번호를 Git에 커밋하지 않는다.
- `--check --diff` 후에도 파일럿 실제 적용을 별도로 수행한다.
- SSH/PAM 변경 시 기존 관리자 세션과 콘솔 복구 경로를 유지한다.
- root SSH 및 비밀번호 인증 차단은 공개키·sudo·재부팅 검증 후 시행한다.
- 승인 목록 없이 발견한 포트나 서비스를 자동 종료하지 않는다.
