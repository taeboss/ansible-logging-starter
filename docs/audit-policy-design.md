# audit 정책 상세 설계

이 문서는 기존 기본 감사 규칙을 확장하기 위한 설계안이다. 아래 파일 구조와
변수는 아직 Role에 구현되지 않았으며 현재 플레이북 실행 결과를 바꾸지 않는다.

## 규칙 파일 구조

```text
roles/logging_baseline/templates/audit/
├── 30-login.rules.j2
├── 40-identity-config.rules.j2
├── 50-access-denied.rules.j2
├── 60-privileged.rules.j2
├── 70-package-cron.rules.j2
├── 80-user-command.rules.j2
└── 99-finalize.rules.j2
```

| 파일 | 감사 대상 | 설계 요점 |
|---|---|---|
| `30-login` | utmp, wtmp, btmp, lastlog | 로그인·로그아웃·실패와 세션 근거를 보존한다. |
| `40-identity-config` | 계정, sudo, SSH, PAM, 네트워크, sysctl, systemd 설정 | 존재하는 OS별 경로만 watch하고 동일한 key 이름을 사용한다. |
| `50-access-denied` | `EACCES`, `EPERM`, 삭제·이름변경·권한변경 실패 | 일반 로그인 사용자의 성공·실패를 분리하고 b32/b64 아키텍처를 처리한다. |
| `60-privileged` | sudo, su, systemctl, passwd, chage, 사용자 관리 명령 | 실행을 기록한다. SUID/SGID 기준 목록은 비교만 하고 자동 삭제하지 않는다. |
| `70-package-cron` | apt, dpkg, yum, dnf, rpm 및 cron 설정 | 패키지 도구 실행과 배포판 이력 파일을 함께 사용한다. cron 실제 실행은 syslog/journal로 확인한다. |
| `80-user-command` | 로그인 사용자의 `execve` | `auid` 기준으로 명령을 기록하되 인자에 포함된 비밀번호·토큰 노출과 로그량을 점검한다. |
| `99-finalize` | 실패 모드와 immutable 옵션 | 초기에는 `-e 2`를 사용하지 않는다. 운영 승인 후 선택적으로 활성화한다. |

cron 실행 프로세스는 시스템 계정으로 동작해 `auid`가 설정되지 않을 수 있으므로
모든 실행을 syscall 하나로 포착하려 하면 잡음과 부하가 커질 수 있다. cron 설정
변경은 auditd, 실제 실행 성공·실패는 cron facility와 journald/rsyslog를 함께
사용하는 방식으로 설계한다.

## 정책 변수 초안

```yaml
audit_policy:
  minimum_user_uid: 1000
  record_login_sessions: true
  record_user_commands: true
  record_failed_file_access: true
  record_delete_rename: true
  record_permission_changes: true
  record_privileged_commands: true
  record_service_management: true
  record_package_management: true
  record_cron_configuration: true
  immutable: false
```

`minimum_user_uid`는 `/etc/login.defs`의 `UID_MIN`을 우선 수집하고 1000은
fallback으로만 사용한다. `immutable: true`로 `-e 2`를 적용하면 이후 audit
규칙 변경에 재부팅이 필요하므로 최초 배포와 조정 기간에는 사용하지 않는다.

## 배포와 검증

1. OS와 아키텍처, 기존 규칙, `UID_MIN`, 현재 backlog/lost 값을 수집한다.
2. 파일럿 서버에서 규칙 구문을 검사하고 로드한다.
3. `auditctl -l`, `auditctl -s`로 규칙, enabled, failure, backlog, lost 상태를
   검증한다.
4. 로그인 실패, sudo, 파일 접근 거부, 패키지·서비스 작업 테스트 이벤트가
   각각 의도한 key로 생성되는지 확인한다.
5. 최소 1주간 일평균 용량과 성능 영향을 측정한 후 단계적으로 확대한다.

구문검사 또는 로드에 실패하면 해당 호스트에서 즉시 중단한다. 운영 중에는
`lost > 0` 또는 지속적인 backlog 증가를 경보 조건으로 사용한다. 규칙 배포와
auditd 설정 갱신은 일반적으로 재부팅 없이 가능하지만 immutable 모드가 이미
활성화된 서버의 규칙 변경은 예외이다.

## 참고 문서

- [Red Hat Enterprise Linux 9 감사 가이드](https://docs.redhat.com/en/documentation/red_hat_enterprise_linux/9/html/security_hardening/auditing-the-system_security-hardening)
- [audit.rules(7)](https://man7.org/linux/man-pages/man7/audit.rules.7.html)
