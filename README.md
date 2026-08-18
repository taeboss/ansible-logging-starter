# Ansible Linux 로깅 기준선 스타터

CentOS/RHEL 계열과 Ubuntu 서버의 로그인 경고, 시간 동기화, auditd,
journald, rsyslog 및 정기 점검을 단계적으로 관리하기 위한 시작점입니다.

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
