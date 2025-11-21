---
layout: post
title: Git - git bisect
date: 2025-03-16 21:20:23 +0900
category: Git
---
# git bisect: 버그 추적 자동화 도구

## 핵심 개념 요약

- `git bisect`는 **이진 탐색(binary search)** 로 “정상(good) 상태”와 “오류(bad) 상태” 사이에서 **최초로 문제가 발생한 커밋**을 찾는 도구다.
- N개의 커밋 중 원인 커밋을 찾는 시간 복잡도는 대략 **O(log₂ N)** 이다.
- 사람이 테스트 결과를 입력하는 **수동 모드**와, 스크립트가 자동으로 판정하는 **자동 모드**(`git bisect run`)가 있다.

---

## 빠른 시작: 최소 절차 예제

### 수동 이진 탐색

```bash
# 시작

git bisect start

# "현재 커밋은 버그 발생"을 선언

git bisect bad

# 과거의 "정상 작동" 커밋을 지정 (해시, 태그, 브랜치 모두 OK)

git bisect good abc1234

# 이후 Git이 중간 커밋으로 체크아웃한다.
# 테스트 실행 후 결과에 따라 다음 중 하나:

git bisect good   # 문제가 없으면
git bisect bad    # 문제가 있으면
git bisect skip   # 빌드 불가, 테스트 불가 등 판정 불가 시
```

마지막에 Git이 아래와 같이 알려준다:

```
abcd1234 is the first bad commit
```

이때 원래 작업 지점으로 돌아가려면:

```bash
git bisect reset
```

### 자동 탐색(스크립트 이용)

```bash
git bisect start
git bisect bad
git bisect good abc1234
git bisect run ./test-script.sh
```

`git bisect run`은 **스크립트의 종료 코드**로 판정을 받는다.

- 0 → **good**
- 1~127(단, 125 제외) → **bad**
- 125 → **skip** (테스트 불가)

예시 스크립트:

```bash
#!/usr/bin/env bash

set -euo pipefail

# 언제나 "깨끗한 빌드"를 보장해야 커밋 간 비교가 공정하다.
# (아래는 Node.js 예시. 언어/도구에 맞게 교체)

rm -rf node_modules dist
npm ci
npm test

# 테스트가 통과하면 0으로 종료되어 "good" 처리된다.

```

---

## 사용 흐름 디테일: 실전 패턴

### 시작과 종료의 축약형

```bash
# start + bad + good 을 한 줄로

git bisect start <bad> <good>
# 예: 현재 HEAD가 bad이고, v1.4.0 태그는 good

git bisect start HEAD v1.4.0

# 탐색 종료 후 특정 지점으로 돌아가고 싶다면 해시나 브랜치 지정 가능

git bisect reset main
```

### bisect 범위 최소화(필요한 경로만)

**특정 디렉터리/파일만** 영향을 주는 버그라면, 그 경로에 변경이 있는 커밋만 대상으로 하면 더 빠르다.

```bash
# good/bad 지정 후

git bisect start
git bisect bad
git bisect good abc1234

# 경로를 지정(예: src/auth/** 만)

git bisect run bash -lc '
  # 변경 없는 커밋이면 125(exit skip)로 건너뛰기
  git diff --quiet HEAD^ -- src/auth || true
  if git diff --quiet HEAD^ -- src/auth; then exit 125; fi

  rm -rf node_modules dist
  npm ci
  npm test -- -t "auth"
'
```

> `exit 125`는 “이 커밋은 판정 대상이 아니다(건너뜀)”을 의미한다.

### 병합 커밋/브랜치가 복잡할 때(ancestry path 제한)

`good..bad` 사이에 **비(非)직선 경로**가 많으면 불필요한 커밋까지 섞일 수 있다. 이럴 땐 **조상 경로만** 타도록 제한한다.

```bash
# good..bad 사이에서 "직계 경로(ancestry path)"만

git rev-list --ancestry-path good..bad
# 위 결과를 바탕으로 수동 bisect를 하거나,
# 또는 검색 대상을 좁히는 스크립트에서 활용

```

> 실무에서는 복잡한 병합 히스토리에서 false-positive를 줄이기 위해 유용하다.

---

## 빌드/테스트 실패, 충돌 등 예외 처리

### 빌드 불가/외부 의존성 문제

- 오래된 커밋은 최신 빌드 체인과 맞지 않아 실패할 수 있다.
- 이런 경우 `git bisect skip` 또는 자동 모드에서 **종료 코드 125**로 표기한다.

```bash
# 스크립트 내부 예

if ! make build; then
  exit 125    # 이 커밋은 테스트 불가 → skip
fi
make test || exit 1  # 실패 → bad
exit 0               # 성공 → good
```

### 병합 충돌

- bisect 중 체크아웃 시 충돌이 발생할 수 있다. 수동 해결이 어렵다면 `git bisect skip`으로 넘긴다.
- 충돌이 많다면 **범위 축소**(경로 제한, ancestry path)로 접근하는 편이 낫다.

### 테스트

- 불안정 테스트는 오탐을 만든다. 자동 스크립트에서 **재시도**를 통해 안정시킨다.

```bash
#!/usr/bin/env bash

set -euo pipefail

retry=0
max=3
while (( retry < max )); do
  if npm test; then
    exit 0  # good
  fi
  retry=$((retry+1))
  sleep 1
done
exit 1      # bad
```

---

## 성능 회귀(Regression)도 bisect로 잡는다

기능 실패뿐 아니라 **느려진 최초 커밋**도 찾을 수 있다. 기준을 정해 **threshold**로 판정한다.

### 예: API 응답 지연 회귀 탐지

```bash
#!/usr/bin/env bash

set -euo pipefail

# 빌드/실행

docker compose build --no-cache api
docker compose up -d api
sleep 2

# 성능 측정 (예: curl + time)

ms=$( (time -p curl -sS http://localhost:3000/health >/dev/null) 2>&1 \
      | awk '/real/ {print $2 * 1000}' )

docker compose down -v

# 임계치: 120ms

threshold=120

# 측정값이 임계치 초과면 "bad(1)"

awk -v v="$ms" -v t="$threshold" 'BEGIN { exit (v>t ? 1 : 0) }'
```

- 0 → good(성능 OK), 1 → bad(회귀), 125 → skip(실행 불가)
- `git bisect run`에 이 스크립트를 연결하면 **성능 회귀 최초 커밋**을 자동 탐지한다.

### 예: Node.js/Java/Go 등 다양한 빌드 체인

핵심은 **항상 같은 방식으로 클린 빌드/실행/측정**하는 것. 언어만 달라질 뿐 패턴은 동일하다.

---

## 로그/재현/공유: 팀에서 함께 살펴보기

### 탐색 로그 남기고 재생하기

```bash
git bisect log > bisect.log
# 동료가 같은 repo에서 재현

git bisect reset
git bisect replay bisect.log
```

- `bisect log`는 good/bad/skip 입력을 기록한다.
- 문제가 재현되는 환경에서 **같은 절차**를 빠르게 반복할 수 있다.

### 시각화

```bash
git bisect visualize
# 내부적으로 'gitk'를 실행(설치 필요)

```

또는 간단히:

```bash
git log --graph --oneline --decorate <good>..HEAD
```

---

## 고급 팁 모음

### terms 커스터마이즈

기본 용어는 `good/bad` 이지만, 필요하면 바꿔 쓸 수 있다.

```bash
git bisect terms
# good bad 출력

# 용어 변경 기능이 제공될 수 있으나
# 일반적 실무에선 good/bad를 그대로 사용하는 것을 권장

```

※ 대부분의 팀 도구/문서가 good/bad에 맞춰져 있으므로 **표준 용어 유지**가 협업에 유리하다.

### 테스트를 빠르게 만드는 요령

- **캐시**는 끄고(혼선 방지), 대신 **입력 동일성**을 보장해 반복 측정 편차를 줄인다.
- 외부 네트워크/서비스 의존을 최소화하고 **로컬 더블(mock)** 을 사용한다.
- DB 마이그레이션이 필요한 커밋을 대비해 **마이그레이션 단계**를 포함한다.

```bash
# 예: 마이그레이션 돌리고 테스트

npm run db:migrate || exit 125
npm test || exit 1
```

### 대규모 히스토리: 태그/릴리스로 good/bad 고정

- “마지막 정상 릴리스 태그”를 **good**, “현재 HEAD”를 **bad**로 시작하면 범위가 명확하다.
- 릴리스 태그를 꾸준히 달아두면 bisect 효율이 올라간다.

```bash
git bisect start HEAD v1.12.3   # v1.12.3은 정상 태그
```

### 모노레포에서의 활용

- 변경 경로 필터링(§3.2)으로 **관련 패키지 디렉터리만** 대상으로 한다.
- Nx/Turbo 등 “변경 영향도” 도구가 있다면, **대상 커밋 리스트를 사전 생성**해 스크립트에 주입하는 기법도 가능하다.

---

## CI에서의 자동 bisect(아이디어)

> 워크플로에서 재현 가능한 실패가 있고, “마지막 정상 SHA”가 기록되어 있다면, Actions에서 자동 bisect를 시도할 수 있다. 아래는 아이디어 스니펫이다(실운영 전 리소스/시간 검토 필요).

{% raw %}
```yaml
# .github/workflows/bisect.yml

name: Bisect

on:
  workflow_dispatch:
    inputs:
      good:
        description: "Last known good SHA"
        required: true
      bad:
        description: "Current bad SHA (default: HEAD)"
        required: false

jobs:
  bisect:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
        with:
          fetch-depth: 0

      - name: Prepare script
        run: |
          cat > test-script.sh <<'EOF'
          #!/usr/bin/env bash
          set -euo pipefail
          rm -rf node_modules dist
          npm ci
          npm test || exit 1
          exit 0
          EOF
          chmod +x test-script.sh

      - name: Run bisect
        run: |
          BAD=${{ github.event.inputs.bad || 'HEAD' }}
          GOOD=${{ github.event.inputs.good }}
          git bisect start "$BAD" "$GOOD"
          git bisect run ./test-script.sh || true
          git bisect reset
```
{% endraw %}

- CI 러너에서 **중간 커밋마다 깨끗한 환경**이 제공되므로 bisect 자동화에 적합하다.
- 다만 실행 시간이 길어질 수 있으니 **경로 제한/테스트 축소**를 병행하자.

---

## 문제 원인 커밋을 찾은 뒤: 다음 단계

1) **원인 커밋 diff 읽기**
   `git show <bad_commit>`

2) **재현 테스트 추가**
   해당 케이스를 커버하는 단위/통합 테스트를 작성해 회귀 방지.

3) **버그 수정 브랜치 생성**
   `git switch -c fix/regression-XXXX`
   수정 후 PR에 **원인 커밋 링크**와 **재현/수정 테스트**를 함께 첨부.

4) **릴리스 노트 반영**
   버그 영향 범위를 명시하고, 고객 영향이 있으면 패치 버전 배포.

---

## 언어별/도구별 스크립트 템플릿

### Node.js (Jest/Vitest)

```bash
#!/usr/bin/env bash

set -euo pipefail
rm -rf node_modules dist
npm ci
npm run build --if-present
npm test -- --runInBand || exit 1
exit 0
```

### Python (pytest)

```bash
#!/usr/bin/env bash

set -euo pipefail
python -m venv .venv
source .venv/bin/activate
pip install -U pip
pip install -r requirements.txt
pytest -q || exit 1
deactivate
exit 0
```

### Java (Maven)

```bash
#!/usr/bin/env bash

set -euo pipefail
mvn -q -e -DskipTests=false -B clean test || exit 1
exit 0
```

### Go

```bash
#!/usr/bin/env bash

set -euo pipefail
go clean -testcache
go test ./... -count=1 || exit 1
exit 0
```

### Windows PowerShell (예: .ps1)

```powershell
#Requires -Version 7

$ErrorActionPreference = "Stop"

if (Test-Path node_modules) { Remove-Item node_modules -Recurse -Force }
npm ci
npm test

exit 0
```

> 각 언어/환경에서 **매 커밋 동일 조건**으로 실행되도록 캐시 초기화나 의존 재설치를 포함하라.

---

## 자주 묻는 질문(FAQ)

**Q1. good/bad를 반대로 지정했다면?**
A. 걱정하지 말자. bisect는 결국 “경계”를 찾아낸다. 하지만 탐색 횟수가 늘 수 있으니 가능하면 올바르게 시작하자.

**Q2. 머지 커밋이 bad로 잡히면?**
A. 머지 자체가 문제일 수도 있고, 한쪽 부모의 변경이 문제일 수도 있다. `git show <merge>`로 부모를 확인하고, 각 부모 라인으로 **재-bisect** 하여 좁혀가자.

**Q3. 서브모듈이 있는 저장소에서?**
A. 과거 커밋으로 이동할 때 서브모듈 포인터도 바뀐다. 매 판정 전 `git submodule update --init --recursive`를 스크립트에 넣어 일관 상태를 맞추자. 실패 시 125로 skip.

**Q4. bisect 결과가 항상 같지 않다(가끔 다른 커밋이 bad로)?**
A. 플래키 테스트/비결정적(비동기 타이밍) 실행이 원인일 가능성이 크다. 재시도 로직과 안정화(고정 포트/랜덤시드, 네트워크 차단)를 추가하라.

---

## 명령어 요약 표

| 명령 | 의미 |
|------|------|
| `git bisect start` | bisect 시작 |
| `git bisect bad [<commit>]` | bad 지점 표시(생략 시 HEAD) |
| `git bisect good <commit>` | good 지점 표시 |
| `git bisect run <cmd>` | 자동 판정(0=good, 1=bad, 125=skip) |
| `git bisect skip` | 해당 커밋 판정 불가 처리 |
| `git bisect log` | 입력 로그 출력 |
| `git bisect replay <file>` | 로그 재생 |
| `git bisect visualize` | 히스토리 시각화(gitk) |
| `git bisect reset [<commit>]` | 탐색 종료 및 복귀 |

---

## 결론

- `git bisect`는 **빠르고 신뢰성 있게 원인 커밋을 찾는 표준 도구**다.
- 포인트는 **일관된 테스트·빌드 환경**, **명확한 good/bad 기준**, **skip의 적극적 활용**이다.
- 기능/성능 회귀 모두 자동화 스크립트로 처리 가능하며, **로그/재현**으로 팀 협업을 수월하게 한다.
- CI에서도 응용할 수 있으나, 실행 시간/리소스를 고려해 **경로 제한과 캐시 전략**을 병행하라.

---

## 참고

- Git 공식 문서: https://git-scm.com/docs/git-bisect
- Pro Git Book — Debugging with Git: https://git-scm.com/book/en/v2/Git-Tools-Debugging-with-Git
