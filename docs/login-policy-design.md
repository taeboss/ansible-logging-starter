# 로그인 정책 잔여 요구사항 설계

이 문서는 아직 구현되지 않은 별도 `login_policy` Role의 설계 기준이다.
PAM 변경은 접속 장애 위험이 있으므로 OS별 현황수집과 단일 서버 파일럿 이후
구현·적용한다. PAM 설정 변경은 다음 인증부터 적용되므로 원칙적으로 재부팅하지
않는다. 자산 진단 체크 취약점별 플레이북이 소유한 설정은 이 Role에서 중복
적용하지 않는다.

## 기능 분류

| 정책 | 구분 | 구현 또는 통제 방법 |
|---|---|---|
| Session Timeout 300초 이하 | 취약점별 자동 강제 | U0308이 `/etc/profile`에서 관리하므로 이 Role에서는 중복 적용하지 않음 |
| 1인 1계정 | 일부 가능 | 선언된 계정 생성·중복 UID 점검, 실제 소유자는 IAM/HR 확인 |
| 비밀번호 본인 관리 | 절차 | Ansible은 비밀번호가 아니라 정책만 관리 |
| 길이와 문자 종류 | 자동 강제 | `pam_pwquality`, 최소 12자·2종류 초안 |
| Null 비밀번호 금지 | 일부 자동 강제 | SSH `PermitEmptyPasswords no`는 U0309에서 적용, 이 Role은 shadow 점검만 담당 |
| 사용자 ID와 동일 금지 | 자동 강제 | `usercheck=1` |
| 예측 가능한 비밀번호 금지 | 일부 가능 | 사전·연속·반복문자 검사 |
| 주기성 재사용 금지 | 일부 가능 | 정확히 같은 최근 비밀번호는 이력으로 차단 |
| 패스워드 힌트 금지 | 절차 | Linux 로컬계정 외 포털·헬프데스크 통제 |
| 최초 로그인 즉시 변경 | 자동 강제 | `chage -d 0` |
| 실패 10회 이하·잠금 | 자동 강제 | `pam_faillock` 또는 레거시 PAM |
| 실패 로그와 연결 해제 | 자동 강제 | PAM/audit/SSH, `MaxAuthTries` |
| 6개월 만료 | 자동 강제 | 기존 사용자 `chage -M 180` |
| MFA 사용자는 1년 | 일부 가능 | IAM이 제공한 MFA 여부에 따라 `chage -M 365` |
| 직전 2개 재사용 금지 | 자동 강제 | `pam_pwhistory remember=2` |
| 일방향 해시 | 자동 강제 | OS별 SHA-512 또는 yescrypt |
| 분실 시 본인 확인 | 절차 | IAM·헬프데스크 담당 |
| 승인 후 재발급 | 일부 가능 | 임시 해시 적용과 최초 변경 강제 |

## 공통 정책값 초안

```yaml
password_policy:
  min_length: 12
  min_classes: 2
  dictionary_check: true
  user_name_check: true
  max_sequence: 3
  max_repeat: 3
  history_count: 2
  max_age_days: 180
  mfa_max_age_days: 365
  min_age_days: 1
  warning_days: 14

login_policy:
  deny: 10
  fail_interval_seconds: 900
  unlock_time_seconds: 1800
  ssh_max_auth_tries: 6
  lock_root: false
  allow_reboot: false

login_policy_rollout:
  serial: 1
  max_fail_percentage: 0
```

`ssh_max_auth_tries`는 정책의 상한인 10회보다 낮은 6회로 시작한다. root 계정은
원격 잠금 유발 공격과 복구 불능을 피하기 위해 초기에는 `pam_faillock` 대상에서
제외한다. 이 예외는 root SSH 허용을 의미하지 않는다.

비밀번호 만료 정책은 선언된 일반 사용자 계정에만 적용한다. `ansible-admin`,
서비스 계정, 시스템 계정에는 일괄 적용하지 않으며, MFA 사용자의 365일 기준은
IAM 등 신뢰 가능한 대상 목록이 제공될 때만 사용한다.

## 예정 구조

```text
playbooks/11-apply-login-policy.yml
roles/login_policy/
├── defaults/main.yml
├── tasks/main.yml
├── tasks/preflight.yml
├── tasks/redhat7.yml
├── tasks/redhat8plus.yml
├── tasks/ubuntu_legacy.yml
├── tasks/ubuntu_modern.yml
├── tasks/verify.yml
├── handlers/main.yml
└── templates/
    ├── pwquality.conf.j2
    └── faillock.conf.j2
```

RHEL 8 이상은 `authselect`, Ubuntu는 `pam-auth-update`, 구형 OS는 지원 가능한
PAM 모듈을 확인해야 한다. PAM 파일을 배포판 구분 없이 덮어쓰지 않는다.

## 무재부팅 적용 원칙

1. 한 번에 한 서버만 처리하고 기존 관리자 SSH 세션을 유지한다.
2. PAM 원본 파일과 현재 `authselect` 또는 `pam-auth-update` 상태를 백업한다.
3. RHEL 8 이상은 사용자 정의 `authselect` 프로필, Ubuntu는
   `pam-auth-update`, RHEL 7은 검증된 레거시 절차로 변경한다.
4. 이 Role이 관리하는 `sshd` 설정은 `sshd -t` 검증이 성공한 경우에만 반영한다.
   U0308의 Session Timeout과 U0309의 root SSH 접근 제한은 취약점별 플레이북에서
   별도로 관리한다.
5. 별도의 새 세션에서 공개키 로그인, 비밀번호 로그인, `sudo`, 실패 잠금과
   잠금 해제를 검증한 뒤 다음 서버로 진행한다.
6. 검증 실패 시 기존 세션으로 즉시 백업 설정을 복원한다.

PAM 설정 자체는 재부팅이나 SSH 서비스 재시작이 필요하지 않다. 단, 이미 열린
인증 세션에는 새 정책이 소급되지 않는다. SSSD 설정을 함께 바꾼 경우에는 SSSD
재시작이 필요할 수 있지만 서버 재부팅은 하지 않는다. 커널 업데이트나 audit
immutable 모드 해제처럼 PAM 외부 사유의 재부팅은 이 Role이 수행하지 않는다.

## 구현 전 확인할 정보

- 실제 OS별 PAM 스택과 `authselect`/`pam-auth-update` 관리 여부
- 로컬 일반 사용자, 서비스 계정, 비상복구 계정 목록
- LDAP/AD/SSSD 사용 여부와 MFA 적용 사용자 원천
- 잠금 해제 절차와 콘솔 또는 out-of-band 복구 경로
- Ubuntu 16.04와 CentOS 7 파일럿 지원 여부

## 참고 문서

- [pam_faillock(8)](https://man7.org/linux/man-pages/man8/pam_faillock.8.html)
- [pam_pwhistory(8)](https://man7.org/linux/man-pages/man8/pam_pwhistory.8.html)
- [Ansible community.general.pamd 모듈](https://docs.ansible.com/projects/ansible/latest/collections/community/general/pamd_module.html)
