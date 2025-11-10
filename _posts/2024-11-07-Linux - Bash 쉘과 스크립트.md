---
layout: post
title: Linux - Bash 쉘과 스크립트
date: 2024-11-07 19:20:23 +0900
category: Linux
---
# Bash 쉘과 스크립트 — 환경 설정, 조건문, 반복문, 함수, 리디렉션

## Bash란?
**Bash (Bourne Again SHell)** 는 리눅스에서 가장 널리 쓰이는 셸이다. 명령 실행과 스크립팅을 모두 지원하며, GNU 유틸리티와 결합해 강력한 자동화를 제공한다.

---

## 셸 종류와 시작 모드
- **로그인 셸**: 콘솔/SSH 최초 로그인 시. 시스템 전역/사용자 프로파일을 로드.
- **인터랙티브 셸**: 프롬프트와 상호작용. alias, 프롬프트(PS1) 등 적용.
- **논인터랙티브 셸**: 스크립트 실행(`bash script.sh`)이나 cron 등.

**시작 파일 로딩(전형적 순서, 배포판에 따라 차이 가능)**  
- **로그인 셸**: `/etc/profile` → `~/.bash_profile` (또는 `~/.bash_login`, `~/.profile`)  
- **인터랙티브 비로그인 셸**: `~/.bashrc`  
- **논인터랙티브 셸**: `BASH_ENV`가 설정되어 있으면 그 파일을 읽음

---

## 환경 설정 파일
| 파일 | 대상 | 주요 용도 |
|---|---|---|
| `/etc/profile` | 시스템 전역 로그인 | PATH, locale 등 |
| `~/.bash_profile` | 사용자 로그인 | PATH, 환경 변수 export, `source ~/.bashrc` 권장 |
| `/etc/bash.bashrc` | 시스템 전역 인터랙티브 | 공용 alias/프롬프트 |
| `~/.bashrc` | 사용자 인터랙티브 | alias, 함수, 프롬프트(PS1), 셸 옵션 |

**권장 템플릿**
```bash
# ~/.bash_profile
# 로그인 시 한 번만
export PATH="$HOME/.local/bin:$PATH"
# 인터랙티브 설정을 재사용
if [ -f ~/.bashrc ]; then
  . ~/.bashrc
fi
```

```bash
# ~/.bashrc (인터랙티브)
# 안전 옵션(인터랙티브에선 -e는 호불호가 있음)
shopt -s histappend checkwinsize
export HISTCONTROL=ignoredups:erasedups
alias ll='ls -alF'
alias gs='git status'
PS1='\u@\h:\w\$ '     # 간단 PS1
```

---

## 변수와 환경
```bash
name="DoHyun"
echo "Hello, $name"
export APP_ENV=prod      # 자식 프로세스에 전달되는 환경변수
```

**중요 규칙**
- 대입에 **공백 없음**: `x = 1` (X) / `x=1` (O)
- 안전한 확장: `"${var}"` (공백·특수문자 포함 시 안전)

**유용한 내장 변수**
```bash
echo "$HOME" "$USER" "$PWD" "$SHELL" "$BASH_VERSION"
```

---

## 안전 플래그와 관용구
스크립트의 **실패를 빠르게 감지**하고, **파이프라인 에러**를 놓치지 않기 위한 표준:

```bash
#!/usr/bin/env bash
set -Eeuo pipefail
# -E: trap ERR 에 함수/서브셸 전파
# -e: 명령 실패 시 즉시 종료
# -u: 설정되지 않은 변수 사용 시 에러
# -o pipefail: 파이프라인 어느 곳에서든 실패 감지
IFS=$'\n\t'  # IFS 기본값 축소 (공백 관련 버그 방지)
```

> 인터랙티브 셸에서는 `set -e`가 예상치 못한 종료를 유발할 수 있으니 주의.

---

## quoting(따옴표)와 글로빙
```bash
echo "$HOME"   # 변수 확장 O, 공백 보존
echo '$HOME'   # 리터럴 문자열
echo file\ name.txt     # 이스케이프
```

**글로빙과 nullglob**
```bash
shopt -s nullglob       # 매칭 없을 때 패턴 자체가 아닌 빈 목록
for f in *.log; do
  echo "$f"
done
```

---

## 명령 치환과 산술
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

# 기본값/필수값
echo "${USER_NAME:-anonymous}"      # 비어있거나 unset이면 anonymous
: "${REQUIRED?need REQUIRED env}"   # 없으면 즉시 에러 종료

# 치환
s="a-b-c"
echo "${s//-/_}"                    # a_b_c
```

---

## 배열과 연관배열
### 인덱스 배열
```bash
fruits=(apple banana cherry)
echo "${fruits[1]}"                # banana
echo "${fruits[@]}"                # 모든 요소
echo "${#fruits[@]}"               # 길이
for f in "${fruits[@]}"; do echo "$f"; done
```

### 연관배열(딕셔너리)
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
  echo "같다(전통 test)"
fi

if [[ $x == ki* ]]; then
  echo "[[ ]]는 패턴 매칭/리디렉션 안전"
fi
```

**숫자/파일 조건**
```bash
if [[ $a -lt $b ]]; then ...; fi
if [[ -f /etc/hosts ]]; then ...; fi
if [[ -d /tmp ]]; then ...; fi
if [[ -s bigfile ]]; then echo "크기>0"; fi
```

**정규식 매칭(확장 테스트)**
```bash
line="status: 200 OK"
if [[ $line =~ ^status:\ ([0-9]{3})\ .* ]]; then
  code="${BASH_REMATCH[1]}"
  echo "code=$code"
fi
```

---

## case 문
```bash
read -r -p "확장자: " ext
case "$ext" in
  txt)  echo "텍스트";;
  jpg|png) echo "이미지";;
  *)    echo "기타";;
esac
```

---

## 반복문(for/while/select)
```bash
for i in {1..3}; do echo "$i"; done

# 파일 목록
shopt -s nullglob
for f in *.txt; do echo "처리: $f"; done

# while + read -r (공백/백슬래시 안전)
while IFS= read -r line; do
  echo ">$line"
done < input.txt

# select — 간단 메뉴
select opt in start stop quit; do
  case $opt in
    start) echo start; break;;
    stop)  echo stop; break;;
    quit)  exit 0;;
    *)     echo "번호 선택";;
  esac
done
```

---

## 함수, 지역변수, 반환값
```bash
sum() {
  local a="$1" b="$2"
  echo $((a+b))    # echo로 반환(표준 출력)
}

v="$(sum 3 4)"
echo "$v"          # 7

# 종료코드(return)는 0~255
is_dir() { [[ -d "$1" ]]; }
if is_dir "/tmp"; then echo "OK"; fi
```

---

## 입력(읽기)와 프롬프트
```bash
read -r -p "이름: " name
read -r -s -p "비밀번호: " pw; echo
```

---

## 리디렉션/파일디스크립터
| 기호 | 의미 |
|---|---|
| `>` | stdout 덮어쓰기 |
| `>>` | stdout 추가 |
| `<` | 파일→stdin |
| `2>` | stderr |
| `&>` | stdout+stderr |

**예시**
```bash
ls > out.txt
ls missing 2> err.txt
cmd &> all.log
```

**전체 스크립트 표준에러만 파일로**
```bash
exec 2>error.log
echo "stderr로만" >&2
```

**FD 복제/교체**
```bash
# FD 3에 파일 열기
exec 3>app.log
echo "hello" >&3
exec 3>&-        # 닫기
```

---

## Here Document / Here String
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

> `<<'EOF'` (작은따옴표)면 **변수 치환 없음**.

---

## 파이프라인과 pipefail
```bash
set -o pipefail
grep -i error big.log | awk '{print $1}' | sort -u > keys.txt
```
- 어느 단계가 실패해도 종료코드가 전파되도록 보장.

---

## 프로세스 치환 / 명령 그룹 / 서브셸
```bash
# 프로세스 치환: 파일처럼 다루기
diff <(sort a.txt) <(sort b.txt)

# 그룹화: 리디렉션 한 번에
{ echo "start"; run_task; echo "end"; } > log.txt 2>&1

# 서브셸: 현재 쉘 오염 없이 임시로
( cd dir && make )
```

---

## getopts — 표준 옵션 파싱
```bash
#!/usr/bin/env bash
set -Eeuo pipefail

usage(){ echo "사용법: $0 [-f FILE] [-v] [--] args..."; }

file=""
verbose=0
while getopts ":f:v" opt; do
  case "$opt" in
    f) file="$OPTARG" ;;
    v) verbose=1 ;;
    \?) echo "알 수 없는 옵션: -$OPTARG"; usage; exit 2;;
    :) echo "옵션 -$OPTARG 에 값 필요"; exit 2;;
  esac
done
shift $((OPTIND-1))

(( verbose )) && echo "파일=$file, 나머지=$*"
```

---

## trap과 안전한 정리(cleanup)
```bash
#!/usr/bin/env bash
set -Eeuo pipefail
tmpdir="$(mktemp -d)"
cleanup(){
  rm -rf "$tmpdir"
}
trap cleanup EXIT INT TERM

# 작업...
cp data.csv "$tmpdir/"
# 스크립트가 어떤 방식으로든 끝나면 cleanup 보장
```

---

## 동시성: 백그라운드, wait, wait -n, flock, xargs -P
```bash
# 병렬 실행 (간단)
task() { sleep 1; echo "$1 done"; }
for i in {1..5}; do task "$i" & done
wait  # 모두 완료 대기

# 첫 완료를 기다림 (Bash 5.0+)
while true; do
  if wait -n; then
    echo "하나 완료"
  else
    break
  fi
done
```

**파일 잠금으로 동시성 제어**
```bash
# 단일 인스턴스 실행
exec 9>/var/lock/myjob.lock
flock -n 9 || { echo "이미 실행 중"; exit 0; }
# 안전한 본문...
```

**xargs 병렬화**
```bash
printf '%s\n' file1 file2 file3 | xargs -n1 -P4 gzip -9
```

---

## 입출력 파싱: read/mapfile/printf
```bash
# 한 줄씩 배열로
mapfile -t lines < input.txt
printf 'line: %s\n' "${lines[@]}"
```

---

## 텍스트 처리 팁(awk/sed/cut/sort/uniq)
```bash
awk -F, '{sum+=$3} END{print sum}' data.csv
sed -E 's/[[:space:]]+$//' file.txt > trimmed.txt
sort access.log | uniq -c | sort -nr | head
```

---

## 컬러/로그 유틸
```bash
# 색상 (tput 사용)
info(){ printf '%s\n' "$*"; }
ok(){ printf '\e[32m%s\e[0m\n' "$*"; }      # green
warn(){ printf '\e[33m%s\e[0m\n' "$*"; }
err(){ printf '\e[31m%s\e[0m\n' "$*" >&2; }

ts(){ date '+%F %T'; }
log(){ printf '%s %s\n' "$(ts)" "$*"; }

log "시작"; ok "성공"; warn "주의"; err "실패"
```

---

## 크론/시스템드와 환경 차이
- **cron/systemd**는 로그인 셸의 설정을 자동으로 읽지 않는다.  
- 필요한 환경은 스크립트에서 **명시적으로 export** 하거나 systemd 유닛의 `Environment=`를 사용.

```bash
# systemd 유닛
[Service]
Environment=APP_ENV=prod
EnvironmentFile=/etc/myapp/env
```

---

## Here Doc로 설정/스크립트 생성 자동화 (실전)
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

## 실전 스크립트 1 — 디렉토리 없으면 생성
```bash
#!/usr/bin/env bash
set -Eeuo pipefail
dir="/tmp/mydir"
if [[ ! -d "$dir" ]]; then
  mkdir -p "$dir"
  echo "디렉토리 생성: $dir"
fi
```

---

## 실전 스크립트 2 — 날짜 기반 로그 백업
```bash
#!/usr/bin/env bash
set -Eeuo pipefail
src="/var/log/syslog"
dst="/backup/syslog_$(date +%F).log"
install -D -m 0640 "$src" "$dst"
echo "백업: $dst"
```

---

## 실전 스크립트 3 — 결과 로그 누적
```bash
#!/usr/bin/env bash
set -Eeuo pipefail
{
  echo "시작: $(date)"
  uptime
  echo "종료: $(date)"
} >> run.log
```

---

## 실전 스크립트 4 — robust downloader (재시도/로그)
```bash
#!/usr/bin/env bash
set -Eeuo pipefail
url="$1"; out="$2"
: "${url?URL 필요}" "${out?OUT 필요}"

retry=5
for i in $(seq 1 "$retry"); do
  if curl -fsS --retry 3 -o "$out" "$url"; then
    echo "다운로드 성공: $out"; exit 0
  fi
  echo "실패($i/$retry), 2초 후 재시도..."
  sleep 2
done
echo "포기"; exit 1
```

---

## 실전 스크립트 5 — getopts + trap + 병렬
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
  case_esac=true
  esac
done
shift $((OPTIND-1))

(( $# > 0 )) || { usage; exit 2; }

printf '%s\0' "$@" | xargs -0 -n1 -P"$concurrency" -I{} sh -c '
  f="$1"
  sha256sum "$f"
' _ {} > "$tmp/$out"

mv "$tmp/$out" "./$out"
echo "생성: $out"
```

---

## 성능/대용량 데이터 주의점
- `while read -r` + 파이프 대신 **리다이렉션**을 사용하면 서브셸 생성을 피할 수 있다:
  ```bash
  while IFS= read -r line; do
    # ...
  done < input.txt
  ```
- `grep/sed/awk`는 **라인 스트리밍**이므로 매우 큰 파일에도 적합.
- 공백/특수문자 파일명은 **NUL 구분**(`-print0`, `xargs -0`)로 다룬다.

---

## 스타일 가이드/품질
- **ShellCheck**로 린트:
  ```bash
  shellcheck script.sh
  ```
- 일관된 **에러 처리**(trap/사용법/종료코드)와 **로그**를 갖춘다.
- **POSIX sh** 호환이 필요하면 Bash 전용 기능(`[[ ]]`, 배열, `mapfile`, 프로세스 치환)을 피한다. (이 글은 Bash 전용 문법을 다룸)

---

## 수학 예시(산술 확장 개념)
Bash 산술은 **정수** 연산이다.  
$$
\text{result} = a \times b + c \quad (\text{정수 범위})
$$
부동소수점이 필요하면 `bc` 또는 `python`을 호출한다.
```bash
echo "scale=3; 10/3" | bc
```

---

## 요약 표
| 주제 | 핵심 명령/문법 |
|---|---|
| 시작 파일 | `/etc/profile`, `~/.bash_profile`, `~/.bashrc`, `BASH_ENV` |
| 안전 옵션 | `set -Eeuo pipefail`, `IFS=$'\n\t'` |
| quoting/글로빙 | `"${var}"`, `shopt -s nullglob` |
| 파라미터 확장 | `${var:-def}`, `${var//a/b}`, `${var%/*}`, `${var##*/}` |
| 배열 | `arr=(a b)`, `"${arr[@]}"`, 연관배열 `declare -A` |
| 조건/반복 | `[[ ]]`, `case`, `for`, `while read -r`, `select` |
| 함수/반환 | `local`, `echo`로 값, 종료코드로 상태 |
| 리디렉션/FD | `>`, `2>`, `&>`, `exec 3>file`, here-doc/strings |
| 파이프라인 | `set -o pipefail` |
| 치환/그룹 | `<( )`, `{ ...; } >file`, `( subshell )` |
| 옵션 파싱 | `getopts` |
| 종료/정리 | `trap cleanup EXIT`, `mktemp` |
| 병렬 | `&`, `wait`, `wait -n`, `xargs -P`, `flock` |
| 검증 | `ShellCheck`, 로깅, 종료코드 일관성 |

---

## 마무리
Bash는 **작은 도구들을 파이프라인으로 조합**해 문제를 푸는 언어다. 본문에서 정리한 **안전한 스크립트 틀(set -Eeuo pipefail + trap + getopts + mktemp)** 을 템플릿으로 삼아, 반복 작업을 스크립트로 치환해 나가면 운영·개발 모두에서 생산성이 비약적으로 향상된다. 다음 편에서는 **네트워킹/프로세스/파일 시스템 유틸과 Bash를 결합한 운영 자동화 플레이북**을 다룬다.