---
layout: post
title: Git - Git Submodule
date: 2025-02-19 19:20:23 +0900
category: Git
---
# Git Submodule

## 서브모듈이란 무엇인가?

- **서브모듈(Submodule)**은 Git 저장소 내부에 **다른 독립적인 Git 저장소를 디렉터리 형태로 포함**시키는 기능입니다.
- 상위 저장소는 서브모듈의 전체 코드를 저장하는 것이 아니라, 해당 서브모듈이 **가리키는 특정 커밋 해시(정확한 버전)**만을 기록합니다.
- **주요 장점**: 빌드 환경의 **정밀한 재현성 보장(버전 고정)**, 코드의 재사용성, 조직/팀 간의 코드 경계와 권한 유지.
- **주의 사항**: 초기화, 업데이트, 권한 관리, CI/CD 통합이 **명시적이고 수동적인 작업**을 필요로 합니다.

---

## 서브모듈을 언제 사용해야 할까?

서브모듈은 다음 상황에서 유용한 선택이 될 수 있습니다.

- 여러 프로젝트가 공유하는 **내부 공용 라이브러리**를 독립된 저장소로 관리하며, **별도의 릴리스 주기와 권한 체계**를 유지하고 싶을 때.
- **외부 오픈소스 프로젝트**의 특정 버전(커밋 또는 태그)을 프로젝트에 정확히 고정하여, 언제나 동일한 코드로 빌드되는 **재현성을 극대화**하고 싶을 때.
- 포함된 코드의 **변경 주기, 접근 권한, 배포 절차**를 상위 프로젝트와 분리하여 관리해야 할 때 (예: 보안상 이유 또는 라이선스 경계).

> **대안 고려사항**: 상위 저장소에 외부 코드를 **실제로 복사하여 통합**하고, 동기화를 보다 단순하게 관리하고 싶다면 **Git Subtree**를 검토해 보는 것이 좋습니다. 두 방식의 비교는 아래 §13을 참고하세요.

---

## 기본 사용법 — 추가, 커밋, 클론, 업데이트

### 1. 서브모듈 추가하기

```bash
git submodule add https://github.com/my-org/common-ui.git libs/common-ui
```

이 명령을 실행하면:
- 상위 저장소에 `.gitmodules` 파일이 생성됩니다.
- `libs/common-ui` 디렉터리가 **별도의 Git 저장소**로 초기화됩니다.
- 상위 저장소는 `libs/common-ui`가 현재 가리키고 있는 **정확한 커밋 해시**를 기록합니다.

### 2. 변경사항 커밋 및 푸시

```bash
git add .gitmodules libs/common-ui
git commit -m "Add common-ui as submodule"
git push origin main
```

생성된 `.gitmodules` 파일 예시:
```ini
[submodule "libs/common-ui"]
  path = libs/common-ui
  url = https://github.com/my-org/common-ui.git
```

### 3. 서브모듈이 포함된 저장소 클론하기

서브모듈을 포함한 저장소를 받을 때는 두 가지 방법이 있습니다.

**방법 A: 클론 시 한 번에 모든 서브모듈 받기**
```bash
git clone --recurse-submodules https://github.com/my-org/my-repo.git
```

**방법 B: 나중에 서브모듈 초기화하기**
```bash
git clone https://github.com/my-org/my-repo.git
cd my-repo
git submodule update --init --recursive
```
`--recursive` 옵션은 **서브모듈 내부에 또 다른 서브모듈이 있는 경우(중첩)**까지 모두 초기화합니다.

### 4. 서브모듈 상태 확인 및 업데이트

```bash
git submodule status                   # 현재 고정된(Pinned) 커밋 해시 확인
git submodule update --remote          # .gitmodules에 정의된 브랜치의 최신 커밋으로 업데이트 (설정된 경우)
git submodule update --init --recursive # 누락된 서브모듈 초기화 및 업데이트
```

> **핵심 이해**: 기본적으로 서브모듈은 특정 브랜치를 자동으로 따라가지 않고, **고정된 커밋 해시를 가리킵니다**. 최신 코드로 업데이트하려면 명시적인 작업이 필요합니다.

---

## 서브모듈의 내부 동작 방식

서브모듈은 상위 저장소의 Git 트리에서 해당 디렉터리를 **특정 커밋 해시를 가리키는 "링크" 또는 "특별한 엔트리"**로 관리합니다. 상위 저장소의 각 커밋에는 "서브모듈 경로 → 커밋 해시"라는 매핑 정보가 포함됩니다.

따라서 상위 저장소를 특정 커밋으로 체크아웃하면, 서브모듈 디렉터리가 올바른 버전의 코드를 가리키도록 하기 위해 **명시적인 `git submodule update` 명령이 필요**합니다.

이 구조는 빌드 환경의 정확한 재현성을 보장하지만, CI/CD 파이프라인의 복잡성과 시간을 증가시킬 수 있습니다. 서브모듈이 많을수록 초기화 및 업데이트에 필요한 네트워크 비용과 시간이 선형적으로 증가하므로, 캐싱 전략이나 부분 체크아웃을 고려해야 합니다.

---

## 서브모듈 버전 업데이트 (핀 이동) 표준 절차

상위 프로젝트에서 서브모듈을 더 새로운 버전으로 업데이트하려면 다음 단계를 따릅니다.

```bash
# 1. 서브모듈 디렉터리로 이동
cd libs/common-ui

# 2. 원하는 기준으로 코드 업데이트 (예: main 브랜치의 최신 커밋, 특정 태그)
git fetch origin
git checkout main
git pull origin main

# 3. 상위 저장소 디렉터리로 돌아와서 변경사항을 커밋
cd ../..
git add libs/common-ui       # 서브모듈이 가리키는 새로운 커밋 해시를 스테이징
git commit -m "Bump common-ui to <new-commit-hash>"
git push
```

> **핵심**: 서브모듈 자체의 코드를 수정했다면, **해당 서브모듈 저장소에 별도로 `git push`를 수행**해야 합니다. 상위 저장소는 단지 "어느 커밋을 참조하는지"에 대한 정보만을 관리합니다.

---

## `.gitmodules` 파일의 고급 설정

`.gitmodules` 파일을 통해 서브모듈의 동작을 세밀하게 제어할 수 있습니다.

```ini
[submodule "libs/common-ui"]
  path = libs/common-ui
  url = https://github.com/my-org/common-ui.git
  branch = main         # 추적할 브랜치 지정 (선택 사항)
  update = rebase       # 업데이트 방식: rebase, merge, checkout (선택 사항)
  shallow = true        # 얕은 클론(Shallow Clone) 사용 (선택 사항, Git 2.9+)
```

### `branch = <브랜치명>`
- `git submodule update --remote` 명령을 실행할 때, 지정한 브랜치의 최신 커밋을 자동으로 가져와 서브모듈의 포인터를 이동시킵니다.
- **주의**: 이 설정이 있다고 해서 상위 저장소의 포인터가 자동으로 갱신되는 것은 아닙니다. `update --remote` 후 반드시 상위 저장소에서 변경사항을 커밋해야 합니다.
    ```bash
    git submodule update --remote --recursive
    git add libs/common-ui
    git commit -m "Track common-ui to latest main"
    ```

### `update = rebase|merge|checkout`
- `git submodule update` 명령이 로컬 변경사항을如何处理할지 결정합니다.
- 기본값은 `checkout`으로, 서브모듈 디렉터리의 모든 로컬 변경사항을 덮어씁니다. 로컬 커밋을 유지하고 싶다면 `rebase`나 `merge`를 고려하세요.

### `shallow = true`
- 서브모듈을 얕은 클론(최근 커밋만 다운로드)하여 CI 시간과 네트워크 대역폭을 절약합니다.
- 서브모듈의 전체 히스토리가 필요하지 않은 경우 유용합니다.

---

## 팀 협업을 위한 워크플로우

### 시나리오 1: 상위 저장소만 변경 (서브모듈 포인터 업데이트)
1.  개발자는 상위 저장소에서 서브모듈이 참조하는 **커밋 해시만을 변경**하여 PR을 생성합니다.
2.  리뷰어는 `git submodule status` 명령이나 GitHub UI를 통해 **어느 커밋으로 변경되는지 확인**합니다.
3.  CI 시스템은 서브모듈을 포함한 전체 코드를 체크아웃하여 빌드와 테스트를 수행합니다.
4.  PR이 병합되면 상위 저장소의 서브모듈 포인터가 새로운 커밋 해시로 갱신됩니다.

### 시나리오 2: 하위(서브모듈)와 상위 저장소 모두 변경
1.  먼저 **서브모듈 저장소에 PR을 생성하고 병합**합니다 (필요시 태그 생성).
2.  그런 다음 **상위 저장소에서 서브모듈의 새 커밋/태그를 가리키도록 포인터를 업데이트하는 PR**을 생성합니다.
3.  두 변경사항 간에 의존성이 있다면, **체인드 PR(Chained PR)** 전략이나 **멀티-리포지토리 동시 릴리스** 관리 도구를 고려해야 합니다.

---

## CI/CD 통합 — GitHub Actions 예제

### 기본: 재귀적 서브모듈 체크아웃
```yaml
# .github/workflows/ci.yml
name: CI
on: [push, pull_request]
jobs:
  build:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
        with:
          submodules: recursive     # 모든 서브모듈을 재귀적으로 체크아웃
          fetch-depth: 0            # 필요한 경우 전체 히스토리 가져오기

      - name: Verify submodule pointers
        run: git submodule status

      - name: Build & Test
        run: |
          npm ci
          npm run build
          npm test -- --ci
```

### 최적화: 필요한 서브모듈만 부분 체크아웃 (성능/권한 제약 시)
```yaml
- uses: actions/checkout@v4
  with:
    submodules: false  # 서브모듈 체크아웃 비활성화

- name: Initialize only required submodules
  run: |
    # 특정 서브모듈만 초기화
    git submodule update --init libs/common-ui
    # 얕은 클론 및 특정 브랜치 체크아웃 (선택 사항)
    git -C libs/common-ui fetch --depth=1 origin main
    git -C libs/common-ui checkout FETCH_HEAD
```

### 캐싱 전략
서브모듈은 독립된 저장소이므로, **의존성 캐시**도 각각 별도로 관리해야 효율적입니다. Node.js의 `node_modules`, Java의 `.gradle`/`.m2` 디렉터리 캐시와 함께, **빌드된 아티팩트 자체를 캐싱**하는 전략을 결합하면 CI 실행 시간을 크게 줄일 수 있습니다.

---

## 권한 관리: 프라이빗 서브모듈과 인증

- **프라이빗 서브모듈**을 체크아웃하려면 별도의 인증이 필요합니다.
- **GitHub Actions**에서 **동일 조직 내의 프라이빗 서브모듈**은 `actions/checkout@v4` 액션이 자동으로 `GITHUB_TOKEN`을 사용하여 처리할 수 있습니다 (저장소 권한 설정 확인 필요).
- 서로 다른 조직 또는 Git 호스트에 있는 서브모듈의 경우, **Deploy Key**, **Personal Access Token**, **SSH 키**를 구성해야 합니다.
    ```yaml
    - uses: actions/checkout@v4
      with:
        submodules: recursive
        ssh-key: ${{ secrets.SUBMODULE_DEPLOY_KEY }} # SSH 키를 통한 인증
    ```

---

## 서브모듈 관리: 삭제, 경로 변경, URL 변경

### 서브모듈 삭제 (정식 절차)
서브모듈을 제거할 때는 관련된 모든 설정을 깔끔하게 정리해야 합니다.
```bash
# 1. 서브모듈 초기화 해제 및 작업 디렉터리 정리
git submodule deinit -f libs/common-ui

# 2. 상위 저장소에서 서브모듈 디렉터리 제거
git rm -f libs/common-ui

# 3. (선택) .git/modules/ 내부 캐시 제거
rm -rf .git/modules/libs/common-ui

# 4. 변경사항 커밋
git commit -m "Remove submodule common-ui"
```
`.gitmodules` 파일에서 해당 섹션이 자동으로 제거됩니다. 수동으로 `.git/config` 파일을 정리하지 않아도 됩니다.

### 서브모듈 경로 변경
```bash
# 1. 디렉터리 이름 변경
git mv libs/common-ui libs/ui

# 2. .gitmodules 파일의 경로 정보 업데이트
git config -f .gitmodules submodule.libs/common-ui.path libs/ui

# 3. 변경사항을 내부 설정에 동기화
git submodule sync

# 4. 변경사항 커밋
git add .gitmodules
git commit -m "Move submodule to libs/ui"
```

### 서브모듈 원격 URL 변경
```bash
# 1. .gitmodules 파일의 URL 정보 업데이트
git config -f .gitmodules submodule.libs/ui.url git@github.com:my-org/ui.git

# 2. 변경사항을 내부 설정에 동기화
git submodule sync

# 3. 새 URL에서 서브모듈 초기화
git submodule update --init --recursive

# 4. 변경사항 커밋
git add .gitmodules
git commit -m "Point submodule to new remote"
```

---

## 자주 사용하는 명령어 요약 (치트 시트)

```bash
### 상태 확인 및 초기화 ###
git submodule status                 # 각 서브모듈의 현재 커밋 해시 확인
git submodule init                   # .gitmodules에 정의된 서브모듈 초기화
git submodule update                 # 서브모듈을 기록된 커밋 해시로 업데이트
git submodule update --init --recursive # 초기화와 업데이트를 한 번에 수행

### 브랜치 추적 및 업데이트 ###
git submodule update --remote        # .gitmodules의 branch 설정에 따라 최신 커밋 가져오기
git config -f .gitmodules submodule.libs/common-ui.branch main # 브랜치 추적 설정
git add .gitmodules && git commit -m "Track branch for common-ui"

### 모든 서브모듈에 명령 실행 ###
git submodule foreach 'git status'   # 각 서브모듈에서 git status 실행
git submodule foreach --recursive 'git pull --rebase --autostash' # 재귀적으로 풀

### 삭제 ###
git submodule deinit -f libs/common-ui
git rm -f libs/common-ui
rm -rf .git/modules/libs/common-ui

### 설정 동기화 (경로/URL 변경 후) ###
git submodule sync --recursive
```

---

## 중첩 서브모듈 (Nested Submodules) 및 모노레포에서의 사용

- **중첩 서브모듈**: A(상위) → B(서브모듈) → C(서브모듈)와 같은 구조입니다. `--recursive` 옵션으로 한 번에 관리할 수 있지만, 권한, 네트워크 비용, 복잡성이 증가합니다.
- **모노레포와 서브모듈**: 모노레포는 여러 패키지를 단일 저장소에서 관리하는 반면, 서브모듈은 외부 저장소를 링크합니다. 모노레포 내부에서 서브모듈을 사용하면 **모노레포의 통합된 CI/CD 파이프라인과 공용 규약**이 서브모듈의 독립성과 충돌할 수 있습니다. 가능하다면 **Subtree**나 **사내 패키지 레지스트리(npm, Maven 등)**를 대안으로 고려하세요.

---

## 문제 해결 (트러블슈팅)

| 증상 | 원인 | 해결 방법 |
|---|---|---|
| 서브모듈 디렉터리가 비어 있음 | 서브모듈이 초기화되지 않음 | `git submodule update --init --recursive` 실행 |
| 프라이빗 서브모듈 접근 권한 오류 | CI/로컬 환경에 인증 정보 없음 | HTTPS 토큰 또는 SSH 키 설정. GitHub Actions에서는 `ssh-key` 또는 `token` 파라미터 사용 |
| 서브모듈 업데이트 실패 (로컬 변경 존재) | 서브모듈 디렉터리에 커밋되지 않은 변경사항이 있어 체크아웃 방해 | `git -C libs/common-ui stash`로 변경사항 임시 저장 후 `git submodule update` 실행. 또는 `.gitmodules`에서 `update=rebase` 정책 사용 |
| 상위 저장소의 포인터는 변경됐으나 빌드 시 예전 코드 사용 | CI 워크플로우가 서브모듈을 체크아웃하지 않음 | GitHub Actions에서 `submodules: recursive` 설정 확인. 또는 수동 `git submodule update --init` 추가 |
| 경로/URL 변경 후 동기화 안 됨 | `git submodule sync` 명령 누락 | 변경 후 반드시 `git submodule sync --recursive` 실행 |
| 의존성 캐시가 자주 무효화됨 | 서브모듈의 커밋 해시가 빈번하게 변경 | 서브모듈의 릴리스 주기 재검토. 빌드 아티팩트 캐싱으로 보완 |

---

## Git Submodule vs. Git Subtree 비교

| 항목 | **Submodule** | **Subtree** |
|---|---|---|
| 저장 방식 | **링크(참조)**만 상위 저장소에 기록 | **실제 코드가 상위 저장소에 복사/통합**됨 |
| 외부성/권한 | 원본 저장소와의 **완전한 분리** 유지 | 상위 저장소의 **일부로 흡수**됨 |
| 업데이트 | 명시적인 `git submodule update` 필요 | `git subtree pull/push`로 상위 저장소 내에서 관리 |
| 히스토리 재현성 | **매우 강력** (정확한 커밋 해시 고정) | 코드는 통합되나, 원본 출처 추적이 추가 설명 필요 |
| 학습/운영 난이도 | 상대적으로 **높음** (개념과 명령어 이해 필요) | **낮음** (표준 Git 명령어와 유사) |
| CI/네트워크 부하 | 서브모듈 수만큼 **별도 클론 필요** | 상위 저장소 크기 증가 가능성 |
| **권장 사용처** | **코드 경계, 권한, 릴리스 주기를 철저히 분리해야 할 때** | **단일 저장소로 단순화하여 운영하고 싶을 때** |

**변환 팁**: Submodule에서 Subtree로 전환하려면, `git subtree add --prefix=libs/common-ui <url> <ref>` 명령으로 코드를 가져온 후, 위의 서브모듈 삭제 절차를 수행하세요.

---

## 실전 시나리오 예제

### 시나리오 1: 공용 UI 라이브러리를 여러 프론트엔드 서비스에서 사용

**프로젝트 구조**
```
app-frontend/          # 메인 프론트엔드 애플리케이션 저장소
└── libs/common-ui     # 서브모듈 (my-org/common-ui 저장소를 가리킴)
```

**서브모듈 추가**
```bash
cd app-frontend
git submodule add https://github.com/my-org/common-ui.git libs/common-ui
git add .gitmodules libs/common-ui
git commit -m "Add common-ui as submodule"
```

**일상적인 업데이트 (수동)**
```bash
# common-ui 저장소의 최신 main 브랜치로 업데이트
git -C libs/common-ui fetch origin
git -C libs/common-ui checkout main
git -C libs/common-ui pull origin main

# 상위 저장소에 새로운 포인터 커밋
git add libs/common-ui
git commit -m "Bump common-ui to latest main"
git push
```

**CI 설정 (GitHub Actions)**
```yaml
- uses: actions/checkout@v4
  with:
    submodules: recursive
- run: npm ci && npm run build && npm test
```

### 시나리오 2: 외부 오픈소스 라이브러리 특정 버전 고정
```bash
git submodule add https://github.com/some/oss.git vendor/oss
git -C vendor/oss fetch --tags
git -C vendor/oss checkout v2.3.1
git add vendor/oss
git commit -m "Pin oss library to v2.3.1"
```

### 시나리오 3: 브랜치 추적 자동화 설정
`.gitmodules` 파일 설정:
```ini
[submodule "libs/common-ui"]
  path = libs/common-ui
  url = https://github.com/my-org/common-ui.git
  branch = main   # main 브랜치를 추적
```

작업 명령:
```bash
# 설정된 브랜치의 최신 커밋으로 모든 서브모듈 업데이트
git submodule update --remote --recursive

# 상위 저장소에 변경사항 커밋
git add libs/common-ui
git commit -m "Track common-ui: update to latest main"
```

---

## 성능 및 안정성 최적화 가이드

- **서브모듈 수 최소화**: 서브모듈이 많을수록 관리 비용과 CI 시간이 선형적으로 증가합니다.
- **얕은 클론(Shallow Clone) 활용**: `shallow = true` 설정 또는 `--depth 1` 옵션으로 불필요한 히스토리 다운로드를 방지합니다.
- **필요한 서브모듈만 부분 초기화**: CI 파이프라인에서 모든 서브모듈이 필요하지 않다면, 필요한 것만 선택적으로 초기화하세요.
- **릴리스 태그 사용**: 서브모듈을 항상 특정 릴리스 태그에 고정하면 버전 관리와 재현성이 예측 가능해집니다.
- **권한 및 토큰 전략 문서화**: 새로운 팀원이나 CI 시스템이 프라이빗 서브모듈에 쉽게 접근할 수 있도록 가이드를 마련하세요.

---

## 전체 작업 흐름 명령어 요약

```bash
### 서브모듈 추가 ###
git submodule add <url> path/to/sub
git add .gitmodules path/to/sub
git commit -m "Add submodule"

### 저장소 클론 및 서브모듈 초기화 ###
git clone --recurse-submodules <root-repo-url>
# 또는
git clone <root-repo-url>
cd <repo>
git submodule update --init --recursive

### 브랜치 추적을 통한 업데이트 ###
git submodule update --remote --recursive

### 수동 포인터 업데이트 ###
git -C path/to/sub fetch origin
git -C path/to/sub checkout <branch-or-tag>
git -C path/to/sub pull --ff-only origin <branch>
git add path/to/sub
git commit -m "Bump submodule"
git push

### 서브모듈 삭제 ###
git submodule deinit -f path/to/sub
git rm -f path/to/sub
rm -rf .git/modules/path/to/sub
git commit -m "Remove submodule"

### 설정 동기화 ###
git submodule sync --recursive
```

---

## 자주 묻는 질문 (FAQ)

**Q1. 서브모듈은 왜 항상 특정 '브랜치'가 아니라 '커밋'을 가리키나요?**
A. 상위 프로젝트의 빌드 재현성을 보장하기 위해서입니다. 브랜치는 시간에 따라 이동하는 포인터이므로, 특정 시점의 정확한 상태를 보장할 수 없습니다. 커밋 해시를 고정함으로써 항상 동일한 코드를 참조할 수 있습니다. 브랜치를 자동으로 추적하고 싶다면 `.gitmodules`의 `branch` 설정과 `git submodule update --remote` 명령을 함께 사용할 수 있지만, 최종적으로는 상위 저장소에서 새로운 커밋 해시를 커밋해야 합니다.

**Q2. 로컬에서 서브모듈 코드를 수정했는데, 상위 저장소만 푸시했더니 변경사항이 반영되지 않았습니다.**
A. 서브모듈은 독립된 저장소입니다. 따라서 서브모듈 디렉터리 내에서 수정한 코드는 **해당 서브모듈 저장소에 별도로 `git push`를 수행해야 합니다**. 상위 저장소는 단지 서브모듈이 "어느 커밋을 가리키는지"에 대한 정보만을 관리합니다.

**Q3. 패키지 매니저(npm, Maven, pip 등)를 사용하는 것과 서브모듈을 사용하는 것 중 어떤 것이 좋나요?**
A. 일반적으로 **패키지 매니저 사용을 우선적으로 권장합니다**. 패키지 매니저는 버전 관리, 의존성 해결, 바이너리 배포에 최적화되어 있습니다. 서브모듈은 **소스 코드 수준의 경계를 유지해야 하거나, 빌드되지 않는 리소스(설정 파일, 에셋)를 공유해야 하거나, 권한을 엄격하게 분리해야 하는 경우**에 고려하는 것이 좋습니다.

---

## 결론

Git 서브모듈은 프로젝트 간 코드를 공유하면서도 **버전 고정, 경계 분리, 권한 독립**을 유지해야 하는 복잡한 시나리오에서 강력한 도구입니다. 그러나 그만큼 **명시적인 관리와 이해가 필요**합니다.

성공적인 도입을 위해서는 팀 내에 **명확한 온보딩 문서, 표준화된 CI/CD 설정, 서브모듈 포인터 업데이트 절차**를 마련하는 것이 중요합니다. 이는 "서브모듈이 너무 어렵다"는 인식을 줄이고, 실수 가능성을 낮춥니다.

궁극적으로 프로젝트의 요구사항을 신중히 평가하여, 서브모듈이 정말 최선의 선택인지, 아니면 **Subtree**, **모노레포**, **패키지 레지스트리**와 같은 대안이 더 적합한지 판단하는 것이 장기적인 유지 보수성과 개발자 경험에 도움이 될 것입니다.
