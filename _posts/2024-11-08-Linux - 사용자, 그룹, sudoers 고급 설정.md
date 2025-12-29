---
layout: post
title: Linux - 사용자, 그룹, sudoers 고급 설정
date: 2024-11-08 19:20:23 +0900
category: Linux
---
# 사용자, 그룹, sudoers 고급 설정

## 사용자와 그룹의 핵심 구조

리눅스 시스템에서 사용자와 그룹 관리는 권한 제어와 시스템 보안의 기초를 형성합니다. 관련 정보는 몇 가지 주요 파일에 저장되어 있으며, 이 파일들의 구조를 이해하는 것이 중요합니다.

| 파일 | 역할 | 비고 |
|---|---|---|
| `/etc/passwd` | 사용자 기본 정보(UID, GID, 홈 디렉터리, 로그인 셸) 저장 | 비밀번호 필드는 `x`로 표시되며, 실제 해시는 `/etc/shadow`에 있음 |
| `/etc/shadow` | 암호화된 비밀번호 해시와 계정 만료, 정책 정보 저장 | root 사용자만 읽기 가능한 보안 파일 |
| `/etc/group` | 그룹 목록과 각 그룹의 구성원 정보 저장 | 보조 그룹 관리에 사용 |
| `/etc/gshadow` | 그룹 암호 및 관리자 정보 저장 | root 사용자만 읽기 가능 |

### `/etc/passwd` 파일 형식

각 라인은 다음과 같은 형식을 가집니다:
```
username:password:UID:GID:GECOS:home_directory:login_shell
```
- **username**: 사용자 로그인 이름
- **password**: 과거에는 암호화된 비밀번호가 저장되었으나, 현대 시스템에서는 보안을 위해 `x`로 표시되고 실제 해시는 `/etc/shadow`에 저장됨
- **UID**: 사용자 ID (숫자)
- **GID**: 주 그룹 ID (숫자)
- **GECOS**: 사용자에 대한 추가 정보 (전체 이름, 사무실 번호 등)
- **home_directory**: 사용자의 홈 디렉터리 경로
- **login_shell**: 사용자가 로그인 시 실행되는 셸 프로그램

### UID/GID 범위 이해

- **0**: root 사용자 (최고 관리자 권한)
- **1–999** (배포판에 따라 다름): 시스템 사용자로, 주로 데몬이나 서비스 전용으로 사용되며 일반적인 로그인이 불가능함
- **1000 이상**: 일반 사용자 계정으로, 데스크톱이나 서버에서 생성하는 사용자 계정의 기본 시작점

---

## 계정 생성, 수정, 삭제

### adduser vs useradd

리눅스 배포판에는 사용자 생성에 두 가지 주요 도구가 있습니다:

- **Debian/Ubuntu 계열**: `adduser` 명령어를 권장합니다. 이는 대화형으로 사용자 생성 과정을 안내하는 고수준 래퍼 스크립트입니다. `useradd`도 사용 가능하지만 더 저수준의 도구입니다.
- **RHEL/Fedora 계열**: `useradd` 명령어가 표준이며, `adduser`는 종종 `useradd`에 대한 심볼릭 링크이거나 별도 패키지로 제공됩니다.

### 신규 사용자 생성 (안전한 템플릿)

```bash
# Debian/Ubuntu (adduser 권장)
sudo adduser devkim --gecos "Dev Kim,,,"

# Debian/Ubuntu (useradd 사용)
sudo useradd -m -c "Dev Kim" -s /bin/bash devkim && sudo passwd devkim

# RHEL/Fedora (useradd 사용)
sudo useradd -m -c "Dev Kim" -s /bin/bash devkim && sudo passwd devkim
```

주요 옵션 설명:
- `-m` 또는 `--create-home`: 사용자 홈 디렉터리를 생성합니다.
- `-s`: 사용자의 로그인 셸을 지정합니다.
- `-c` 또는 `--gecos`: 사용자에 대한 설명 정보를 설정합니다.
- `-u`: 특정 UID를 고정적으로 할당합니다.
- `-g`: 사용자의 주 그룹을 지정합니다.
- `-G`: 사용자를 추가 그룹(보조 그룹)에 추가합니다.

### 스켈레톤 디렉터리와 기본 설정

```bash
# 신규 사용자 홈 디렉터리의 템플릿 확인
sudo ls -la /etc/skel

# 시스템 기본 사용자 생성 설정 확인
sudo grep -E '^(CREATE_HOME|UMASK)' /etc/login.defs
```
- `/etc/skel` 디렉터리에는 새 사용자 생성 시 홈 디렉터리에 복사될 기본 파일들(주로 숨김 설정 파일들)이 포함되어 있습니다.
- `/etc/login.defs` 파일은 UID/GID 범위, UMASK 값, 홈 디렉터리 생성 정책 등 조직의 표준에 맞게 조정할 수 있는 시스템 전반의 설정을 포함합니다.

### 사용자 정보 수정

```bash
# 보조 그룹 추가 (기존 그룹 유지 필수: -a 옵션)
sudo usermod -aG developers devkim

# 주 그룹 변경
sudo usermod -g devs_primary devkim

# 홈 디렉터리 변경 (기존 홈 디렉터리 내용 이동: -m 옵션)
sudo usermod -d /home/dev.kim -m devkim

# 로그인 셸 변경
sudo chsh -s /bin/bash devkim
```

### 사용자 삭제

```bash
# Debian/Ubuntu (deluser 사용)
sudo deluser devkim                    # 계정만 삭제
sudo deluser --remove-home devkim      # 계정과 홈 디렉터리 삭제
sudo deluser --remove-all-files devkim # 계정과 소유한 모든 파일 삭제 (매우 주의 필요)

# RHEL/Fedora (userdel 사용)
sudo userdel devkim                    # 계정만 삭제
sudo userdel -r devkim                 # 계정과 홈 디렉터리, 메일 스풀 함께 삭제
```

**실전 권고사항**: 실제 프로젝트나 팀 디렉터리에 있는 파일은 먼저 소유권을 변경하거나 아카이브한 후에 계정을 삭제하는 것이 안전합니다. 사용자 삭제는 신중하게 접근해야 하는 작업입니다.

---

## 그룹 관리와 협업 디렉터리 설계

효율적인 협업을 위해서는 그룹 기반의 권한 관리가 필수적입니다.

### 그룹 생성, 삭제, 변경

```bash
sudo groupadd developers          # 새 그룹 생성
sudo groupdel developers          # 그룹 삭제 (그룹이 비어 있어야 함)
sudo groupmod -n devs developers  # 그룹 이름 변경 (developers → devs)
```

### 그룹 구성원 관리

```bash
sudo usermod -aG devs devkim  # 사용자를 그룹에 추가
groups devkim                  # 사용자가 속한 그룹 목록 확인
id devkim                      # 사용자의 UID, GID, 그룹 정보 상세 확인
```

### 협업 디렉터리 설정 (그룹 상속 + 기본 권한)

여러 사용자가 안전하게 공동 작업할 수 있는 디렉터리를 설정하는 방법입니다:

```bash
# 협업 디렉터리 생성
sudo mkdir -p /srv/project

# 디렉터리 소유 그룹 설정
sudo chgrp -R devs /srv/project

# setgid 비트 설정 (이 디렉터리 안에서 생성된 파일이 그룹 소유권 상속)
sudo chmod 2775 /srv/project

# 디폴트 ACL 설정 (신규 파일/디렉터리에 대한 기본 권한 보장)
sudo setfacl -m d:g:devs:rwx /srv/project
sudo setfacl -m d:o::--- /srv/project
```

---

## 로그인 셸 전략과 시스템 사용자

### 로그인 불가 계정 (서비스/데몬용)

일부 계정은 실제 로그인을 위한 것이 아니라 시스템 서비스나 데몬 전용으로 사용됩니다. 이러한 계정에는 로그인을 방지하는 셸을 지정해야 합니다.

```bash
# 로그인 불가 계정에 nologin 셸 설정
sudo chsh -s /usr/sbin/nologin nobody

# 또는 /bin/false 사용 (더 간단한 접근 방식)
sudo chsh -s /bin/false daemonuser
```
- `/usr/sbin/nologin`은 사용자가 로그인을 시도할 때 친절한 안내 메시지를 출력한 후 종료합니다.
- `/bin/false`는 아무 작업도 수행하지 않고 즉시 종료합니다.

### 시스템 사용자 생성 (서비스 전용)

시스템 서비스나 애플리케이션을 실행하기 위한 전용 사용자를 생성할 때는 다음과 같은 접근 방식을 사용합니다:

```bash
# 시스템 사용자 생성: 홈 디렉터리 없음, 로그인 불가, 시스템 UID 범위 사용
sudo useradd --system --no-create-home --shell /usr/sbin/nologin mysvc
```

---

## 비밀번호 정책과 계정 만료 관리

### 비밀번호 설정 및 계정 잠금

```bash
sudo passwd devkim        # 비밀번호 설정/변경
sudo passwd -l devkim     # 계정 잠금 (lock)
sudo passwd -u devkim     # 계정 잠금 해제 (unlock)
sudo passwd -e devkim     # 다음 로그인 시 비밀번호 변경 강제
```

### 비밀번호 및 계정 만료 정책 (`chage`)

`chage` 명령어는 비밀번호 만료 정책을 세밀하게 제어할 수 있습니다.

```bash
sudo chage -l devkim                    # 현재 설정 확인
sudo chage -M 90 -W 7 -I 14 devkim      # 정책 설정 예시
```
위 예시에서:
- `-M 90`: 비밀번호 최대 사용 기간을 90일로 설정
- `-W 7`: 만료 7일 전부터 경고 메시지 표시
- `-I 14`: 계정이 비활성화된 후 14일 후에 완전히 잠금

### 로그인 실패 잠금 (faillock)

반복된 로그인 실패 시 계정을 일시적으로 잠금하는 정책은 무차별 대입 공격(brute-force attacks)으로부터 시스템을 보호합니다.

```bash
# 특정 사용자의 실패 기록 확인
sudo faillock --user devkim

# 특정 사용자의 실패 기록 초기화 (잠금 해제)
sudo faillock --user devkim --reset
```

### 비밀번호 복잡도 정책 (pwquality)

비밀번호의 강도를 보장하기 위해 복잡도 요구사항을 설정할 수 있습니다.

```bash
# /etc/security/pwquality.conf 설정 예시
minlen = 12     # 최소 길이: 12자
ucredit = -1    # 최소 1개의 대문자 필수
lcredit = -1    # 최소 1개의 소문자 필수
dcredit = -1    # 최소 1개의 숫자 필수
ocredit = -1    # 최소 1개의 특수문자 필수
```

**중요 참고사항**: PAM(Pluggable Authentication Modules) 설정은 배포판마다 상이합니다 (`/etc/pam.d/common-auth`, `/etc/pam.d/system-auth` 등). 보안 정책 변경은 반드시 테스트 환경에서 먼저 검증한 후 프로덕션에 적용해야 합니다.

---

## SSH 키와 홈 디렉터리 권한 관리

SSH 공개키 인증을 안전하게 사용하기 위해서는 적절한 파일 권한이 필수적입니다.

```bash
# SSH 디렉터리 및 파일 권한 설정
sudo chmod 700 ~devkim/.ssh                    # 디렉터리는 소유자만 읽기/쓰기/실행 가능
sudo chmod 600 ~devkim/.ssh/authorized_keys    # authorized_keys 파일은 소유자만 읽기/쓰기 가능
sudo chown -R devkim:devkim ~devkim/.ssh       # 소유권 설정
```
일반적으로 홈 디렉터리는 `0750` (소유자: 모든 권한, 그룹: 읽기/실행, 기타: 권한 없음) 또는 `0700` (소유자만 모든 권한)으로 설정하는 것이 보안상 권장됩니다. 조직의 보안 정책에 따라 적절한 설정을 선택하세요.

---

## sudo 권한 부여와 검증

### sudo 사용 그룹에 사용자 추가

```bash
# Debian/Ubuntu 계열: sudo 그룹에 사용자 추가
sudo usermod -aG sudo devkim

# RHEL/Fedora 계열: wheel 그룹에 사용자 추가
sudo usermod -aG wheel devkim
```

### sudo 권한 확인

```bash
sudo -l                    # 현재 사용자의 sudo 권한 확인
sudo -l -U devkim          # 다른 사용자의 sudo 권한 확인 (root만 실행 가능)
```

---

## sudoers 파일 편집 원칙과 드롭인 방식

### visudo 사용의 중요성

```bash
sudo visudo                 # 기본 /etc/sudoers 파일 편집
sudo visudo -f /etc/sudoers.d/deploy  # 드롭인 파일 편집
sudo visudo -c              # sudoers 파일 구문 검사
```
**중요**: `/etc/sudoers` 파일을 직접 편집기로 열어 수정하지 마세요. `visudo`는 다음과 같은 중요한 보호 기능을 제공합니다:
1. **편집 경합 방지**: 동시에 여러 사람이 수정하는 것을 방지합니다.
2. **구문 검사**: 저장하기 전에 구문 오류를 검사하여 잘못된 설정으로 인한 시스템 접근 불가를 방지합니다.
3. **임시 파일 사용**: 직접 원본 파일을 수정하지 않고 임시 파일에서 작업합니다.

**드롭인 방식 권장**: `/etc/sudoers.d/` 디렉터리에 역할별로 작은 설정 파일을 생성하는 방식이 관리와 문제 해결에 더 용이합니다.

### sudoers 기본 항목 구조 복습

```
root    ALL=(ALL:ALL) ALL
```
이 항목은 다음과 같은 구조를 가집니다:
- `사용자 호스트=(실행사용자:실행그룹) 명령어목록`

---

## sudoers 고급 문법: Alias, Defaults, 제한 패턴

### Alias 정의

sudoers 파일에서 반복되는 사용자, 호스트, 명령어 그룹을 Alias로 정의하면 관리가 용이해집니다.

```text
User_Alias   ADMINS = devkim, ops1, admin2
Runas_Alias  WEB_USERS = www-data, nginx, apache
Host_Alias   DMZ_SERVERS = web01, web02, web03
Cmnd_Alias   SYSTEM_CMDS = /usr/sbin/sysctl, /usr/sbin/modprobe, /usr/bin/systemctl
```

### Defaults 지시어 (글로벌/사용자/호스트/명령 단위)

Defaults 지시어는 sudo의 전역 동작 방식을 제어합니다.

```text
Defaults        env_reset,timestamp_timeout=10
Defaults:ADMINS secure_path=/usr/local/sbin:/usr/local/bin:/usr/sbin:/usr/bin:/sbin:/bin
Defaults:www    !authenticate
```

주요 옵션:
- `timestamp_timeout=0`: 매번 sudo 명령 실행 시 패스워드 요구
- `timestamp_timeout=10`: 10분 동안 패스워드 기억
- `lecture=always`: sudo를 처음 사용할 때 경고 메시지 표시
- `requiretty`: 터미널에서만 sudo 실행 허용 (자동화 스크립트와 충돌 가능성 있음)
- `env_keep+=HTTP_PROXY`: 특정 환경 변수 보존
- `secure_path`: sudo 실행 시 사용할 PATH 고정 (PATH 하이재킹 방지)

### 최소 권한 원칙에 따른 명령어 제한

가능한 한 최소한의 권한만 부여하는 것이 보안상 중요합니다.

```text
# 특정 서비스 재시작만 허용 (비밀번호 없음)
%deploy ALL=(root) NOPASSWD: /bin/systemctl restart myapp.service

# 웹 서버 로그 조회만 허용
%support ALL=(root) /usr/bin/tail -n * /var/log/nginx/*.log
```

**보안 주의사항**: 와일드카드(`*`) 사용 시 주의가 필요합니다. 예를 들어 `tail -n *`에서 사용자가 `;`나 `&`, 백틱(`` ` ``) 등을 이용하여 추가 명령을 실행할 수 있는지 검증해야 합니다. 더 안전한 방법은 인자를 제한하는 스크립트 래퍼를 고정 경로로 제공하고 해당 스크립트만 실행 권한을 부여하는 것입니다.

### 환경 및 셸 탈출 방지 옵션

- `NOEXEC`: 동적 로더를 통한 하위 프로세스 실행 방지 (완벽하지는 않음)
- `SETENV`/`!SETENV`: 환경 변수 변경 허용/금지
- `sudoedit`: 안전한 파일 편집 모드 (임시 파일 사용 및 소유권 유지). 에디터를 통한 권한 상승 공격 가능성을 줄입니다.

### sudoers 설정 예시 모음

```text
# 전체 권한 부여 (가능한 한 피해야 할 설정)
devkim ALL=(ALL) ALL

# 서비스 재시작 전용 권한 (비밀번호 없음)
%deploy ALL=(root) NOPASSWD: /bin/systemctl restart myapp

# 특정 사용자로만 명령 실행 허용
ops  ALL=(www-data) NOPASSWD: /usr/bin/ps, /usr/bin/kill, /usr/bin/ls

# 로그 조회만 허용
support ALL=(root) NOPASSWD: /usr/bin/journalctl -u myapp --since *@

# 설정 파일 편집은 sudoedit로만 허용
Defaults!sudoedit secure_path=/usr/bin:/bin
devops ALL=(root) sudoedit /etc/myapp/*.conf
```

---

## 실전 시나리오 플레이북

### 플레이북 A — 신규 개발자 온보딩 (팀 표준 절차)

```bash
# 1. 사용자 생성 및 초기 비밀번호 설정
sudo adduser devkim --gecos "Dev Kim,,,"
sudo passwd -e devkim  # 최초 로그인 시 비밀번호 변경 강제

# 2. 그룹 생성 및 사용자 추가
sudo groupadd -f devs
sudo usermod -aG devs devkim

# 3. 협업 디렉터리 설정
sudo install -d -m 2775 -o root -g devs /srv/project
sudo setfacl -m d:g:devs:rwx /srv/project

# 4. 최소 권한 sudo 설정 (배포 스크립트 실행만 허용)
sudo visudo -f /etc/sudoers.d/deploy
```
`/etc/sudoers.d/deploy` 파일 내용 예시:
```text
# 배포 스크립트 실행만 허용 (비밀번호 없음)
devkim ALL=(root) NOPASSWD: /usr/local/bin/deploy_myapp
```

### 플레이북 B — 퇴사자 처리 절차 (계정 비활성화 → 자산 이전 → 삭제)

```bash
# 1. 즉시 계정 잠금
sudo passwd -l devkim          # 계정 잠금
sudo chage -E 0 devkim         # 계정 만료일을 과거로 설정

# 2. 사용자 파일 소유권 이전 및 아카이브
sudo find / -xdev -user devkim -print0 | sudo tar --null -cvf /root/devkim_backup.tar --files-from=-

# 3. sudoers 정리 (개별 드롭인 파일 제거)
sudo rm -f /etc/sudoers.d/devkim*

# 4. 일정 기간 보관 후 계정 삭제 (필요시)
# sudo userdel -r devkim
```

### 플레이북 C — 로그인 실패 잠금 정책 관리

```bash
# 실패 로그인 시도 기록 확인
sudo faillock --user devkim

# 특정 사용자의 실패 기록 초기화 (잠금 해제)
sudo faillock --user devkim --reset
```

### 플레이북 D — 루트리스 컨테이너를 위한 subordinate ID 설정

Podman이나 Docker의 루트리스 모드를 사용하려면 사용자에게 subordinate UID/GID 범위를 할당해야 합니다.

```bash
# 사용자에게 65536개의 UID/GID 범위 할당 (시작: 100000)
echo "devkim:100000:65536" | sudo tee -a /etc/subuid
echo "devkim:100000:65536" | sudo tee -a /etc/subgid
```

---

## 시스템 감사와 점검 도구

정기적인 시스템 감사는 보안 유지에 필수적입니다.

```bash
# 최근 로그인 기록 확인
last -n 20                # 최근 20개의 로그인 기록
lastlog                   # 모든 사용자의 마지막 로그인 시간 요약

# 실패한 로그인 시도 기록
sudo faillog -a           # 모든 사용자의 실패 로그인 기록 요약

# sudo 사용 기록 조회
sudo journalctl -t sudo -S "2025-11-01"  # 2025년 11월 1일 이후의 sudo 사용 기록
```

### 중요 시스템 파일 권한 점검

시스템 보안을 위해 중요한 파일들의 권한이 적절히 설정되었는지 정기적으로 점검해야 합니다.

```bash
# 중요한 인증 파일 권한 점검 및 설정
sudo chmod 640 /etc/shadow /etc/gshadow
sudo chmod 644 /etc/passwd /etc/group
sudo chown root:root /etc/{passwd,shadow,group,gshadow}
```

---

## 명령어 및 파일 요약

| 주제 | 핵심 명령어/파일 |
|---|---|
| 사용자 관리 | `adduser`/`useradd`, `usermod`, `userdel`/`deluser`, `chsh` |
| 그룹 관리 | `groupadd`/`groupdel`/`groupmod`, `gpasswd`, `id`, `groups` |
| 비밀번호 관리 | `passwd`, `chage`, `faillock`, `/etc/shadow` |
| 홈 디렉터리 템플릿 | `/etc/skel`, `/etc/login.defs`, `/etc/default/useradd` |
| 협업 디렉터리 권한 | `chmod 2775` (setgid), `setfacl -m d:...` (기본 ACL), `chgrp` |
| sudoers 관리 | `visudo`, `/etc/sudoers`, `/etc/sudoers.d/*`, `sudo -l` |
| 시스템 감사 | `last`, `lastlog`, `journalctl -t sudo`, `faillog` |
| 컨테이너 지원 | `/etc/subuid`, `/etc/subgid` (루트리스 컨테이너용) |

---

## 결론

리눅스 시스템에서 사용자, 그룹, sudoers 관리는 시스템 보안과 효율적인 협업의 핵심 요소입니다. 이 문서에서는 이러한 요소들을 체계적으로 관리하기 위한 실용적인 방법들을 살펴보았습니다.

효율적인 사용자 관리의 핵심은 **최소 권한 원칙**을 준수하는 것입니다. 각 사용자와 서비스는 자신의 역할을 수행하는 데 필요한 최소한의 권한만을 가져야 합니다. sudoers 설정에서는 역할별 드롭인 파일을 사용하고, 가능한 한 절대 경로와 제한된 인자를 지정하는 것이 좋습니다.

또한 **정기적인 감사와 점검**이 중요합니다. `last`, `lastlog`, `journalctl` 등을 활용하여 시스템 접근 패턴을 모니터링하고, 비정상적인 활동을 조기에 발견할 수 있어야 합니다.

마지막으로, **자동화와 문서화**를 통해 사용자 관리 프로세스를 표준화하는 것이 장기적으로 시스템 안정성과 보안을 유지하는 데 도움이 됩니다. 신규 사용자 온보딩, 퇴사자 처리, 권한 변경 등 모든 절차가 명확히 정의되고 가능한 한 자동화되어야 합니다.

이러한 원칙과 도구들을 잘 활용한다면, 복잡한 다중 사용자 환경에서도 안전하고 효율적인 리눅스 시스템을 운영할 수 있을 것입니다.