# 로그 보존정책 상세 설계

이 문서는 1년 이상 보존 요구사항을 구현하기 위한 설계안이다. 현재 소스의
`MaxRetentionSec=365day`와 4GB 상한만으로는 1년 보존을 보장할 수 있으므로,
실측 용량과 중앙 로그 저장소가 확정되기 전까지 현재 상태는 부분 구현이다.

## 로그 유형별 기준

| 유형 | 예시 | 최소 보존 초안 | 담당 계층 |
|---|---|---:|---|
| 인증·접속 | sshd, PAM, wtmp, btmp, lastlog | 365일 | journald/rsyslog 및 중앙 저장소 |
| 감사 | auditd 규칙 이벤트 | 365일 | auditd 및 중앙 저장소 |
| 사용자 명령 | audit `execve` | 365일 | auditd 및 중앙 저장소 |
| 시스템·커널·부팅 | kernel, boot, service | 365일 | journald/rsyslog 및 중앙 저장소 |
| cron·패키지 | cron facility, apt/dpkg/yum/dnf/rpm | 365일 | 배포판 로그 및 중앙 저장소 |
| 애플리케이션·DB | 제품별 접근·오류 로그 | 미정 | 제품 담당자와 별도 결정 |

로컬 보존과 중앙 보존을 같은 것으로 간주하지 않는다. 로컬 디스크가 부족하면
중앙 저장소에서 365일을 보장할 수 있으나, 전송 지연·유실 검증과 중앙 저장소의
불변 또는 append-only 통제가 함께 필요하다.

## 용량 산정

각 로그 유형의 최소 필요 공간은 다음 식으로 산정한다.

```text
일평균 로그량 × 365일 × 안전계수 1.3
```

파일럿 서버에서 최소 1주간 실제 발생량을 측정하고, 서버 역할별 최대값과 장애
시 폭증량을 반영한다. 현재 `SystemMaxUse=4G`는 임시 상한일 뿐이며 산정값보다
작으면 365일 전에 오래된 journal이 삭제된다.

## journald 변수 초안

```yaml
journal_policy:
  retention_days: 365
  system_max_use: TBD
  system_keep_free: TBD
  compress: true
  seal: true
  forward_to_syslog: true
```

기간과 용량 제한 중 먼저 도달한 조건이 적용되므로 `system_max_use`는 실측 후
정한다. `system_keep_free`도 별도 지정해 로그가 파일시스템 전체를 소진하지
않도록 한다.

## auditd 변수 초안

아래 값은 파일럿 출발점이며 서버별 일평균 로그량과 `/var/log/audit` 크기에 따라
조정한다.

```yaml
audit_retention_policy:
  flush: INCREMENTAL_ASYNC
  freq: 100
  max_log_file_mb: 100
  max_log_file_action: ROTATE
  num_logs: 50
  space_left_percent: 10
  space_left_action: SYSLOG
  admin_space_left_percent: 5
  admin_space_left_action: SUSPEND
  disk_full_action: SUSPEND
  disk_error_action: SYSLOG
```

초기 운영은 가용성을 고려해 `ROTATE`와 `SUSPEND`를 사용한다. 로그를 삭제하지
않는 `KEEP_LOGS`, 시스템을 single-user 또는 halt 상태로 전환하는 동작은 별도
audit 파티션, 중앙 전송, 복구 인력과 영향도 승인을 갖춘 뒤 선택한다.

## 주간 무결성·용량 점검 목표

아래 목록은 목표 상태이다. 현재 `20-weekly-check.yml`에는 파일시스템 용량과
inode, journal 사용량, audit 상태 및 규칙 파일 체크섬까지만 구현되어 있다.

- 로그 파일시스템의 사용률과 inode, 예상 잔여 보존일
- `auditctl -s`의 enabled, failure, backlog, lost 값
- 현재 로드된 audit key와 배포된 규칙의 일치 여부
- `journalctl --verify` 결과와 journal 디스크 사용량
- audit, auth, journal 파일 및 디렉터리 소유자·권한
- 최근 7일의 비정상적인 시간 공백과 중앙 전송 큐·오류
- 설정파일 승인 체크섬과 AIDE 등 기준 무결성 결과

계속 변경되는 원시 로그 파일의 SHA-256을 고정값과 비교하는 방식은 사용하지
않는다. 설정파일은 승인 체크섬으로 검증하고, 원시 로그는 journal 검증,
연속성·전송 건수, 중앙 저장소의 append-only 또는 WORM 정책으로 검증한다.

## 구현 전 결정할 정보

- 서버별 일평균·최대 로그 발생량과 `/var/log` 가용 공간
- `/var/log/audit` 전용 파티션 사용 여부
- 365일 전체를 로컬과 중앙 중 어디에서 보장할지
- 디스크 부족 시 로그 중단, 서비스 제한, single-user, halt 중 허용 동작
- 중앙 로그 제품, 전송 TLS, 저장 불변성, 유형별 보존 및 삭제 권한

journald, rsyslog, auditd 설정 갱신은 정상적인 경우 서비스 reload 또는 restart로
처리하며 서버 재부팅을 요구하지 않는다. 단, audit immutable 모드가 이미
활성화된 상태에서 규칙을 변경하는 경우는 예외이다.

## 참고 문서

- [Red Hat Enterprise Linux 9 감사 가이드](https://docs.redhat.com/en/documentation/red_hat_enterprise_linux/9/html/security_hardening/auditing-the-system_security-hardening)
