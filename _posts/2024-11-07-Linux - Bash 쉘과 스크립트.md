---
layout: post
title: Linux - Bash 쉘과 스크립트
date: 2024-11-07 19:20:23 +0900
category: Linux
---
# Bash 쉘과 스크립트

## Bash란 무엇인가

Bash(Bourne Again SHell)는 리눅스에서 가장 널리 사용되는 셸이다. 명령 실행과 스크립트 작성을 모두 지원하며, GNU 유틸리티와 결합해 강력한 자동화 환경을 제공한다.

## 셸의 종류와 시작 모드

셸은 실행 방식에 따라 세 가지 모드로 구분된다.

* **로그인 셸**: 콘솔이나 SSH로 처음 로그인할 때 실행된다.
* **인터랙티브 셸**: 사용자가 프롬프트와 상호작용하는 모드이다.
* **논인터랙티브 셸**: 스크립트 실행이나 cron처럼 상호작용 없이 실행된다.

시작 파일의 로딩 순서는 다음과 같다.

| 모드 | 읽는 파일 |
|------|----------|
| 로그인 셸 | `/etc/profile` → `~/.bash_profile` (또는 `~/.bash_login`, `~/.profile`) |
| 인터랙티브 비로그인 셸 | `~/.bashrc` |
| 논인터랙티브 셸 | `BASH_ENV` 환경변수에 지정된 파일 |

## 환경 설정 파일

주요 설정 파일과 용도는 다음과 같다.

| 파일 | 대상 | 주요 용도 |
|------|------|----------|
| `/etc/profile` | 시스템 전역 로그인 | PATH, locale 등 시스템 기본 설정 |
| `~/.bash_profile` | 사용자 로그인 | 사용자별 환경 변수, 초기화 |
| `~/.bashrc` | 사용자 인터랙티브 | alias, 함수, 프롬프트 설정 |

일반적인 구성 예시는 다음과 같다.

```bash
# ~/.bash_profile
export PATH="$HOME/.local/bin:$PATH"
if [ -f ~/.bashrc ]; then
    . ~/.bashrc
fi
```

```bash
# ~/.bashrc
alias ll='ls -alF'
export HISTCONTROL=ignoredups
PS1='\u@\h:\w\$ '
```

## 변수와 환경

변수는 `변수명=값` 형태로 정의하며, 공백을 넣으면 안 된다.

```bash
name="DoHyun"
echo "Hello, $name"
```

`export`를 사용하면 자식 프로세스에 환경변수로 전달된다.

```bash
export APP_ENV=prod
```

유용한 내장 변수: `$HOME`, `$USER`, `$PWD`, `$SHELL`, `$BASH_VERSION`.

## 스크립트 안전 옵션

스크립트 상단에 다음을 추가하면 예기치 않은 오류를 빠르게 감지할 수 있다.

```bash
#!/usr/bin/env bash
set -Eeuo pipefail
IFS=$'\n\t'
```

* `-E`: trap ERR를 함수와 서브셸에 전파
* `-e`: 명령 실패 시 즉시 종료
* `-u`: 설정되지 않은 변수 사용 시 에러
* `-o pipefail`: 파이프라인 어느 부분에서든 실패 시 감지
* `IFS` 재정의: 공백 관련 버그 방지

## 따옴표와 글로빙

```bash
echo "$HOME"     # 변수 확장 O, 공백 보존
echo '$HOME'     # 리터럴 문자열
echo file\ name.txt   # 공백 이스케이프
```

패턴 매칭 시 매칭되는 파일이 없으면 패턴 그대로 남을 수 있다. `nullglob`을 설정하면 빈 배열로 처리된다.

```bash
shopt -s nullglob
for f in *.log; do
    echo "$f"
done
```

## 명령 치환과 산술 연산

```bash
now="$(date +%F)"
echo "오늘은 $now"
n=$((3 + 5 * 2))   # 산술 확장, 결과는 13
```

## 파라미터 확장

문자열 조작에 자주 사용된다.

```bash
file="dir/archive.tar.gz"
echo "${file##*/}"      # basename: archive.tar.gz
echo "${file%/*}"       # dirname: dir
echo "${file%%.*}"      # dir/archive
echo "${file#*/}"       # archive.tar.gz

# 기본값 설정
echo "${USER_NAME:-anonymous}"   # USER_NAME이 없으면 anonymous

# 문자열 치환
s="a-b-c"
echo "${s//-/_}"                 # a_b_c
```

## 배열

```bash
fruits=(apple banana cherry)
echo "${fruits[1]}"        # banana
echo "${fruits[@]}"        # 모든 요소
echo "${#fruits[@]}"       # 배열 길이

for f in "${fruits[@]}"; do
    echo "$f"
done
```

연관 배열(딕셔너리)도 사용 가능하다.

```bash
declare -A conf=([host]=example.com [port]=443)
echo "${conf[host]}"
for k in "${!conf[@]}"; do
    echo "$k=${conf[$k]}"
done
```

## 조건문

`[ ]`와 `[[ ]]` 모두 사용 가능하다. `[[ ]]`는 패턴 매칭을 지원하고 안전하다.

```bash
if [[ "$name" == "kim" ]]; then
    echo "같다"
fi

if [[ -f /etc/hosts ]]; then
    echo "파일 존재"
fi

if [[ -d /tmp ]]; then
    echo "디렉토리 존재"
fi
```

숫자 비교: `-lt`, `-gt`, `-eq` 등을 사용한다.

```bash
if [[ $a -lt $b ]]; then
    echo "a가 b보다 작다"
fi
```

## case 문

```bash
read -r -p "확장자: " ext
case "$ext" in
    txt)  echo "텍스트 파일";;
    jpg|png) echo "이미지 파일";;
    *)    echo "기타";;
esac
```

## 반복문

```bash
# for 문
for i in {1..3}; do
    echo "$i"
done

# 파일 목록 처리
for f in *.txt; do
    echo "처리 중: $f"
done

# while 문으로 파일 읽기
while IFS= read -r line; do
    echo "라인: $line"
done < input.txt
```

## 함수

```bash
sum() {
    local a="$1" b="$2"
    echo $((a + b))        # 결과를 표준 출력으로 반환
}

result="$(sum 3 4)"
echo "결과: $result"       # 7

# 종료 코드로 성공/실패 반환
is_dir() {
    [[ -d "$1" ]]
}
if is_dir "/tmp"; then
    echo "디렉토리 존재"
fi
```

## 리디렉션

| 기호 | 의미 |
|------|------|
| `>` | 표준 출력 덮어쓰기 |
| `>>` | 표준 출력 추가 |
| `<` | 파일을 표준 입력으로 |
| `2>` | 표준 에러 리디렉션 |
| `&>` | 표준 출력과 에러 모두 리디렉션 |

```bash
ls > out.txt
ls missing 2> err.txt
cmd &> all.log
```

## Here Document와 Here String

```bash
# Here Document (변수 확장 안 하려면 'EOF' 사용)
cat <<'EOF'
리눅스 공부는 재미있습니다.
EOF

# 설정 파일 생성
cat > config.ini <<EOF
port=8080
debug=true
EOF

# Here String
grep -i linux <<< "I love Linux"
```

## getopts: 옵션 파싱

```bash
#!/usr/bin/env bash
usage() { echo "사용법: $0 [-f FILE] [-v]"; }

file=""
verbose=0
while getopts ":f:v" opt; do
    case "$opt" in
        f) file="$OPTARG" ;;
        v) verbose=1 ;;
        \?) usage; exit 2 ;;
        :) echo "옵션 -$OPTARG 에 값이 필요합니다"; exit 2 ;;
    esac
done
shift $((OPTIND-1))

(( verbose )) && echo "파일=$file, 나머지 인자=$*"
```

## trap: 안전한 정리

```bash
#!/usr/bin/env bash
tmpdir="$(mktemp -d)"
cleanup() {
    rm -rf "$tmpdir"
}
trap cleanup EXIT INT TERM

# 작업 수행
cp data.csv "$tmpdir/"
# 스크립트 종료 시 cleanup 자동 실행
```

## 실전 예제

### 1. 디렉토리 생성

```bash
#!/usr/bin/env bash
set -Eeuo pipefail
dir="/tmp/mydir"
if [[ ! -d "$dir" ]]; then
    mkdir -p "$dir"
    echo "생성됨: $dir"
fi
```

### 2. 날짜 기반 백업

```bash
#!/usr/bin/env bash
set -Eeuo pipefail
src="/var/log/syslog"
dst="/backup/syslog_$(date +%F).log"
install -D -m 0640 "$src" "$dst"
echo "백업 완료: $dst"
```

### 3. 간단한 다운로더

```bash
#!/usr/bin/env bash
set -Eeuo pipefail
url="$1"; out="$2"
retry=5
for i in $(seq 1 "$retry"); do
    if curl -fsS -o "$out" "$url"; then
        echo "다운로드 성공"
        exit 0
    fi
    echo "실패 ($i/$retry), 2초 후 재시도..."
    sleep 2
done
echo "모든 재시도 실패"
exit 1
```

## 핵심 명령어 요약

| 기능 | 예시 |
|------|------|
| 변수 할당 | `name="value"` |
| 환경 변수 | `export VAR=value` |
| 명령 치환 | `$(cmd)` |
| 산술 연산 | `$(( a + b ))` |
| 파라미터 확장 | `${var:-default}`, `${var#pattern}` |
| 조건 테스트 | `[[ -f file ]]` |
| 반복 | `for i in ...; do ...; done` |
| 함수 | `func() { ...; }` |
| 리디렉션 | `cmd > out 2>&1` |
| here-doc | `cat <<'EOF' ... EOF` |
| 옵션 파싱 | `getopts` |
| 정리 | `trap cleanup EXIT` |

## 결론

Bash는 리눅스 환경에서 가장 강력한 자동화 도구 중 하나이다. 작은 명령어들을 파이프로 연결하고 조건문, 반복문, 함수를 활용하면 복잡한 작업도 간결하게 처리할 수 있다.

스크립트를 작성할 때는 항상 `set -Eeuo pipefail`로 기본 안전 장치를 걸고, 임시 파일은 `mktemp`로 생성하며, `trap`으로 정리를 보장하는 것이 좋다. 이 문서에서 소개한 패턴들을 기본 템플릿으로 삼으면 일상적인 시스템 관리 작업을 효율적으로 자동화할 수 있다.

더 자세한 내용은 `man bash`와 [ShellCheck](https://www.shellcheck.net/)를 활용해 코드 품질을 높이길 권장한다.