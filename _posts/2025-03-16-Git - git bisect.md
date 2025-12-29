---
layout: post
title: Git - git bisect
date: 2025-03-16 21:20:23 +0900
category: Git
---
# git bisect: 버그 추적 자동화 도구

## 핵심 개념 요약

`git bisect`는 **이진 탐색(Binary Search)** 알고리즘을 활용하여 소프트웨어의 "정상 작동(good) 상태"와 "문제 발생(bad) 상태" 사이에서 **최초로 버그가 도입된 커밋**을 효율적으로 찾아주는 Git 내장 도구입니다. 커밋 히스토리를 체계적으로 절반씩 범위를 좁혀가며, 대략 **O(log₂ N)** 의 시간 복잡도로 문제의 근원을 찾을 수 있습니다. 사용 방법은 사용자가 각 커밋을 테스트한 후 결과를 직접 입력하는 **수동 모드**와, 사전에 정의된 테스트 스크립트가 자동으로 각 커밋의 상태를 판단하는 **자동 모드**(`git bisect run`)로 나뉩니다.

---

## 빠른 시작: 최소한의 절차로 따라 하기

### 수동 이진 탐색

```bash
# 1. bisect 세션 시작
git bisect start

# 2. 현재 위치(보통 HEAD)에 문제가 있음을 표시
git bisect bad

# 3. 과거의 정상 작동하던 커밋을 지점으로 지정 (해시, 태그, 브랜치 모두 가능)
git bisect good abc1234

# 4. Git이 자동으로 중간 지점의 커밋으로 체크아웃합니다.
#    해당 상태에서 프로젝트를 빌드하고 버그를 테스트합니다.
#    테스트 결과에 따라 다음 명령 중 하나를 실행하세요:
git bisect good   # 문제가 발견되지 않음
git bisect bad    # 문제가 여전히 존재함
git bisect skip   # 빌드 실패 등으로 테스트 불가

# 5. 위 과정을 반복하면 Git이 최종적으로 최초의 bad 커밋을 찾아 알려줍니다.
#    출력 예: abcd1234 is the first bad commit

# 6. bisect 세션을 종료하고 원래 작업 브랜치로 돌아갑니다.
git bisect reset
```

### 자동 탐색 (테스트 스크립트 활용)

`git bisect run` 명령은 테스트 스크립트의 **종료 코드(Exit Code)** 를 기반으로 각 커밋의 상태를 자동 판단합니다.

```bash
# 1. bisect 세션을 시작하고 good/bad 범위 설정
git bisect start
git bisect bad
git bisect good abc1234

# 2. 테스트 스크립트를 실행하여 자동 탐색 시작
git bisect run ./test-script.sh
```

스크립트의 종료 코드 해석:
- **0**: 성공 → `good`으로 판정
- **1~127** (단, 125 제외): 실패 → `bad`로 판정
- **125**: 테스트 불가 → `skip`으로 판정 (이 커밋은 탐색에서 제외)

간단한 테스트 스크립트 예시 (`test-script.sh`):
```bash
#!/usr/bin/env bash
set -euo pipefail

# 각 커밋에서 공정한 비교를 위해 깨끗한 빌드 환경을 보장합니다.
rm -rf node_modules dist
npm ci
npm test  # 테스트가 통과하면 종료 코드 0, 실패하면 1이 반환됩니다.
```

---

## 상세 사용 흐름: 실전 패턴

### 시작과 종료 단축 명령

```bash
# start, bad, good 지정을 한 줄로 처리
git bisect start <bad_commit> <good_commit>
# 예: 현재 HEAD가 bad이고, v1.4.0 태그 시점은 good일 때
git bisect start HEAD v1.4.0

# 탐색 종료 후 특정 지점(예: main 브랜치)으로 바로 돌아가기
git bisect reset main
```

### 탐색 범위 최소화: 특정 경로만 대상으로 하기

버그가 특정 디렉터리나 파일에만 영향을 준다면, 해당 경로에 변경이 있는 커밋만 탐색 대상으로 삼으면 효율이 크게 향상됩니다.

```bash
git bisect start
git bisect bad
git bisect good abc1234

# 'src/auth/' 디렉터리와 관련된 변경만 있는 커밋을 대상으로 자동 탐색
git bisect run bash -lc '
  # 현재 커밋이 src/auth 디렉터리를 변경하지 않았다면 skip(125) 처리
  if git diff --quiet HEAD^ -- src/auth; then
    exit 125
  fi

  # 변경이 있다면 테스트 수행
  rm -rf node_modules dist
  npm ci
  npm test -- -t "auth"
'
```

### 복잡한 병합 히스토리 다루기

`good`과 `bad` 사이에 여러 병합 커밋이 있는 등 비선형 경로가 많으면, 불필요한 커밋까지 탐색에 포함될 수 있습니다. 이 경우 **직계 조상 경로(ancestry path)** 만을 따라가도록 제한할 수 있습니다.

```bash
# good..bad 범위에서 직계 조상 경로에 있는 커밋 해시만 나열
git rev-list --ancestry-path good..bad
# 이 목록을 활용하여 수동 bisect를 진행하거나, 스크립트에서 탐색 대상을 필터링할 수 있습니다.
```

---

## 예외 상황 처리: 빌드 실패, 충돌, 불안정한 테스트

### 빌드 불가 또는 외부 의존성 문제

오래된 커밋은 현재의 빌드 도구 체인과 호환되지 않아 실패할 수 있습니다. 이런 경우 해당 커밋을 탐색에서 제외(`skip`)해야 합니다.

```bash
# 테스트 스크립트 내부 예시
if ! make build; then
  exit 125    # 빌드 실패 → 이 커밋은 skip
fi
make test || exit 1  # 테스트 실패 → bad
exit 0               # 테스트 성공 → good
```

### 병합 충돌

Bisect 과정 중 특정 커밋으로 체크아웃할 때 병합 충돌이 발생할 수 있습니다. 수동 해결이 어렵다면 `git bisect skip`으로 넘어갑니다. 충돌이 빈번하다면 탐색 범위를 축소하는 것이 좋습니다.

### 불안정한(Flaaky) 테스트

간헐적으로 실패하는 테스트는 오탐을 유발할 수 있습니다. 스크립트 내에서 재시도 로직을 추가하여 안정성을 높일 수 있습니다.

```bash
#!/usr/bin/env bash
set -euo pipefail

max_retries=3
for ((retry=0; retry<max_retries; retry++)); do
  if npm test; then
    exit 0  # 한 번이라도 성공하면 good
  fi
  sleep 1
done
exit 1  # 모든 재시도 후에도 실패하면 bad
```

---

## 성능 회귀 탐지에도 활용하기

`git bisect`는 기능 결함뿐만 아니라 **성능 저하(Performance Regression)** 가 처음 발생한 커밋을 찾는 데도 사용할 수 있습니다. 성능 측정값과 기준 임계값(Threshold)을 비교하여 `good`/`bad`를 판정하면 됩니다.

### API 응답 지연 회귀 탐지 예시

```bash
#!/usr/bin/env bash
set -euo pipefail

# 1. 애플리케이션 빌드 및 실행
docker compose build --no-cache api
docker compose up -d api
sleep 2  # 서버 준비 대기

# 2. 성능 측정 (예: health 엔드포인트 응답 시간)
response_time_ms=$( (time -p curl -sS http://localhost:3000/health >/dev/null) 2>&1 \
      | awk '/real/ {print $2 * 1000}' )

# 3. 리소스 정리
docker compose down -v

# 4. 임계값(120ms)과 비교하여 판정
threshold=120
if (( $(echo "$response_time_ms > $threshold" | bc -l) )); then
  exit 1  # 성능 회귀 발생 → bad
else
  exit 0  # 성능 정상 → good
fi
```

이 스크립트를 `git bisect run`에 연결하면 성능이 기준치를 초과하게 만든 최초의 커밋을 자동으로 찾을 수 있습니다. 핵심은 **모든 커밋에서 동일한 방식으로 빌드, 실행, 측정을 수행**하여 공정한 비교를 보장하는 것입니다.

---

## 로그 기록, 재현, 팀 협업

### 탐색 로그 저장 및 재생

Bisect 과정을 로그 파일로 저장하면, 동료가 동일한 환경에서 정확히 같은 절차를 재현하여 문제를 검증할 수 있습니다.

```bash
# 현재 bisect 세션의 모든 입력(good/bad/skip)을 파일로 저장
git bisect log > bisect.log

# 다른 환경에서 로그 파일을 재생하여 동일한 탐색 수행
git bisect replay bisect.log
```

### 히스토리 시각화

`git bisect visualize` 명령은 `gitk` 도구를 실행하여 현재 탐색 범위와 커밋 그래프를 시각적으로 확인할 수 있게 합니다. `gitk`가 설치되어 있어야 합니다. 또는 다음 명령으로 간단히 그래프를 볼 수 있습니다.

```bash
git log --graph --oneline --decorate <good_commit>..HEAD
```

---

## 고급 사용 팁 모음

### 빠르고 안정적인 테스트 환경 구축 요령

- **캐시 비활성화**: 각 커밋 테스트 시 캐시를 초기화하여 일관된 결과를 보장하세요. 대신 입력(예: 테스트 데이터)의 동일성을 유지하세요.
- **외부 의존성 최소화**: 네트워크 요청이나 외부 서비스에 의존하는 테스트는 비결정적 결과를 초래할 수 있습니다. 가능하면 목(Mock) 객체나 더블(Double)을 사용하세요.
- **데이터베이스 마이그레이션 고려**: 과거 커밋으로 이동할 때 필요한 DB 스키마 변경이 있을 수 있습니다. 테스트 전 마이그레이션 스크립트를 실행하도록 스크립트에 포함하세요.
    ```bash
    npm run db:migrate || exit 125  # 마이그레이션 실패 시 skip
    npm test || exit 1
    ```

### 대규모 히스토리에서 효율적으로 시작하기

릴리스 태그를 잘 관리하고 있다면, 탐색 시작점을 설정하는 데 큰 도움이 됩니다.
```bash
# "마지막으로 정상 작동한 릴리스" 태그를 good으로, 현재 HEAD를 bad로 설정
git bisect start HEAD v1.12.3
```

### 모노레포(Monorepo)에서의 활용

모노레포에서는 특정 패키지의 변경만 관련된 버그가 많습니다. **경로 필터링**을 활용해 해당 패키지 디렉터리만을 대상으로 bisect를 실행하면 효율적입니다. Nx, Turborepo 같은 도구가 있다면 변경 영향도를 분석해 탐색 대상을 사전에 추릴 수도 있습니다.

---

## CI 환경에서의 자동화 (개념 예시)

재현 가능한 실패 케이스와 마지막 정상 커밋 정보가 있다면, GitHub Actions 같은 CI 시스템에서 bisect를 자동화할 수 있습니다. 실행 시간과 리소스 사용량을 고려하여 신중하게 설계해야 합니다.

{% raw %}
```yaml
# .github/workflows/bisect.yml
name: Auto Bisect
on:
  workflow_dispatch:  # 수동 트리거
    inputs:
      good_sha:
        description: "Last known good commit SHA"
        required: true
      bad_sha:
        description: "Current bad commit SHA (default: HEAD)"
        required: false
        default: "HEAD"

jobs:
  find-regression:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
        with:
          fetch-depth: 0  # 전체 히스토리 필요

      - name: Set up test script
        run: |
          cat > test.sh << 'EOF'
          #!/bin/bash
          set -euxo pipefail
          # 여기에 프로젝트의 빌드/테스트 명령을 작성하세요.
          npm ci
          npm run build
          npm test
          EOF
          chmod +x test.sh

      - name: Run git bisect
        run: |
          BAD_SHA="${{ github.event.inputs.bad_sha }}"
          GOOD_SHA="${{ github.event.inputs.good_sha }}"
          git bisect start "$BAD_SHA" "$GOOD_SHA"
          git bisect run ./test.sh
          git bisect log  # 결과 로그 출력
          git bisect reset
```
{% endraw %}

---

## 버그 원인 커밋을 찾은 후의 작업

1.  **원인 분석**: `git show <bad_commit>` 명령으로 해당 커밋의 정확한 변경 사항을 검토합니다.
2.  **재현 테스트 작성**: 발견된 버그를 명시적으로 재현하고 검증하는 단위 또는 통합 테스트를 작성합니다. 이는 동일한 회귀가 미래에 다시 발생하는 것을 방지합니다.
3.  **수정 브랜치 생성**: `git switch -c fix/regression-description` 명령으로 새 브랜치를 만들어 버그를 수정합니다. PR을 생성할 때는 원인 커밋에 대한 링크와 새로 작성한 재현 테스트를 포함시키세요.
4.  **변경 사항 문서화**: 버그의 영향 범위를 평가하고, 필요하다면 릴리스 노트에 반영하거나 고객에게 통보합니다.

---

## 언어별 테스트 스크립트 템플릿

### Node.js (Jest)
```bash
#!/usr/bin/env bash
set -euo pipefail
rm -rf node_modules dist
npm ci
npm run build --if-present
npm test -- --runInBand --passWithNoTests || exit 1
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
pytest -xvs || exit 1  # -x: 첫 실패 시 중지, -v: 상세 출력, -s: 출력 표시
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
go clean -testcache  # 테스트 캐시 초기화
go test ./... -count=1 -v || exit 1  # -count=1: 캐시 무시
exit 0
```

### Windows PowerShell (.ps1)
```powershell
#Requires -Version 7
$ErrorActionPreference = "Stop"
if (Test-Path node_modules) { Remove-Item node_modules -Recurse -Force }
npm ci
npm test
exit 0
```

> 모든 스크립트의 공통 목표는 **각 커밋을 동일한 조건에서 테스트**할 수 있도록 환경을 일관되게 초기화하는 것입니다.

---

## 자주 묻는 질문 (FAQ)

**Q1. `good`과 `bad`를 반대로 지정해서 시작했습니다. 어떻게 해야 하나요?**
A. 걱정하지 마세요. Bisect는 궁극적으로 "상태가 변하는 경계"를 찾는 도구입니다. `good`과 `bad`가 뒤바뀌어도 경계는 찾을 수 있지만, 탐색 횟수가 불필요하게 증가할 수 있습니다. 가능하면 정확히 지정하여 시작하는 것이 좋습니다.

**Q2. 병합(Merge) 커밋이 문제의 원인(bad)으로 지목되었습니다. 어떻게 해야 하나요?**
A. 병합 커밋 자체에 문제가 있을 수도 있고, 병합된 두 브랜치 중 한쪽의 변경 사항에 문제가 있을 수도 있습니다. `git show <merge_commit>`으로 부모 커밋을 확인한 후, 각 부모 브랜치에 대해 별도로 bisect를 실행하여 문제의 정확한 출처를 좁혀나가는 방법을 고려해 보세요.

**Q3. 서브모듈이 있는 저장소에서 bisect를 사용할 때 주의점은 무엇인가요?**
A. Bisect로 과거 커밋으로 이동하면 서브모듈의 포인터(참조 커밋)도 함께 변경됩니다. 테스트 스크립트 시작 부분에 `git submodule update --init --recursive` 명령을 추가하여 서브모듈을 올바른 버전으로 동기화하세요. 초기화에 실패하면 `exit 125`로 해당 커밋을 건너뛸 수 있습니다.

**Q4. 동일한 버그에 대해 bisect를 여러 번 실행했는데 매번 다른 커밋이 원인으로 나옵니다.**
A. 이는 **불안정한(Flaaky) 테스트**나 **비결정적(Non-deterministic)인 코드 실행**(예: 경쟁 조건, 네트워크 타이밍) 때문에 발생할 가능성이 높습니다. 테스트 스크립트에 재시도 로직을 추가하거나, 테스트 환경을 안정화(고정 포트 사용, 난수 시드 고정, 네트워크 격리)하는 방향으로 개선해 보세요.

---

## 핵심 명령어 요약

| 명령 | 설명 |
|------|------|
| `git bisect start` | Bisect 세션을 시작합니다. |
| `git bisect bad [<commit>]` | 지정된 커밋(생략 시 현재 HEAD)에 문제가 있음을 표시합니다. |
| `git bisect good <commit>` | 지정된 커밋이 정상 작동함을 표시합니다. |
| `git bisect run <command>` | 주어진 명령어/스크립트를 실행하여 각 커밋을 자동으로 테스트합니다. |
| `git bisect skip` | 현재 커밋을 테스트할 수 없어 판정에서 제외합니다. |
| `git bisect log` | 현재까지의 bisect 입력 기록을 출력합니다. |
| `git bisect replay <file>` | 파일에 저장된 bisect 로그를 재생하여 동일한 탐색을 수행합니다. |
| `git bisect visualize` | `gitk`를 통해 현재 탐색 범위를 그래픽으로 표시합니다. |
| `git bisect reset [<commit>]` | Bisect 세션을 종료하고 지정된 커밋(생략 시 원래 위치)으로 돌아갑니다. |

---

## 결론

`git bisect`는 소프트웨어 개발에서 버그나 성능 회귀의 근본 원인을 체계적이고 효율적으로 찾아주는 강력한 도구입니다. 성공적인 사용의 핵심은 **재현 가능하고 일관된 테스트 환경을 구성**하는 것, **정상(good)과 비정상(bad) 상태를 명확히 정의**하는 것, 그리고 빌드 실패 등으로 인한 방해 요소를 `skip`으로 적절히 처리하는 것에 있습니다.

자동화 모드(`git bisect run`)를 활용하면 이 과정을 완전히 스크립트화하여, CI/CD 파이프라인에 통합하거나 반복적인 디버깅 작업에서 큰 시간을 절약할 수 있습니다. 탐색 로그(`git bisect log`)를 저장하고 재생하는 기능은 팀 내 협업과 문제 재현을 수월하게 만들어 줍니다.

처음에는 작은 범위에서 수동으로 시작해 보는 것을 권장합니다. 도구에 익숙해지고 테스트 스크립트를 견고하게 만들수록, 복잡하고 오래된 버그도 자신 있게 추적해 낼 수 있을 것입니다. `git bisect`는 단순한 명령어가 아니라, 소프트웨어의 시간적 변화를 이해하고 제어하는 데 유용한 강력한 방법론임을 기억하세요.
