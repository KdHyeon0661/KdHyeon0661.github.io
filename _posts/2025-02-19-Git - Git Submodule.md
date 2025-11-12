---
layout: post
title: Git - Git Submodule
date: 2025-02-19 19:20:23 +0900
category: Git
---
# Git Submodule

## 0. 서브모듈 한 줄 요약

- **서브모듈(Submodule)** = 상위 리포 안에서 **별도 Git 리포를 디렉터리로 연결**하는 링크.  
- 상위 리포는 **서브모듈의 “커밋 해시(정확한 한 점)”만 기록**한다.  
- 장점: **정밀 재현성(핀고정)**, 코드 재사용, 조직 간 경계 유지.  
- 단점: 초기화/업데이트/권한/CI 관리가 **명시적이고 수동적**이다.

---

## 1. 언제 서브모듈을 선택할까?

- 여러 프로젝트가 공유하는 **내부 공용 라이브러리**를 모듈화하여 **독립적 릴리스/권한**을 유지하고 싶은 경우
- **외부 오픈소스**를 특정 커밋에 고정하여 재현성을 극대화하고 싶은 경우
- **변경 주기/권한/배포**를 상위와 분리하고 싶은 경우 (예: 보안/라이선스 경계)

> 반대로 상위 리포에 **코드를 “실제 복사”**해 운영하고 싶거나, **동기화를 상위에서 자동화**하고 싶다면 **Subtree**가 더 단순한 선택일 수 있다(아래 §13 비교 참조).

---

## 2. 기본 사용 — 추가→커밋→클론/초기화→업데이트

### 2.1 서브모듈 추가

```bash
git submodule add https://github.com/my-org/common-ui.git libs/common-ui
```

- 상위 리포에 `.gitmodules`가 생성되고, `libs/common-ui` 폴더가 **별도 Git 리포**로 초기화된다.
- 상위 리포는 `libs/common-ui`가 가리키는 **정확한 커밋 해시**를 스냅샷으로 기록한다.

### 2.2 커밋/푸시

```bash
git add .gitmodules libs/common-ui
git commit -m "Add common-ui as submodule"
git push origin main
```

`.gitmodules` 예시:

```ini
[submodule "libs/common-ui"]
  path = libs/common-ui
  url = https://github.com/my-org/common-ui.git
```

### 2.3 클론과 초기화

서브모듈 포함 리포를 클론할 때 두 가지 접근:

```bash
# 방법 A: 한 번에 전부
git clone --recurse-submodules https://github.com/my-org/my-repo.git

# 방법 B: 나중에 초기화
git clone https://github.com/my-org/my-repo.git
cd my-repo
git submodule update --init --recursive
```

> `--recursive`는 **서브모듈 내부의 서브모듈(중첩 서브모듈)**까지 초기화한다.

### 2.4 서브모듈 상태/업데이트

```bash
git submodule status                   # 현재 pin 된 커밋 확인
git submodule update --remote          # .gitmodules의 branch 설정이 있을 때 최신 추적
git submodule update --init --recursive
```

> 기본적으로 서브모듈은 **브랜치가 아닌 커밋**을 가리킨다(핀고정). “최신을 당겨오고 싶다”면 §5의 **브랜치 추적** 또는 §4의 **수동 업그레이드 절차**를 사용한다.

---

## 3. 내부 구조와 사고방식

- 상위 리포는 서브모듈 디렉터리를 **Git 트리에서 “링크(특별한 엔트리)”**로 보유하고, 해당 링크가 **특정 커밋 해시**를 참조한다.
- 상위 리포의 커밋에는 “서브모듈 디렉터리 → 커밋 해시” 매핑이 포함된다.
- 따라서 상위 리포가 특정 커밋을 체크아웃하면, 올바른 서브모듈 커밋을 받기 위해 **명시적인 `submodule update`**가 필요하다.

간단한 비용 모델(의사 수식):

$$
T_{\text{build}} \approx T_{\text{clone\_root}}
+\sum_{i=1}^{N} \Big(T_{\text{clone\_submodule}_i} + T_{\text{checkout}_i}\Big),
$$

여기서 \(N\)은 서브모듈 수. **서브모듈이 많을수록** CI 시간이 증가하므로 **캐싱/부분 체크아웃/병렬화** 고려가 중요하다.

---

## 4. 서브모듈 “버전 업(핀 이동)” 표준 절차

상위 리포에서 서브모듈을 최신으로 끌어올리고 싶을 때:

```bash
# 1. 서브모듈 디렉터리로 진입
cd libs/common-ui

# 2. 원하는 기준으로 업데이트 (브랜치 최신, 특정 태그 등)
git fetch origin
git checkout main
git pull origin main

# 3. 상위로 돌아와 "핀 이동" 커밋
cd ../..
git add libs/common-ui       # 서브모듈 디렉터리의 pointer 변경을 stage
git commit -m "Bump common-ui to <new-commit>"
git push
```

> **핵심**: 서브모듈에도 **별도의 `git push`**가 필요하다(서브모듈 자체 변경이 있다면). 상위 리포는 오직 “어느 커밋을 가리키는지”만 커밋한다.

---

## 5. `.gitmodules` 고급 설정 — 브랜치 추적/업데이트 정책/깊은 클론

```ini
[submodule "libs/common-ui"]
  path = libs/common-ui
  url = https://github.com/my-org/common-ui.git
  branch = main         # 추적할 브랜치(선택) — update --remote 시 기준
  update = rebase       # 또는 merge/checkout — submodule update 정책(선택)
  shallow = true        # 얕은 클론(선택, Git 2.9+)
```

### 5.1 `branch = <name>`
- `git submodule update --remote` 시 해당 브랜치의 최신 커밋으로 **핀 이동**을 자동화한다.
- **주의**: 상위 리포에 “자동으로 최신”이 반영되는 게 아니라, **update 후 상위에서 다시 커밋**해야 고정점이 바뀐다.

```bash
git submodule update --remote --recursive
git add libs/common-ui
git commit -m "Track common-ui to latest main"
```

### 5.2 `update = rebase|merge|checkout`
- `git submodule update` 수행 방식(로컬 변경이 있을 때 정책).
- 기본은 `checkout`(깨끗한 워크트리를 가정). 로컬 커밋 유지가 필요하면 `rebase`나 `merge` 고려.

### 5.3 `shallow = true`
- 서브모듈 클론을 얕게 수행하여 CI 시간/네트워크를 절약.  
- 과거 이력 접근이 필요하면 해제.

---

## 6. 팀 협업 관점의 워크플로

### 6.1 PR 흐름(상위에만 변경)
1. 기여자는 상위 리포에서 서브모듈 **포인터만** 변경해 PR 생성.
2. 리뷰어는 `git submodule status` 또는 UI에서 **어느 커밋으로 바뀌는지** 확인.
3. CI는 `submodules: true`로 체크아웃 후 빌드/테스트.
4. 머지되면 상위 리포의 핀이 갱신.

### 6.2 PR 흐름(하위와 상위 모두 변경)
1. 서브모듈 리포에 PR을 먼저 만들어 머지/태그.
2. 상위 리포에서 해당 커밋/태그로 포인터 이동 PR.
3. 두 PR 간 의존성이 있다면 **체인드 PR** 전략 또는 **멀티-리포 동시 릴리스** 관리.

---

## 7. CI/CD — GitHub Actions 예제

### 7.1 기본(재귀 체크아웃)

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
          submodules: recursive     # 서브모듈까지 체크아웃
          fetch-depth: 0            # 필요 시 전체 이력

      - name: Verify submodule pointers
        run: git submodule status

      - name: Build & test
        run: |
          npm ci
          npm run build
          npm test -- --ci
```

### 7.2 부분 체크아웃(성능/권한 제약 시)

```yaml
- uses: actions/checkout@v4
  with:
    submodules: false

- name: Init only needed submodules
  run: |
    git submodule update --init libs/common-ui
    # 필요 시 얕은 클론/특정 브랜치만
    git -C libs/common-ui fetch --depth=1 origin main
    git -C libs/common-ui checkout FETCH_HEAD
```

### 7.3 캐시 전략
- 서브모듈은 리포가 분리되어 있어 **의존 캐시**를 각자 유지해야 한다.
- Node/Gradle/Maven 등 **빌드 캐시**와 **아티팩트**를 조합해 CI 시간을 줄인다.

---

## 8. 권한/토큰/프라이빗 서브모듈

- **프라이빗 서브모듈**은 체크아웃 시 **개별 인증**이 필요하다.
- GitHub Actions의 경우 **동일 Org 프라이빗 서브모듈**은 `actions/checkout@v4`가 **GITHUB_TOKEN**으로 처리 가능(리포 권한 설정 필요).
- 서로 다른 Org/호스트라면 **Deploy Key/Personal Access Token/SSH** 구성 필요:
  ```yaml
  - uses: actions/checkout@v4
    with:
      submodules: recursive
      ssh-key: ${{ secrets.DEPLOY_KEY }}
  ```

---

## 9. 서브모듈 삭제/경로 변경/URL 변경

### 9.1 삭제(정식 절차)

```bash
# 1. .gitmodules에서 항목 제거
# 2. .git/config에서 submodule.<path> 섹션 제거 (자동 동기화 가능)
git submodule deinit -f libs/common-ui
git rm -f libs/common-ui
rm -rf .git/modules/libs/common-ui
git commit -m "Remove submodule common-ui"
```

> 수동으로 `.gitmodules`, `.git/config`, `.git/modules/<path>`를 정리하지 않으면 **유령 레코드**가 남아 혼란을 준다.

### 9.2 경로 변경

```bash
git mv libs/common-ui libs/ui
git config -f .gitmodules submodule.libs/common-ui.path libs/ui
git submodule sync           # .git/config 반영
git add .gitmodules
git commit -m "Move submodule to libs/ui"
```

### 9.3 URL 변경

```bash
git config -f .gitmodules submodule.libs/ui.url git@github.com:my-org/ui.git
git submodule sync
git submodule update --init --recursive
git add .gitmodules
git commit -m "Point submodule to new remote"
```

---

## 10. 자주 쓰는 명령어 모음(치트시트)

```bash
# 상태/초기화/업데이트
git submodule status
git submodule init
git submodule update
git submodule update --init --recursive

# 업스트림 추적(브랜치 기반)
git submodule update --remote
git config -f .gitmodules submodule.libs/common-ui.branch main
git add .gitmodules && git commit -m "Track branch for common-ui"

# 모두 순회
git submodule foreach 'git status'
git submodule foreach --recursive 'git pull --rebase --autostash'

# 삭제
git submodule deinit -f libs/common-ui
git rm -f libs/common-ui
rm -rf .git/modules/libs/common-ui

# 동기화(경로/URL 변경 시)
git submodule sync --recursive
```

---

## 11. 중첩 서브모듈(Nested Submodules)과 모노레포

- **중첩 구조**: A(상위) → B(서브모듈) → C(서브모듈).  
  `--recursive` 옵션으로 한 번에 초기화/업데이트 가능하나, 권한/토큰/네트워크 비용이 누적된다.
- **모노레포**에서 서브모듈을 사용할 경우, “사실상 외부 리포”인 서브모듈이 **모노레포의 공용 규약/CI 파이프라인**과 엇갈릴 수 있음.  
  - 가능하면 **Subtree**나 **패키지 매니저/사내 레지스트리**(npm, Maven, PyPI, NuGet 등)로 대체를 검토.

---

## 12. 서브모듈 트러블슈팅

### 12.1 “서브모듈 디렉토리가 비어 보임”
- 초기화/업데이트 안 한 상태.  
  → `git submodule update --init --recursive`

### 12.2 “권한 오류(프라이빗 리포 접근 불가)”
- CI/로컬의 인증 미구성.  
  → HTTPS(토큰) 또는 SSH 키 설정, `actions/checkout@v4`의 `ssh-key`/`token` 사용.

### 12.3 “로컬 변경 때문에 업데이트 실패”
- 서브모듈 내부에서 수정이 남아있어 `checkout` 불가.  
  → `git -C libs/common-ui stash` 후 `git submodule update`  
  또는 `.gitmodules`의 `update=rebase`로 정책 조정.

### 12.4 “상위에서 submodule 포인터만 바뀌었는데 빌드가 예전 코드로”
- CI가 `submodules: true` 없이 체크아웃.  
  → Actions에서 `submodules: recursive` 사용, 또는 수동 `git submodule update --init`.

### 12.5 “경로/URL을 바꿨는데 sync가 안 됨”
- `.gitmodules`만 고치고 `sync`를 누락.  
  → `git submodule sync --recursive`

### 12.6 “의존 캐시가 계속 깨짐”
- 서브모듈 해시가 자주 바뀌면 캐시 적중률이 떨어짐.  
  → 서브모듈 릴리스 주기 재설계, 빌드 아티팩트 캐시로 보완.

---

## 13. 서브모듈 vs 서브트리(Subtree) — 실전 비교

| 항목 | Submodule | Subtree |
|---|---|---|
| 저장 방식 | **링크(커밋 핀)**만 상위가 기록 | **코드를 상위에 통합**(실제 파일 복사/병합) |
| 외부성/권한 | 리포/권한 완전 분리 | 상위 리포 규약에 흡수 |
| 업데이트 | 명시적 `submodule update` 필요 | 상위에서 subtree pull/push |
| 이력 재현 | 강력(커밋 핀) | 상위에 섞여 설명 필요 |
| 학습/운영 난도 | 비교적 높음 | 낮음(툴 내장, 추가 개념 적음) |
| CI/네트워크 | 서브모듈 개수만큼 비용 | 상위 리포 크기 증가 가능성 |
| 추천 상황 | 경계·권한·릴리스 분리를 유지해야 할 때 | 단일 리포로 단순 운영하고 싶을 때 |

**변환 팁**: 서브모듈 → 서브트리로 변환 시, `git subtree add --prefix=libs/common-ui <url> <ref>`로 가져온 뒤 서브모듈 제거 절차(§9.1)를 수행.

---

## 14. 시나리오 예제

### 14.1 공통 UI 라이브러리를 여러 서비스가 공용 사용

구성:
```
app-frontend/          # 상위 리포
└── libs/common-ui     # submodule → my-org/common-ui
```

도입:
```bash
cd app-frontend
git submodule add https://github.com/my-org/common-ui.git libs/common-ui
git add .gitmodules libs/common-ui
git commit -m "Add common-ui as submodule"
```

일상 업데이트:
```bash
# common-ui 최신 main으로
git -C libs/common-ui fetch origin
git -C libs/common-ui checkout main
git -C libs/common-ui pull origin main

# 상위에서 핀 이동 커밋
git add libs/common-ui
git commit -m "Bump common-ui to latest main"
git push
```

CI(요지):
```yaml
- uses: actions/checkout@v4
  with: { submodules: recursive }
- run: npm ci && npm run build && npm test
```

### 14.2 외부 OSS를 특정 태그에 고정

```bash
git submodule add https://github.com/some/oss.git vendor/oss
git -C vendor/oss fetch --tags
git -C vendor/oss checkout v2.3.1
git add vendor/oss
git commit -m "Pin oss to v2.3.1"
```

### 14.3 브랜치 추적 자동화(.gitmodules)

```ini
[submodule "libs/common-ui"]
  path = libs/common-ui
  url = https://github.com/my-org/common-ui.git
  branch = main
```

```bash
git submodule update --remote --recursive
git add libs/common-ui
git commit -m "Track common-ui: update to latest main"
```

---

## 15. 성능/안정성 최적화

- 서브모듈 수를 신중히 제한(네트워크/CI 비용 선형 증가).
- **얕은 클론(shallow)** + **부분 초기화**로 필요한 것만 받는다.
- **릴리스 태그 고정**으로 예측 가능한 재현성 확보.
- **권한/토큰 전략**을 문서화(온보딩 시 빠른 성공 경험 제공).

---

## 16. 전체 흐름 명령 요약(한 컷)

```bash
# 추가
git submodule add <url> path/to/sub
git add .gitmodules path/to/sub
git commit -m "Add submodule"

# 클론/초기화
git clone --recurse-submodules <root-url>
# 또는
git submodule update --init --recursive

# 최신 반영(브랜치 추적 시)
git submodule update --remote --recursive

# 포인터 업그레이드(수동)
git -C path/to/sub fetch origin
git -C path/to/sub checkout <branch-or-tag>
git -C path/to/sub pull --ff-only origin <branch>
git add path/to/sub
git commit -m "Bump submodule"
git push

# 삭제
git submodule deinit -f path/to/sub
git rm -f path/to/sub
rm -rf .git/modules/path/to/sub
git commit -m "Remove submodule"

# 동기화
git submodule sync --recursive
```

---

## 17. 자주 묻는 질문(FAQ)

**Q1. 서브모듈이 왜 “브랜치”가 아니라 “커밋”을 가리키나요?**  
A. 상위 리포의 재현성을 보장하려면 “시간에 따라 움직이지 않는 한 점”이 필요하기 때문이다. 브랜치를 추적하고 싶다면 `.gitmodules`의 `branch` + `update --remote`를 활용하되, **결국 상위에서 다시 커밋**해야 고정점이 바뀐다.

**Q2. 로컬에서 서브모듈을 수정했는데 상위 커밋만 푸시했더니 반영이 안 돼요.**  
A. 서브모듈 리포에 **별도 `git push`**가 필요하다. 상위는 “포인터만” 기록한다.

**Q3. 서브모듈 대신 패키지 매니저(예: npm, Maven)를 쓰면 안 되나요?**  
A. 가능하면 패키지 매니저를 권장한다. 서브모듈은 “소스 수준의 경계 유지/권한 분리/비빌드형 리소스”가 필요한 경우에 선택한다.

---

## 18. 결론

- 서브모듈은 **정밀한 고정/경계/권한 분리**에 매우 강력하지만, **명시적 관리 비용**이 따른다.
- 팀에서는 **온보딩 문서·CI 체크아웃 설정·포인터 업데이트 절차**를 표준화하여 “서브모듈이 어렵다”는 인식을 없애자.
- 프로젝트 성격에 따라 **Submodule vs Subtree vs 패키지 레지스트리**를 비교해 가장 단순한 도구를 택하는 것이 장기적으로 유리하다.

---

## 참고

- Git 공식 문서 — Submodules: https://git-scm.com/book/en/v2/Git-Tools-Submodules  
- Atlassian 튜토리얼 — Submodule: https://www.atlassian.com/git/tutorials/git-submodule  
- actions/checkout (submodules 옵션): https://github.com/actions/checkout