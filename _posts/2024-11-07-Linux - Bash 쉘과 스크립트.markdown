---
layout: post
title: Linux - Bash 쉘과 스크립트
date: 2024-11-07 19:20:23 +0900
category: Linux
---
# 리눅스 6편: Bash 쉘과 스크립트 - 환경 설정, 조건문, 반복문, 함수, 리디렉션

## 🖥️ Bash란?

**Bash (Bourne Again SHell)** 는 리눅스에서 가장 널리 사용되는 기본 쉘입니다. 터미널에서 명령을 실행하거나 `.sh` 파일로 스크립트를 작성하여 자동화할 수 있습니다.

---

## ⚙️ 환경 설정 파일

쉘은 시작할 때 다양한 설정 파일을 읽습니다.

| 파일명             | 설명 |
|--------------------|------|
| `~/.bashrc`        | 인터랙티브 셸용 설정 (alias, PS1 등) |
| `~/.bash_profile`  | 로그인 시 실행되는 설정 |
| `/etc/profile`     | 시스템 전체 로그인 설정 |
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

- 변수 할당 시 `=` 주변에 공백이 없어야 함
- 변수는 `$변수명` 또는 `"${변수명}"`으로 사용

### 내장 변수 예시
```bash
echo $HOME     # 사용자 홈 디렉토리
echo $USER     # 현재 사용자 이름
echo $PWD      # 현재 디렉토리
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

### 문자열 비교
```bash
if [ "$name" = "kim" ]; then
    echo "같은 이름입니다"
fi
```

### 숫자 비교
```bash
if [ "$a" -lt "$b" ]; then
    echo "$a 는 $b 보다 작습니다"
fi
```

### 파일 조건

| 조건식 | 의미 |
|--------|------|
| `-f`   | 일반 파일인지 확인 |
| `-d`   | 디렉토리인지 확인 |
| `-z`   | 문자열이 비었는지 |
| `-n`   | 문자열이 비어있지 않은지 |
| `-e`   | 존재 여부 확인 (파일/디렉토리) |

---

## 🔁 반복문 (for, while)

### `for` 문
```bash
for i in 1 2 3; do
    echo "숫자: $i"
done
```

### 파일 목록 반복
```bash
for file in *.txt; do
    echo "$file 처리 중..."
done
```

### `while` 문
```bash
count=1
while [ $count -le 3 ]; do
    echo "$count"
    ((count++))
done
```

---

## 🔄 case 문

```bash
read -p "확장자 입력: " ext
case "$ext" in
  txt) echo "텍스트 파일입니다" ;;
  jpg|png) echo "이미지 파일입니다" ;;
  *) echo "알 수 없는 형식입니다" ;;
esac
```

---

## 🧰 함수 정의

```bash
greet() {
    echo "Hello, $1!"
}

greet DoHyun
```

- `$1`, `$2` 등으로 인자 사용
- 지역 변수는 `local` 키워드 사용

---

## 🧾 사용자 입력 받기

```bash
read -p "이름을 입력하세요: " name
echo "입력한 이름: $name"
```

---

## 📤 입출력 리디렉션

| 기호 | 설명 |
|------|------|
| `>`   | 표준 출력 → 파일 (덮어쓰기) |
| `>>`  | 표준 출력 → 파일 (추가) |
| `<`   | 파일 → 표준 입력 |
| `2>`  | 표준 에러 → 파일 |
| `&>`  | 표준 출력 + 에러 → 파일 |

### 예시
```bash
ls > result.txt             # 정상 출력 저장
ls notfound 2> error.log    # 에러만 저장
ls * &> all.log             # 출력 + 에러 모두 저장
```

---

## 📑 Here Document (`<<EOF`)

여러 줄 텍스트를 한 번에 처리할 때 유용

### 텍스트 출력
```bash
cat <<EOF
리눅스 공부는 재미있습니다.
쉘 스크립트는 강력합니다.
EOF
```

### 설정 파일 자동 생성
```bash
cat <<EOF > config.ini
[setting]
enabled=true
port=8080
EOF
```

> `<<-EOF` 를 사용하면 들여쓰기 탭도 무시 가능

---

## 🔧 실행 가능한 스크립트 만들기

### 1. 스크립트 생성
```bash
nano hello.sh
```

### 2. 스크립트 작성
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

### ✅ 디렉토리 없으면 생성
```bash
#!/bin/bash
dir="/tmp/mydir"
if [ ! -d "$dir" ]; then
  mkdir "$dir"
  echo "디렉토리 생성됨: $dir"
fi
```

---

### ✅ 날짜 기반 로그 백업
```bash
#!/bin/bash
today=$(date +%F)
cp /var/log/syslog "/backup/syslog_$today"
```

---

### ✅ 결과 로그 파일 저장
```bash
#!/bin/bash
echo "시작 시간: $(date)" >> run.log
uptime >> run.log
echo "종료 시간: $(date)" >> run.log
```

---

## 📎 요약 명령어 및 문법

| 항목 | 설명 |
|------|------|
| `alias`, `export`, `read` | 환경 설정, 사용자 입력 |
| `$변수`, `$(명령)` | 변수와 명령어 치환 |
| `[ -f FILE ]`, `[ -d DIR ]` | 파일 존재 여부 확인 |
| `if`, `case`, `for`, `while` | 조건과 반복 제어 |
| `function`, `local`, `$1` | 함수 정의 및 인자 |
| `>`, `>>`, `2>`, `&>` | 입출력 리디렉션 |
| `<<EOF` | 여러 줄 입력 블록 |
| `chmod +x script.sh` | 실행 권한 부여 |

---

## ✨ 마무리

이제 Bash 스크립트로 다양한 작업을 자동화할 수 있습니다.  
단순한 반복 작업부터 조건 분기, 사용자 입력, 로그 저장까지 익히면,  
**시스템 자동화와 DevOps의 기초를 완전히 갖춘 것**이나 다름없습니다.
