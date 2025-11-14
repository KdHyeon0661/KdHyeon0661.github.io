---
layout: post
title: Linux - 사용자, 그룹, sudoers 고급 설정
date: 2024-11-08 19:20:23 +0900
category: Linux
---
# 사용자, 그룹, sudoers 고급 설정

## 사용자/그룹의 핵심 구조와 표준 파일

| 파일 | 역할 | 비고 |
|---|---|---|
| `/etc/passwd` | 사용자 기본 정보(UID, GID, 홈, 셸) | 해시 비밀번호는 `x`로 표기, 실제 해시는 `/etc/shadow` |
| `/etc/shadow` | 비밀번호 해시/만료·정책 | root만 읽기 |
| `/etc/group` | 그룹 목록/구성원 | 보조 그룹 관리 |
| `/etc/gshadow` | 그룹 비밀정보 | root만 읽기 |

### `/etc/passwd` 형식 리캡

```
name:passwd:UID:GID:gecos:home:shell
```
- `passwd` 필드가 `x`면 해시는 `/etc/shadow`에 존재.
- `shell`은 `/bin/bash`, `/usr/sbin/nologin` 등.

### UID/GID 범위 상식

- 0: root
- 1–999(배포판마다 다름): **시스템 사용자**(로그인 불가/데몬용)
- 1000+: 일반 사용자(데스크톱/서버 사용자 계정 기본 시작점)

---

## 계정 생성·수정·삭제: adduser vs useradd

- **Debian/Ubuntu**: `adduser`(대화형 래퍼) 권장, `useradd`(저수준)도 가능
- **RHEL/Fedora**: `useradd`가 일반적, `adduser`는 링크 또는 별도 패키지

### 신규 사용자 생성(안전 템플릿)

```bash
# Debian/Ubuntu

sudo adduser devkim --gecos "Dev Kim,,,"
# 또는 저수준

sudo useradd -m -c "Dev Kim" -s /bin/bash devkim && sudo passwd devkim

# RHEL/Fedora (기본)

sudo useradd -m -c "Dev Kim" -s /bin/bash devkim && sudo passwd devkim
```

옵션 요약:
- `-m`/`--create-home`: 홈 생성
- `-s`: 로그인 셸 지정
- `-c`/`--gecos`: 설명
- `-u`: UID 고정
- `-g`/`-G`: 주/보조 그룹 지정

### 스켈레톤(`/etc/skel`)과 기본값(`/etc/default/useradd`, `/etc/login.defs`)

```bash
sudo ls -la /etc/skel         # 신규 홈 디렉터리 템플릿(기본 dotfiles 등)
sudo grep -E '^(CREATE_HOME|UMASK)' /etc/login.defs
```
- `UMASK`, `CREATE_HOME`, UID/GID 시작 범위를 조직 표준에 맞게 조정 가능.

### 사용자 수정

```bash
# 보조 그룹 추가(기존 유지 필수 옵션 -a)

sudo usermod -aG developers devkim

# 주 그룹 변경

sudo usermod -g devs_primary devkim

# 홈/셸 변경

sudo usermod -d /home/dev.kim -m devkim
sudo chsh -s /bin/bash devkim
```

### 사용자 삭제

```bash
# Debian/Ubuntu

sudo deluser devkim
sudo deluser --remove-home devkim
sudo deluser --remove-all-files devkim  # 소유 파일 전부 제거(매우 주의)

# RHEL/Fedora

sudo userdel devkim
sudo userdel -r devkim                  # 홈/메일스풀 함께 제거
```

> 실전 권고: 실제 프로젝트/팀 디렉터리의 파일은 **chown/아카이빙** 후 계정 삭제.

---

## 그룹 관리와 협업 디렉터리 설계
### 그룹 생성/삭제/변경

```bash
sudo groupadd developers
sudo groupdel developers
sudo groupmod -n devs developers      # 그룹명 변경
```

### 구성원 관리

```bash
sudo usermod -aG devs devkim
groups devkim
id devkim
```

### 협업 디렉터리(그룹 상속 + 기본 권한)

```bash
sudo mkdir -p /srv/project
sudo chgrp -R devs /srv/project
sudo chmod 2775 /srv/project               # setgid 디렉터리(그룹 상속)
setfacl -m d:g:devs:rwx /srv/project       # 디폴트 ACL(신규 항목 권한 보장)
setfacl -m d:o::--- /srv/project
```

---

## 로그인 셸 전략과 시스템 사용자
### 로그인 금지 셸

```bash
# 로그인 불가 계정(데몬/시스템 사용자)에 권장

sudo chsh -s /usr/sbin/nologin nobody
# 또는 /bin/false

```
- `/usr/sbin/nologin`은 친절한 안내 메시지를 출력하고 종료.

### 시스템 사용자(서비스용)

```bash
# 홈 없음, 로그인 불가, 전용 UID 범위

sudo useradd --system --no-create-home --shell /usr/sbin/nologin mysvc
```

---

## 비밀번호 정책·만료·잠금(PAM과 연동)
### 비밀번호 설정/잠금

```bash
sudo passwd devkim
sudo passwd -l devkim      # 잠금(lock)
sudo passwd -u devkim      # 잠금 해제
sudo passwd -e devkim      # 다음 로그인 시 변경 강제
```

### 비밀번호/계정 만료(`chage`)

```bash
sudo chage -l devkim
sudo chage -M 90 -W 7 -I 14 devkim   # 최대 90일, 만료 7일 전 경고, 14일 비활성화
```

### 로그인 실패 잠금(`faillock` 등, 배포판별 PAM 설정 필요)

```bash
# 실패 횟수 확인

faillock --user devkim
# 잠금 해제

faillock --user devkim --reset
```

### 복잡도 정책(`pwquality` 예)

```bash
# /etc/security/pwquality.conf

minlen = 12
ucredit = -1
lcredit = -1
dcredit = -1
ocredit = -1
```

> PAM 스택(`/etc/pam.d/common-auth`, `system-auth` 등)은 배포판별로 상이. **보안 정책 변경은 사전 테스트 필수**.

---

## SSH 키·홈 권한 관례(안전한 기본)

```bash
# 소유자/권한

chmod 700 ~devkim/.ssh
chmod 600 ~devkim/.ssh/authorized_keys
chown -R devkim:devkim ~devkim/.ssh
```
- 홈 디렉터리는 보통 `0750` 또는 `0700` 권장(조직 정책에 따름).

---

## sudo 권한 부여와 검증
### sudo 사용 그룹에 사용자 추가

```bash
# Debian/Ubuntu

sudo usermod -aG sudo devkim
# RHEL/Fedora

sudo usermod -aG wheel devkim
```

### 가능한 명령 확인

```bash
sudo -l            # 현재 사용자
sudo -l -U devkim  # root만 가능: 타 사용자
```

---

## sudoers 편집 원칙과 드롭인
### visudo 필수

```bash
sudo visudo              # 기본 /etc/sudoers
sudo visudo -f /etc/sudoers.d/deploy
sudo visudo -c           # 구문 검사
```
- **직접 편집 금지**: `visudo`가 **경합/구문 오류**를 보호한다.
- **드롭인 권장**: `/etc/sudoers.d/*` 파일로 역할별 분리.

### sudoers 기본 항목 복습

```
root    ALL=(ALL:ALL) ALL
```
- `user  host_list = (runas_user:runas_group) options: command_list`

---

## sudoers 고급 문법: Alias, Defaults, 제한 패턴
### Alias

```text
User_Alias   ADMINS = devkim, ops1
Runas_Alias  WEB   = www-data, nginx
Host_Alias   DMZ   = web01, web02, web03
Cmnd_Alias   SYSCTL= /usr/sbin/sysctl, /usr/sbin/modprobe
```

### Defaults (글로벌/사용자/호스트/명령 단위)

```text
Defaults        env_reset,timestamp_timeout=10
Defaults:ADMINS secure_path=/usr/local/sbin:/usr/local/bin:/usr/sbin:/usr/bin:/sbin:/bin
Defaults:www    !authenticate
```
- `timestamp_timeout=0`: 매번 패스워드
- `lecture=always`: 최초 사용 시 경고 메시지
- `requiretty`(RHEL 전통): 터미널 필요(자동화와 충돌 가능)
- `env_keep+=HTTP_PROXY` 등 환경 변수 보존 제어
- `secure_path`: PATH 고정(하이재킹 방지)

### 최소권한 부여(절대경로, 와일드카드/쉘 허용 금지)

```text
# 특정 서비스 재시작만 허용(비밀번호 없음)

%deploy ALL=(root) NOPASSWD: /bin/systemctl restart myapp.service

# 웹 서버 로그만 tail 허용

%support ALL=(root) /usr/bin/tail -n * /var/log/nginx/*.log
```
> **위험 주의**: `tail -n *` 같은 와일드카드에서 `;`·`&`·백틱 등 **쉘 확장**으로 권한 상승 여지 없는지 검증. **더 안전한 방법은 스크립트 래퍼를 고정 경로로 제공**하고 인자를 제한하는 것.

### 환경/셸 탈출 방지 옵션

- `NOEXEC`: 동적 로더를 이용한 하위 실행 차단(완전하지 않음)
- `SETENV`/`!SETENV`: 환경 변경 허용/금지
- `sudoedit`: 안전한 편집(임시파일 + 소유권 유지), 에디터를 통한 권한 상승 트릭 줄이기

### 예시 모음

```text
# 전체 권한 (지양)

devkim ALL=(ALL) ALL

# 서비스 재시작 전용(비번 無)

%deploy ALL=(root) NOPASSWD: /bin/systemctl restart myapp

# 특정 유저로만 실행 허용

ops  ALL=(www-data) NOPASSWD: /usr/bin/ps, /usr/bin/kill, /usr/bin/ls

# 로그 조회만 허용

support ALL=(root) NOPASSWD: /usr/bin/journalctl -u myapp --since *@

# 편집은 sudoedit로만

Defaults!sudoedit secure_path=/usr/bin:/bin
devops ALL=(root) sudoedit /etc/myapp/*.conf
```

---

## 실전 플레이북

### 플레이북 A — 신규 개발자 온보딩(팀 표준)

```bash
# 사용자 생성 + 초기 비밀번호

sudo adduser devkim --gecos "Dev Kim,,,"
sudo passwd -e devkim                   # 최초 로그인 시 변경

# 그룹/협업 디렉터리

sudo groupadd -f devs
sudo usermod -aG devs devkim
sudo install -d -m 2775 -o root -g devs /srv/project
setfacl -m d:g:devs:rwx /srv/project

# sudo 최소 권한(배포 스크립트만)

sudo visudo -f /etc/sudoers.d/deploy
# 내용:
# %devs ALL=(root) NOPASSWD:/usr/local/bin/deploy_myapp

```

### 플레이북 B — 퇴사자 처리(계정 비활성→자산 이동→삭제)

```bash
# 즉시 잠금

sudo passwd -l devkim
sudo chage -E 0 devkim

# 파일 소유권 이전/아카이브(예시)

sudo find / -xdev -user devkim -print0 | sudo tar --null -cvf /root/devkim_backup.tar --files-from=-

# sudoers/그룹 정리(드롭인 파일 제거)

sudo rm -f /etc/sudoers.d/devkim*

# 일정 기간 보관 후 삭제

sudo userdel -r devkim
```

### 플레이북 C — 실패 로그인 시 잠금 정책

```bash
# faillock 확인

faillock --user devkim

# PAM 설정 조정 후(배포판 문서 준수), 오탐/오용 시 초기화

faillock --user devkim --reset
```

### 플레이북 D — 루트리스 컨테이너용 subordinate ID

```bash
# rootless podman/docker가 필요로 하는 subuid/subgid

echo "devkim:100000:65536" | sudo tee -a /etc/subuid
echo "devkim:100000:65536" | sudo tee -a /etc/subgid
```

---

## 감사/점검 유틸

```bash
last -n 20             # 최근 로그인 기록
lastlog                # 사용자별 마지막 로그인 요약
faillog -a             # 실패 기록 요약
sudo journalctl -t sudo -S "2025-11-01"
```
- **중요 파일 권한 점검**:
```bash
sudo chmod 640 /etc/shadow /etc/gshadow
sudo chmod 644 /etc/passwd /etc/group
sudo chown root:root /etc/{passwd,shadow,group,gshadow}
```

---

## 보안 체크리스트(요지)

- **최소 권한 원칙**: sudoers는 **역할별 드롭인 + 절대경로 + 인자 제한**
- **관리자 그룹 분리**: `wheel`/`sudo`/`devops` 등 **업무별 구분**
- **비밀번호 정책**: `pwquality`/`chage`로 **길이/이력/주기** 강제
- **로그인 실패 잠금**: `faillock` 등으로 공격 감속
- **SSH**: 키 기반, 홈/`.ssh` 권한 엄수, 필요 시 `AllowUsers`/`Match` 정책
- **시스템 사용자**: `/usr/sbin/nologin` 사용, `sudoedit`/`NOEXEC`로 셸 탈출 리스크 완화
- **협업 디렉터리**: **setgid+default ACL**로 일관 권한 유지
- **감사**: `sudo -l`, `journalctl -t sudo`, `last/lastlog`, 정기 점검 자동화

---

## 명령/파일 요약 표

| 주제 | 핵심 명령/파일 |
|---|---|
| 사용자 | `adduser`/`useradd`, `usermod`, `userdel`/`deluser`, `chsh` |
| 그룹 | `groupadd`/`groupdel`/`groupmod`, `gpasswd`, `id`, `groups` |
| 비밀번호 | `passwd`, `chage`, `faillock`, `/etc/shadow` |
| 홈/스켈레톤 | `/etc/skel`, `/etc/login.defs`, `/etc/default/useradd` |
| 협업 권한 | `chmod 2775`, `setfacl -m d:...`, `chgrp` |
| sudoers | `visudo`, `/etc/sudoers`, `/etc/sudoers.d/*`, `sudo -l` |
| 보안/감사 | `last`, `lastlog`, `journalctl -t sudo`, `faillog` |
| 컨테이너 | `/etc/subuid`, `/etc/subgid` |
