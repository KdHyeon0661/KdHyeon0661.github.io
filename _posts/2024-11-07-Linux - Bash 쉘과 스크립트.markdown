---
layout: post
title: Linux - Bash 쉘과 스크립트
date: 2024-11-07 19:20:23 +0900
category: Linux
---
# 리눅스 6편: Bash 쉘과 스크립트 - 환경 설정, 조건문, 반복문, 함수

## 🖥️ Bash란?

**Bash (Bourne Again SHell)** 는 리눅스에서 가장 널리 사용되는 기본 쉘입니다. 터미널에서 명령을 실행하거나 `.sh` 파일로 스크립트를 작성하여 자동화할 수 있습니다.

---

## ⚙️ 환경 설정 파일

쉘은 시작할 때 다양한 설정 파일을 읽습니다.

| 파일명         | 설명 |
|----------------|------|
| `~/.bashrc`    | 인터랙티브 셸용 설정 (alias, PS1 등) |
| `~/.bash_profile` | 로그인 시 실행되는 설정 |
| `/etc/profile` | 시스템 전체 로그인 설정 |
| `/etc/bash.bashrc` | 시스템 전체의 bashrc 설정 |

### 예시: `.bashrc`에서 alias 설정
```bash
alias ll='ls -alF'
alias gs='git status'
```

---

## 📦 변수 사용법

```bash
name="DoHyun"
echo "Hello, $name"
```

- 변수 할당 시 `=` 주변에 공백이 없어야 합니다.
- 변수는 `$변수명` 형태로 사용합니다.

### 내장 변수 예시
```bash
echo $HOME     # 사용자 홈 디렉토리
echo $USER     # 현재 사용자 이름
echo $PWD      # 현재 경로
```

---

## 📜 조건문 (if, else)

```bash
if [ 조건 ]; then
    # 참일 때 실행
else
    # 거짓일 때 실행
fi
```

### 🔍 문자열 비교 예시
```bash
name="kim"
if [ "$name" = "kim" ]; then
    echo "같은 이름입니다"
else
    echo "다른 이름입니다"
fi
```

### 🔢 숫자 비교 예시
```bash
a=10
b=20
if [ "$a" -lt "$b" ]; then
    echo "$a 는 $b 보다 작습니다"
fi
```

### 🚪 파일 존재 여부 확인
```bash
if [ -f "/etc/passwd" ]; then
    echo "파일 존재"
fi
```

| 조건식 | 의미 |
|--------|------|
| `-f`   | 일반 파일인지 확인 |
| `-d`   | 디렉토리인지 확인 |
| `-z`   | 문자열이 비었는지 |
| `-n`   | 문자열이 비어있지 않은지 |
| `-e`   | 존재 여부 확인 (파일/디렉토리) |

---

## 🔁 반복문 (for, while)

### `for` 문 예시
```bash
for i in 1 2 3; do
    echo "숫자: $i"
done
```

### 파일 목록 루프
```bash
for file in *.txt; do
    echo "$file 파일을 처리 중..."
done
```

### `while` 문 예시
```bash
count=1
while [ $count -le 5 ]; do
    echo "Count: $count"
    count=$((count+1))
done
```

---

## 🧰 함수 정의

```bash
greet() {
    echo "Hello, $1!"
}

greet DoHyun
```

- `$1`, `$2`는 인자
- `return`은 숫자만 반환 가능
- 함수 내 지역 변수는 `local` 키워드 사용

---

## 🔧 실행 가능한 스크립트 만들기

### 1. 파일 생성
```bash
nano hello.sh
```

### 2. 내용 입력
```bash
#!/bin/bash
echo "Hello, Linux World!"
```

### 3. 실행 권한 부여 및 실행
```bash
chmod +x hello.sh
./hello.sh
```

---

## 🧪 실전 예시

### 예시 1: 디렉토리 없으면 만들기
```bash
#!/bin/bash
dir="/tmp/myfolder"
if [ ! -d "$dir" ]; then
    mkdir "$dir"
    echo "디렉토리 생성됨: $dir"
else
    echo "이미 존재: $dir"
fi
```

### 예시 2: 로그 파일 날짜별로 백업
```bash
#!/bin/bash
today=$(date +%Y%m%d)
cp /var/log/syslog "/backup/syslog_$today"
```

### 예시 3: 스크립트 실행 결과 로그 파일에 저장
```bash
#!/bin/bash
echo "시작 시간: $(date)" >> script.log
uptime >> script.log
echo "종료 시간: $(date)" >> script.log
```

---

## 📎 요약 명령어 및 문법

| 문법/명령어 | 설명 |
|-------------|------|
| `alias`     | 명령어 단축어 설정 |
| `$변수`, `$(명령)` | 변수와 명령어 치환 |
| `if`, `else`| 조건 분기 |
| `[ -f FILE ]`, `[ -d DIR ]` | 파일 존재 여부 확인 |
| `for`, `while` | 반복문 |
| `function name()` | 함수 정의 |
| `chmod +x` | 실행 권한 부여 |

---

## ✨ 마무리

이번 편에서는 **Bash 쉘의 구조와 기본 스크립트 문법**을 정리했습니다. 자동화, 반복 작업, 환경 설정 등에서 스크립트는 필수적인 도구입니다.
