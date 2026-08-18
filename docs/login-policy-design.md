# 로그인 정책 설계 초안

이 문서는 아직 구현되지 않은 별도 `login_policy` Role의 설계 기준이다.
PAM 변경은 접속 장애 위험이 있으므로 OS별 현황수집과 단일 서버 파일럿 이후
구현·적용한다.

## 기능 분류

| 정책 | 구분 | 구현 또는 통제 방법 |
|---|---|---|
| 1인 1계정 | 일부 가능 | 선언된 계정 생성·중복 UID 점검, 실제 소유자는 IAM/HR 확인 |
| 비밀번호 본인 관리 | 절차 | Ansible은 비밀번호가 아니라 정책만 관리 |
| 길이와 문자 종류 | 자동 강제 | `pam_pwquality`, 최소 12자·2종류 초안 |
| Null 비밀번호 금지 | 자동 강제 | `PermitEmptyPasswords no`, shadow 점검 |
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
  ssh_max_auth_tries: 10
  permit_empty_passwords: false
```

## 예정 구조

```text
playbooks/11-apply-login-policy.yml
roles/login_policy/
├── defaults/main.yml
├── tasks/main.yml
├── tasks/redhat7.yml
├── tasks/redhat8plus.yml
├── tasks/ubuntu_legacy.yml
├── tasks/ubuntu_modern.yml
├── handlers/main.yml
└── templates/
    ├── pwquality.conf.j2
    └── faillock.conf.j2
```

RHEL 8 이상은 `authselect`, Ubuntu는 `pam-auth-update`, 구형 OS는 지원 가능한
PAM 모듈을 확인해야 한다. PAM 파일을 배포판 구분 없이 덮어쓰지 않는다.
