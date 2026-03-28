---
layout: post
title: Linux - 사용자, 그룹, sudoers 고급 설정
date: 2024-11-08 19:20:23 +0900
category: Linux
---
# 사용자, 그룹, sudoers 관리

## 사용자와 그룹의 기본 구조

리눅스에서 사용자와 그룹 정보는 몇 개의 핵심 파일에 저장된다. 이 파일들의 구조를 이해하는 것이 권한 관리의 첫걸음이다.

| 파일 | 역할 | 비고 |
|------|------|------|
| `/etc/passwd` | 사용자 기본 정보 (UID, GID, 홈, 셸) | 비밀번호는 `x`로 표시, 실제 해시는 `/etc/shadow` |
| `/etc/shadow` | 암호화된 비밀번호, 만료 정책 | root만 읽기 가능 |
| `/etc/group` | 그룹 목록과 구성원 | 보조 그룹 관리 |
| `/etc/gshadow` | 그룹 암호 및 관리자 | root만 읽기 가능 |

### `/etc/passwd` 한 줄의 의미

```
username:x:UID:GID:설명:홈디렉토리:로그인셸
```

- **UID**: 사용자 식별 번호. 0은 root, 1-999는 시스템 계정, 1000부터 일반 사용자
- **GID**: 사용자의 주 그룹 ID

---

## 계정 생성, 수정, 삭제

### 사용자 추가

Debian/Ubuntu 계열은 `adduser`를, RHEL/Fedora 계열은 `useradd`를 주로 사용한다.

```bash
# Debian/Ubuntu
sudo adduser devkim

# RHEL/Fedora (옵션 포함)
sudo useradd -m -c "Dev Kim" -s /bin/bash devkim
sudo passwd devkim
```

주요 옵션:
- `-m`: 홈 디렉터리 생성
- `-s`: 로그인 셸 지정
- `-c`: 사용자 설명 (GECOS)
- `-G`: 보조 그룹 추가

### 사용자 정보 수정

```bash
# 보조 그룹 추가 (기존 그룹 유지: -a)
sudo usermod -aG developers devkim

# 주 그룹 변경
sudo usermod -g devs devkim

# 로그인 셸 변경
sudo chsh -s /bin/bash devkim
```

### 사용자 삭제

```bash
# Debian/Ubuntu
sudo deluser --remove-home devkim

# RHEL/Fedora
sudo userdel -r devkim
```

`-r` 옵션은 홈 디렉터리와 메일 스풀을 함께 삭제한다.

---

## 그룹 관리와 협업 디렉터리

### 그룹 생성 및 구성원 관리

```bash
sudo groupadd developers
sudo usermod -aG developers devkim
groups devkim          # 사용자가 속한 그룹 확인
id devkim              # UID, GID, 그룹 정보
```

### 협업 디렉터리 설정

여러 사용자가 공동 작업할 디렉터리를 만들 때는 setgid 비트와 ACL을 활용한다.

```bash
sudo mkdir -p /srv/project
sudo chgrp -R developers /srv/project
sudo chmod 2775 /srv/project               # setgid: 새 파일이 그룹 소유권 상속
sudo setfacl -m d:g:developers:rwx /srv/project   # 기본 ACL
```

- setgid(`2`) : 디렉터리 안에서 생성되는 파일의 그룹을 디렉터리 그룹으로 상속
- default ACL(`d:g:...`) : 새로 생성되는 파일/디렉터리에 기본 권한 부여

---

## 비밀번호 정책과 계정 잠금

### 기본 비밀번호 관리

```bash
sudo passwd devkim               # 비밀번호 설정/변경
sudo passwd -l devkim            # 계정 잠금
sudo passwd -u devkim            # 잠금 해제
sudo passwd -e devkim            # 다음 로그인 시 비밀번호 변경 강제
```

### 비밀번호 만료 정책 (chage)

```bash
sudo chage -l devkim             # 현재 정책 확인
sudo chage -M 90 -W 7 -I 14 devkim
```
- `-M 90`: 최대 사용 기간 90일
- `-W 7`: 만료 7일 전 경고
- `-I 14`: 만료 후 14일간 비활성화 후 잠금

### 로그인 실패 잠금 (faillock)

```bash
sudo faillock --user devkim      # 실패 기록 확인
sudo faillock --user devkim --reset   # 잠금 해제
```

---

## SSH 키와 홈 디렉터리 권한

SSH 공개키 인증을 사용하려면 `.ssh` 디렉터리와 파일 권한이 올바르게 설정되어야 한다.

```bash
sudo mkdir ~devkim/.ssh
sudo chmod 700 ~devkim/.ssh
sudo touch ~devkim/.ssh/authorized_keys
sudo chmod 600 ~devkim/.ssh/authorized_keys
sudo chown -R devkim:devkim ~devkim/.ssh
```

- `.ssh` 디렉터리: `700` (소유자만 접근)
- `authorized_keys`: `600` (소유자만 읽기/쓰기)

---

## sudo 권한 부여와 sudoers

### sudo 그룹에 사용자 추가

배포판별 sudo 권한을 가진 그룹에 사용자를 추가한다.

```bash
# Debian/Ubuntu
sudo usermod -aG sudo devkim

# RHEL/Fedora
sudo usermod -aG wheel devkim
```

### sudo 권한 확인

```bash
sudo -l          # 현재 사용자의 sudo 권한
sudo -l -U devkim   # 다른 사용자 권한 확인 (root 필요)
```

### sudoers 파일 편집 (visudo)

`/etc/sudoers`는 반드시 `visudo`로 편집해야 한다. 구문 오류를 방지할 수 있다.

```bash
sudo visudo                     # 기본 파일 편집
sudo visudo -f /etc/sudoers.d/deploy   # 드롭인 파일 편집
sudo visudo -c                  # 구문 검사
```

**드롭인 방식**을 권장한다. `/etc/sudoers.d/` 아래에 역할별로 파일을 만들어 관리한다.

### 기본 sudoers 항목 구조

```
사용자   호스트=(실행사용자:실행그룹)   명령어목록
```

예:
```
root    ALL=(ALL:ALL) ALL
%wheel  ALL=(ALL) ALL
```

### 명령어 제한 (최소 권한)

```bash
# 특정 서비스 재시작만 허용 (비밀번호 없음)
%deploy ALL=(root) NOPASSWD: /bin/systemctl restart myapp.service

# 웹 서버 로그 조회만 허용
%support ALL=(root) /usr/bin/tail -n * /var/log/nginx/*.log
```

와일드카드(`*`) 사용 시에는 추가 명령어 삽입이 가능한지 주의해야 한다. 더 안전한 방법은 스크립트 래퍼를 만들어 사용하는 것이다.

### Alias와 Defaults (간단한 예)

```text
User_Alias ADMINS = devkim, ops1
Cmnd_Alias SERVICE_CMDS = /bin/systemctl restart nginx, /bin/systemctl reload nginx

ADMINS ALL=(root) NOPASSWD: SERVICE_CMDS
Defaults timestamp_timeout=10
```

---

## 실전 시나리오

### 신규 개발자 온보딩

```bash
# 사용자 생성
sudo adduser devkim
sudo passwd -e devkim    # 첫 로그인 시 비밀번호 변경 강제

# 그룹 추가 및 협업 디렉터리 설정
sudo groupadd developers
sudo usermod -aG developers devkim
sudo mkdir -p /srv/project
sudo chgrp -R developers /srv/project
sudo chmod 2775 /srv/project

# sudo 권한 (배포 스크립트만)
echo "devkim ALL=(root) NOPASSWD: /usr/local/bin/deploy_myapp" | sudo tee /etc/sudoers.d/deploy
```

### 퇴사자 처리

```bash
# 계정 즉시 잠금
sudo passwd -l devkim
sudo chage -E 0 devkim

# 사용자 파일 소유권 이전 또는 백업
sudo find / -xdev -user devkim -print0 | sudo tar --null -cvf /backup/devkim_files.tar --files-from=-

# sudoers 파일 정리
sudo rm -f /etc/sudoers.d/devkim*

# 이후 계정 삭제 (보관 기간 후)
# sudo userdel -r devkim
```

---

## 시스템 감사 기본

```bash
last -n 20                 # 최근 로그인 기록
lastlog                    # 모든 사용자의 마지막 로그인 시간
sudo journalctl -t sudo -S "2025-11-01"   # sudo 사용 기록
sudo faillock --user devkim                # 실패 로그인 기록
```

중요 파일 권한 점검:
```bash
sudo chmod 640 /etc/shadow /etc/gshadow
sudo chmod 644 /etc/passwd /etc/group
sudo chown root:root /etc/{passwd,shadow,group,gshadow}
```

---

## 핵심 명령어 요약

| 주제 | 명령어 / 파일 |
|------|---------------|
| 사용자 추가/삭제 | `adduser`, `useradd`, `userdel`, `deluser` |
| 사용자 정보 수정 | `usermod`, `chsh` |
| 그룹 관리 | `groupadd`, `groupmod`, `gpasswd`, `id`, `groups` |
| 비밀번호 정책 | `passwd`, `chage`, `faillock` |
| sudoers 관리 | `visudo`, `/etc/sudoers`, `/etc/sudoers.d/` |
| 협업 디렉터리 | `chmod 2775`, `setfacl`, `chgrp` |
| 감사 | `last`, `lastlog`, `journalctl -t sudo` |

## 결론

사용자와 그룹 관리는 리눅스 시스템 보안의 기본이다. 최소 권한 원칙에 따라 각 사용자에게 필요한 만큼만 권한을 부여하고, 그룹과 sudoers를 적절히 활용해야 한다.

협업 디렉터리에서는 setgid와 기본 ACL을 사용해 그룹 소유권과 권한이 자동으로 상속되도록 설정하는 것이 좋다. sudoers는 `visudo`로 편집하고, 드롭인 파일을 사용해 관리한다.

정기적으로 로그인 기록과 sudo 사용 내역을 점검하고, 퇴사자 처리 등 프로세스를 표준화하여 안전한 시스템 운영을 유지하자.