---
layout: post
title: Linux - Bash 쉘과 스크립트
date: 2024-11-07 19:20:23 +0900
category: Linux
---
# Bash 쉘과 스크립트 — 환경 설정, 조건문, 반복문, 함수, 리디렉션

## Bash란?

**Bash (Bourne Again SHell)** 는 리눅스에서 가장 널리 사용되는 셸입니다. 명령 실행과 스크립팅을 모두 지원하며, GNU 유틸리티와 결합하여 강력한 자동화 기능을 제공합니다.

---

## 셸 종류와 시작 모드

리눅스의 셸은 실행 방식에 따라 세 가지 모드로 구분됩니다:

- **로그인 셸**: 콘솔이나 SSH로 처음 로그인할 때 실행됩니다. 시스템 전역과 사용자별 프로필을 로드합니다.
- **인터랙티브 셸**: 사용자가 프롬프트와 상호작용하는 모드로, alias나 프롬프트 설정(PS1) 등이 적용됩니다.
- **논인터랙티브 셸**: 스크립트 실행이나 cron 작업처럼 상호작용 없이 실행되는 모드입니다.

**시작 파일 로딩 순서** (배포판에 따라 차이가 있을 수 있습니다):

- **로그인 셸**: `/etc/profile` → `~/.bash_profile` (또는 `~/.bash_login`, `~/.profile`)
- **인터랙티브 비로그인 셸**: `~/.bashrc`
- **논인터랙티브 셸**: `BASH_ENV` 환경변수가 설정되어 있으면 해당 파일을 읽습니다

---

## 환경 설정 파일

| 파일 | 대상 | 주요 용도 |
|---|---|---|
| `/etc/profile` | 시스템 전역 로그인 | PATH, locale 등의 시스템 전역 설정 |
| `~/.bash_profile` | 사용자 로그인 | 사용자별 PATH, 환경 변수 설정 |
| `/etc/bash.bashrc` | 시스템 전역 인터랙티브 | 공용 alias 및 프롬프트 설정 |
| `~/.bashrc` | 사용자 인터랙티브 | 개인별 alias, 함수, 프롬프트 설정 |

**권장 구성 템플릿**:

```bash
# ~/.bash_profile (로그인 시 실행)
export PATH="$HOME/.local/bin:$PATH"

# 인터랙티브 설정을 재사용
if [ -f ~/.bashrc ]; then
    . ~/.bashrc
fi
```

```bash
# ~/.bashrc (인터랙티브 셸에서 실행)
# 안전 옵션 (인터랙티브에서는 -e 옵션 사용 시 주의)
shopt -s histappend checkwinsize
export HISTCONTROL=ignoredups:erasedups

# 편의성 alias
alias ll='ls -alF'
alias gs='git status'

# 프롬프트 설정
PS1='\u@\h:\w\$ '
```

---

## 변수와 환경

```bash
name="DoHyun"
echo "Hello, $name"
export APP_ENV=prod      # 자식 프로세스에 전달되는 환경변수
```

**중요한 규칙**:
- 변수 대입 시 **공백 없이** 작성: `x = 1` (잘못됨) / `x=1` (올바름)
- 변수 확장 시 따옴표 사용: `"${var}"` (공백이나 특수문자가 포함된 경우 안전)

**유용한 내장 변수**:
```bash
echo "$HOME" "$USER" "$PWD" "$SHELL" "$BASH_VERSION"
```

---

## 안전 플래그와 관용구

스크립트의 실패를 빠르게 감지하고 파이프라인 에러를 놓치지 않기 위한 표준 설정:

```bash
#!/usr/bin/env bash

set -Eeuo pipefail
# -E: trap ERR 시그널이 함수와 서브셸에 전파됨
# -e: 명령 실패 시 즉시 종료
# -u: 설정되지 않은 변수 사용 시 에러 발생
# -o pipefail: 파이프라인 어느 부분에서든 실패 시 감지

IFS=$'\n\t'  # IFS 기본값 축소 (공백 관련 버그 방지)
```

> 인터랙티브 셸에서는 `set -e`가 예상치 못한 종료를 유발할 수 있으므로 주의가 필요합니다.

---

## Quoting(따옴표)와 글로빙

```bash
echo "$HOME"   # 변수 확장 O, 공백 보존
echo '$HOME'   # 리터럴 문자열
echo file\ name.txt     # 공백 이스케이프
```

**글로빙과 nullglob**:
```bash
shopt -s nullglob       # 패턴에 매칭되는 파일이 없을 때 빈 목록 반환
for f in *.log; do
    echo "$f"
done
```

---

## 명령 치환과 산술 연산

```bash
now="$(date +%F)"
echo "오늘은 $now"
n=$((3 + 5 * 2))   # 산술 확장
```

---

## 파라미터 확장(문자열 조작의 핵심)

```bash
file="dir/archive.tar.gz"
echo "${file##*/}"       # basename: archive.tar.gz
echo "${file%/*}"        # dirname: dir
echo "${file%%.*}"       # 가장 오른쪽 .부터 전부 제거: dir/archive
echo "${file#*/}"        # 처음 /까지 제거: archive.tar.gz

# 기본값/필수값 설정
echo "${USER_NAME:-anonymous}"      # 비어있거나 unset이면 anonymous 사용
: "${REQUIRED?need REQUIRED env}"   # 변수가 없으면 즉시 에러 종료

# 문자열 치환
s="a-b-c"
echo "${s//-/_}"                    # a_b_c
```

---

## 배열과 연관 배열

### 인덱스 배열

```bash
fruits=(apple banana cherry)
echo "${fruits[1]}"                # banana
echo "${fruits[@]}"                # 모든 요소
echo "${#fruits[@]}"               # 배열 길이
for f in "${fruits[@]}"; do echo "$f"; done
```

### 연관 배열(딕셔너리)

```bash
declare -A conf=([host]=example.com [port]=443)
echo "${conf[host]}:${conf[port]}"
for k in "${!conf[@]}"; do echo "$k=${conf[$k]}"; done
```

---

## 조건문 (test/[ ] vs [[ ]])

```bash
x="kim"
if [ "$x" = "kim" ]; then
    echo "같다 (전통적인 test 명령)"
fi

if [[ $x == ki* ]]; then
    echo "[[ ]]는 패턴 매칭을 지원하고 리디렉션에 안전"
fi
```

**숫자 비교와 파일 조건**:
```bash
if [[ $a -lt $b ]]; then ...; fi
if [[ -f /etc/hosts ]]; then ...; fi
if [[ -d /tmp ]]; then ...; fi
if [[ -s bigfile ]]; then echo "파일 크기가 0보다 큼"; fi
```

**정규식 매칭(확장 테스트)**:
```bash
line="status: 200 OK"
if [[ $line =~ ^status:\ ([0-9]{3})\ .* ]]; then
    code="${BASH_REMATCH[1]}"
    echo "HTTP 코드: $code"
fi
```

---

## case 문

```bash
read -r -p "확장자: " ext
case "$ext" in
    txt)  echo "텍스트 파일";;
    jpg|png) echo "이미지 파일";;
    *)    echo "기타 파일 형식";;
esac
```

---

## 반복문(for/while/select)

```bash
# 기본 for 문
for i in {1..3}; do echo "$i"; done

# 파일 목록 처리
shopt -s nullglob
for f in *.txt; do echo "처리 중: $f"; done

# while 문과 read -r (공백/백슬래시 안전)
while IFS= read -r line; do
    echo "라인: $line"
done < input.txt

# select 문 — 간단한 메뉴 생성
select opt in start stop quit; do
    case $opt in
        start) echo "시작합니다"; break;;
        stop)  echo "중단합니다"; break;;
        quit)  exit 0;;
        *)     echo "번호를 선택하세요";;
    esac
done
```

---

## 함수, 지역변수, 반환값

```bash
# 함수 정의
sum() {
    local a="$1" b="$2"  # 지역 변수 선언
    echo $((a+b))        # echo로 값 반환 (표준 출력)
}

# 함수 호출
v="$(sum 3 4)"
echo "결과: $v"          # 7

# 종료 코드 반환 (0-255)
is_dir() { [[ -d "$1" ]]; }
if is_dir "/tmp"; then echo "디렉토리가 존재합니다"; fi
```

---

## 사용자 입력 처리

```bash
read -r -p "이름: " name
read -r -s -p "비밀번호: " pw; echo
```

---

## 리디렉션과 파일 디스크립터

| 기호 | 의미 |
|---|---|
| `>` | stdout 덮어쓰기 |
| `>>` | stdout 추가 |
| `<` | 파일을 stdin으로 리디렉션 |
| `2>` | stderr 리디렉션 |
| `&>` | stdout과 stderr 모두 리디렉션 |

**예시**:
```bash
ls > out.txt
ls missing 2> err.txt
cmd &> all.log
```

**스크립트 전체의 표준 에러를 파일로 리디렉션**:
```bash
exec 2>error.log
echo "이 메시지는 stderr로만 출력됩니다" >&2
```

**파일 디스크립터 복제 및 교체**:
```bash
# FD 3에 파일 열기
exec 3>app.log
echo "로그 메시지" >&3
exec 3>&-        # 파일 디스크립터 닫기
```

---

## Here Document와 Here String

```bash
# here-doc
cat <<'EOF'
리눅스 공부는 재미있습니다.
쉘 스크립트는 강력합니다.
EOF

# 설정 파일 생성
cat > config.ini <<EOF
[setting]
enabled=true
port=8080
EOF

# here-string
grep -i linux <<< "I love Linux"
```

> `<<'EOF'` (작은따옴표 사용)는 **변수 치환을 하지 않습니다**.

---

## 파이프라인과 pipefail

```bash
set -o pipefail
grep -i error big.log | awk '{print $1}' | sort -u > keys.txt
```
- 파이프라인의 어느 단계에서든 실패하면 종료 코드가 전파되도록 보장합니다.

---

## 프로세스 치환, 명령 그룹, 서브셸

```bash
# 프로세스 치환: 파일처럼 다루기
diff <(sort a.txt) <(sort b.txt)

# 그룹화: 리디렉션 한 번에 적용
{ echo "시작"; run_task; echo "종료"; } > log.txt 2>&1

# 서브셸: 현재 셸 환경을 오염시키지 않고 임시 실행
( cd dir && make )
```

---

## getopts — 표준 옵션 파싱

```bash
#!/usr/bin/env bash

set -Eeuo pipefail

usage() { echo "사용법: $0 [-f FILE] [-v] [--] args..."; }

file=""
verbose=0
while getopts ":f:v" opt; do
    case "$opt" in
        f) file="$OPTARG" ;;
        v) verbose=1 ;;
        \?) echo "알 수 없는 옵션: -$OPTARG"; usage; exit 2;;
        :) echo "옵션 -$OPTARG 에 값이 필요합니다"; exit 2;;
    esac
done
shift $((OPTIND-1))

(( verbose )) && echo "파일=$file, 나머지 인자=$*"
```

---

## trap과 안전한 정리(cleanup)

```bash
#!/usr/bin/env bash

set -Eeuo pipefail
tmpdir="$(mktemp -d)"

# 정리 함수 정의
cleanup(){
    rm -rf "$tmpdir"
}

# 정리 함수를 트랩으로 등록
trap cleanup EXIT INT TERM

# 작업 수행
cp data.csv "$tmpdir/"

# 스크립트가 어떤 방식으로든 종료되면 cleanup이 보장됨
```

---

## 동시성: 백그라운드, wait, flock, xargs

```bash
# 간단한 병렬 실행
task() { sleep 1; echo "$1 완료"; }
for i in {1..5}; do task "$i" & done
wait  # 모든 백그라운드 작업 완료 대기

# 첫 번째 완료 작업 기다리기 (Bash 5.0 이상)
while true; do
    if wait -n; then
        echo "하나의 작업 완료"
    else
        break
    fi
done
```

**파일 잠금을 통한 동시성 제어**:
```bash
# 단일 인스턴스 실행 보장
exec 9>/var/lock/myjob.lock
flock -n 9 || { echo "이미 실행 중입니다"; exit 0; }
# 안전하게 본문 실행...
```

**xargs를 이용한 병렬화**:
```bash
printf '%s\n' file1 file2 file3 | xargs -n1 -P4 gzip -9
```

---

## 입출력 파싱: read, mapfile, printf

```bash
# 파일을 한 줄씩 배열로 읽기
mapfile -t lines < input.txt
printf '라인: %s\n' "${lines[@]}"
```

---

## 텍스트 처리 팁(awk, sed, cut, sort, uniq)

```bash
awk -F, '{sum+=$3} END{print sum}' data.csv
sed -E 's/[[:space:]]+$//' file.txt > trimmed.txt
sort access.log | uniq -c | sort -nr | head
```

---

## 컬러 출력과 로깅 유틸리티

```bash
# 색상 출력 (tput 사용)
info(){ printf '%s\n' "$*"; }
ok(){ printf '\e[32m%s\e[0m\n' "$*"; }      # 초록색
warn(){ printf '\e[33m%s\e[0m\n' "$*"; }    # 노란색
err(){ printf '\e[31m%s\e[0m\n' "$*" >&2; } # 빨간색

# 타임스탬프 포함 로그
ts(){ date '+%F %T'; }
log(){ printf '%s %s\n' "$(ts)" "$*"; }

log "작업 시작"; ok "성공적으로 완료"; warn "주의 필요"; err "오류 발생"
```

---

## cron과 systemd 환경 차이

- **cron**과 **systemd**는 로그인 셸의 설정을 자동으로 읽지 않습니다.
- 필요한 환경 변수는 스크립트에서 **명시적으로 export**하거나 systemd 유닛 파일의 `Environment=` 지시어를 사용해야 합니다.

```bash
# systemd 서비스 유닛 예시
[Service]
Environment=APP_ENV=prod
EnvironmentFile=/etc/myapp/env
```

---

## Here Document를 이용한 설정 파일 자동 생성

```bash
#!/usr/bin/env bash

set -Eeuo pipefail

conf=/etc/myapp/config.ini
sudo install -d -m 0750 -o root -g myapp /etc/myapp

sudo tee "$conf" >/dev/null <<'CFG'
[server]
port=8080
workers=4
log_level=info
CFG

sudo chown root:myapp "$conf"
sudo chmod 0640 "$conf"
```

---

## 실전 스크립트 예제

### 1. 디렉토리가 없으면 생성

```bash
#!/usr/bin/env bash

set -Eeuo pipefail
dir="/tmp/mydir"
if [[ ! -d "$dir" ]]; then
    mkdir -p "$dir"
    echo "디렉토리 생성: $dir"
fi
```

### 2. 날짜 기반 로그 백업

```bash
#!/usr/bin/env bash

set -Eeuo pipefail
src="/var/log/syslog"
dst="/backup/syslog_$(date +%F).log"
install -D -m 0640 "$src" "$dst"
echo "백업 완료: $dst"
```

### 3. 작업 결과 로그 누적

```bash
#!/usr/bin/env bash

set -Eeuo pipefail
{
    echo "시작 시간: $(date)"
    uptime
    echo "종료 시간: $(date)"
} >> run.log
```

### 4. 견고한 다운로더 (재시도와 로깅)

```bash
#!/usr/bin/env bash

set -Eeuo pipefail
url="$1"; out="$2"
: "${url?URL이 필요합니다}" "${out?출력 파일명이 필요합니다}"

retry=5
for i in $(seq 1 "$retry"); do
    if curl -fsS --retry 3 -o "$out" "$url"; then
        echo "다운로드 성공: $out"
        exit 0
    fi
    echo "실패($i/$retry), 2초 후 재시도..."
    sleep 2
done
echo "모든 재시도 실패"; exit 1
```

### 5. getopts + trap + 병렬 처리

```bash
#!/usr/bin/env bash

set -Eeuo pipefail
tmp="$(mktemp -d)"
trap 'rm -rf "$tmp"' EXIT

concurrency=4
out="hashes.txt"

usage(){ echo "사용법: $0 [-j N] [-o OUT] -- files..."; }

while getopts ":j:o:h" opt; do
    case "$opt" in
        j) concurrency="$OPTARG" ;;
        o) out="$OPTARG" ;;
        h) usage; exit 0 ;;
        \?) echo "옵션 오류: -$OPTARG"; usage; exit 2;;
        :) echo "옵션 값 필요: -$OPTARG"; exit 2;;
    esac
done
shift $((OPTIND-1))

(( $# > 0 )) || { usage; exit 2; }

printf '%s\0' "$@" | xargs -0 -n1 -P"$concurrency" -I{} sh -c '
    f="$1"
    sha256sum "$f"
' _ {} > "$tmp/$out"

mv "$tmp/$out" "./$out"
echo "파일 생성 완료: $out"
```

---

## 성능 및 대용량 데이터 처리 주의점

- `while read -r`과 파이프 대신 **리다이렉션**을 사용하면 서브셸 생성을 피할 수 있습니다:
  ```bash
  while IFS= read -r line; do
      # 처리 로직
  done < input.txt
  ```
- `grep`, `sed`, `awk`는 **라인 스트리밍 방식**으로 작동하므로 매우 큰 파일 처리에 적합합니다.
- 공백이나 특수문자가 포함된 파일명은 **NUL 구분자** 방식(`-print0`, `xargs -0`)으로 처리하는 것이 안전합니다.

---

## 스타일 가이드와 코드 품질

- **ShellCheck**를 사용하여 코드 린트:
  ```bash
  shellcheck script.sh
  ```
- 일관된 **에러 처리**(trap, 사용법 메시지, 종료 코드)와 **로깅**을 구현합니다.
- **POSIX sh** 호환성이 필요한 경우 Bash 전용 기능(`[[ ]]`, 배열, `mapfile`, 프로세스 치환)을 피합니다.

---

## 수학 연산과 부동소수점

Bash의 산술 연산은 **정수 연산**만 지원합니다:
$$
\text{result} = a \times b + c \quad (\text{정수 범위})
$$

부동소수점 연산이 필요하면 `bc`나 `python`을 호출합니다:
```bash
echo "scale=3; 10/3" | bc
```

---

## 핵심 개념 요약

| 주제 | 주요 내용 |
|---|---|
| 시작 파일 | 로그인/인터랙티브별 설정 파일 로딩 순서 |
| 안전 옵션 | `set -Eeuo pipefail`과 `IFS` 설정 |
| Quoting/글로빙 | 변수 확장 시 따옴표 사용, nullglob 옵션 |
| 파라미터 확장 | 기본값 설정, 문자열 치환, 경로 조작 |
| 배열 | 인덱스 배열과 연관 배열 선언 및 사용 |
| 조건문/반복문 | `[[ ]]` 테스트, `case`, `for`, `while read` |
| 함수 | 지역변수, 값 반환, 종료 코드 |
| 리디렉션/FD | 표준 입출력 조작, 파일 디스크립터 관리 |
| 파이프라인 | `pipefail` 옵션으로 에러 감지 |
| 프로세스 치환 | `<( )` 구문으로 명령 출력을 파일처럼 사용 |
| 옵션 파싱 | `getopts`로 명령줄 인자 처리 |
| 종료/정리 | `trap`으로 정리 작업 보장, `mktemp`로 임시 파일 관리 |
| 병렬 처리 | 백그라운드 실행, `wait`, `xargs -P`, `flock` |
| 코드 검증 | ShellCheck로 코드 품질 검사 |

---

## 결론

Bash는 리눅스 시스템에서 강력한 자동화 도구로, 작은 명령어들을 파이프라인으로 조합하여 복잡한 작업을 간결하게 처리할 수 있습니다. 본 문서에서 소개한 **안전한 스크립트 작성 패턴**(`set -Eeuo pipefail` + `trap` + `getopts` + `mktemp`)을 기본 템플릿으로 활용하면, 일상적인 시스템 관리 작업을 자동화하는 효율적인 스크립트를 작성할 수 있습니다.

실무에서는 특히 에러 처리와 로깅을 철저히 구현하고, 대용량 데이터 처리 시 성능을 고려한 접근 방식을 채택하는 것이 중요합니다. Bash 스크립팅은 단순한 명령어 나열을 넘어서, 시스템의 상태를 감지하고 조건에 따라 적절히 대응하는 지능적인 자동화 도구로 발전시킬 수 있는 기술입니다. 이러한 기술을 익히면 시스템 관리, 배포 자동화, 모니터링 등 다양한 영역에서 생산성을 크게 향상시킬 수 있습니다.