---
layout: post
title: Linux - 사용자, 그룹, sudoers 고급 설정
date: 2024-11-08 19:20:23 +0900
category: Linux
---
# 리눅스 7편: 사용자, 그룹, sudoers 고급 설정

## 👤 리눅스 사용자와 그룹 구조

리눅스는 멀티유저 OS로, 각 사용자 계정은 고유한 **UID(User ID)**를 가지며, 하나 이상의 **그룹(Group)**에 속할 수 있습니다.

| 파일 경로         | 역할                            |
|-------------------|---------------------------------|
| `/etc/passwd`     | 사용자 계정 정보 (ID, 홈, 셸 등) |
| `/etc/group`      | 그룹 정보                        |
| `/etc/shadow`     | 비밀번호 정보 (암호화됨)        |

---

## 🧑 사용자 계정 관리

### 🔧 사용자 추가
```bash
sudo adduser devkim
```

| 옵션        | 설명                     |
|-------------|--------------------------|
| `--home DIR`| 홈 디렉토리 직접 지정    |
| `--shell SH`| 기본 로그인 셸 지정      |
| `--uid NUM` | 사용자 UID 직접 지정     |

→ 기본적으로 `/home/devkim` 디렉토리 생성 및 동일한 이름의 그룹 생성

---

### 🔑 비밀번호 설정
```bash
sudo passwd devkim
```

| 옵션 | 설명                   |
|------|------------------------|
| 없음 | 사용자 비밀번호 설정   |
| `-l` | 계정 잠금              |
| `-u` | 계정 잠금 해제         |
| `-e` | 다음 로그인 시 변경 요구 |

---

### 🗑️ 사용자 삭제
```bash
sudo deluser devkim
sudo deluser --remove-home devkim
```

| 옵션             | 설명                         |
|------------------|------------------------------|
| `--remove-home`  | 홈 디렉토리도 함께 삭제      |
| `--remove-all-files` | 계정 소유 모든 파일 제거  |

---

## 🧑‍🤝‍🧑 그룹(Group) 관리

### ➕ 그룹 추가
```bash
sudo groupadd developers
```

| 옵션       | 설명                     |
|------------|--------------------------|
| `-g GID`   | 그룹 ID 지정              |
| `-r`       | 시스템 그룹으로 생성       |

---

### ➕ 사용자 그룹 추가 (보조 그룹)
```bash
sudo usermod -aG developers devkim
```

| 옵션   | 설명                           |
|--------|--------------------------------|
| `-a`   | 기존 그룹 유지 (필수!)         |
| `-G`   | 추가할 그룹 목록 (쉼표 구분)   |
| `-g`   | 주 그룹 변경                   |

> 💡 `usermod -G`만 쓰면 기존 보조 그룹이 덮어써집니다. 항상 `-aG`를 함께 써야 합니다.

---

### 🗑️ 그룹 삭제
```bash
sudo groupdel developers
```

> 사용 중인 그룹은 삭제되지 않으며, 해당 그룹을 먼저 비워야 합니다.

---

### 📄 그룹 확인
```bash
groups devkim
id devkim
```

---

## 🧷 사용자 로그인 셸 변경

```bash
sudo chsh -s /bin/bash devkim
```

| 옵션   | 설명                  |
|--------|-----------------------|
| `-s`   | 사용할 로그인 셸 지정 |
| `-l`   | 사용자 이름 지정      |

현재 로그인 셸은 `$SHELL`로 확인 가능합니다.

---

## 🔐 sudo 권한 부여

일반 사용자가 `sudo` 명령을 사용하려면, 적절한 **그룹(sudo, wheel 등)**에 속해 있어야 합니다.

### 📥 사용자에게 sudo 권한 부여
```bash
sudo usermod -aG sudo devkim     # Debian/Ubuntu
sudo usermod -aG wheel devkim    # CentOS/Fedora
```

---

### 🔍 현재 sudo 권한 확인
```bash
sudo -l
```

| 옵션 | 설명                    |
|------|-------------------------|
| 없음 | 현재 사용자 권한 목록 출력 |
| `-U username` | 다른 사용자 권한 보기 (root만 가능) |

---

## 🧾 /etc/sudoers 파일 설정

### ⚠️ 반드시 visudo로 수정할 것!
```bash
sudo visudo
```

- 구문 오류 방지 및 롤백 가능
- 기본 편집기는 vi 또는 nano

---

### 📄 기본 구성
```text
root    ALL=(ALL:ALL) ALL
```

| 항목      | 의미                                     |
|-----------|------------------------------------------|
| `user`    | sudo 권한 부여 대상 사용자 또는 그룹      |
| `ALL`     | 어떤 호스트에서든                        |
| `(ALL:ALL)` | 어떤 사용자로 실행하고 어떤 그룹을 소유하든 |
| `ALL`     | 모든 명령 실행 허용                       |

---

### ✅ 특정 사용자에게 전체 sudo 권한
```text
devkim  ALL=(ALL) ALL
```

### ✅ 특정 명령만 허용 (예: reboot)
```text
devkim  ALL=(ALL) NOPASSWD: /sbin/reboot
```

| 키워드     | 설명                           |
|------------|--------------------------------|
| `NOPASSWD:` | 비밀번호 없이 실행 허용        |
| `PASSWD:`   | 비밀번호 입력 필요 (기본값)    |

---

### ✅ 특정 그룹에게 sudo 허용
```text
%developers  ALL=(ALL) ALL
```

- `%`는 그룹 지정 의미

---

## 🧪 실전 예시

### 예시 1: 신규 개발자 devkim을 추가하고 개발 그룹에 넣기
```bash
sudo adduser devkim
sudo groupadd developers
sudo usermod -aG developers devkim
```

### 예시 2: devkim 계정에 sudo 없이 reboot만 허용
```bash
sudo visudo
```
```text
devkim ALL=(ALL) NOPASSWD: /sbin/reboot
```

### 예시 3: 현재 로그인한 사용자의 UID, GID 확인
```bash
id
```

### 예시 4: 현재 로그인 사용자의 홈 디렉토리 확인
```bash
echo $HOME
```

---

## 📎 명령어 요약

| 명령어                     | 설명                            |
|----------------------------|---------------------------------|
| `adduser`, `deluser`       | 사용자 생성/삭제               |
| `passwd`                   | 비밀번호 변경 및 잠금/해제     |
| `groupadd`, `groupdel`     | 그룹 생성/삭제                 |
| `usermod -aG`, `-g`        | 그룹 추가 및 변경              |
| `groups`, `id`             | 그룹 정보 및 UID/GID 확인      |
| `sudo`, `sudo -l`          | 관리자 권한 실행, 권한 목록    |
| `visudo`                   | sudoers 안전 편집기            |
| `chsh -s`, `echo $SHELL`   | 로그인 셸 변경 및 확인         |

---

## ✨ 마무리

이번 편에서는 리눅스 사용자와 그룹을 효율적으로 관리하는 방법, 그리고 안전하게 `sudo` 권한을 부여하는 방법까지 다뤘습니다.  
특히 `sudoers` 설정은 시스템 보안과 직결되므로 `visudo`로 조심히 다루는 것이 중요합니다.