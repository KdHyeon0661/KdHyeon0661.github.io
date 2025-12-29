---
layout: post
title: Git - Fork vs Clone
date: 2025-01-19 20:20:23 +0900
category: Git
---
# GitHub의 Fork란 무엇인가?

## 핵심 개념 정리

- **Fork**: 다른 사람이나 조직의 GitHub 저장소를 **나의 GitHub 계정 아래로 독립적으로 복제**하는 기능입니다. 이를 통해 원본 저장소에 직접적인 권한이 없더라도 **내가 push할 수 있는 저장소**를 확보할 수 있습니다. 주로 **오픈소스 프로젝트에 기여**하기 위해 사용됩니다.
- **Clone**: 원격 저장소(원본이든, 내 포크든)를 **나의 로컬 컴퓨터 디렉터리**로 복제하는 Git 명령어입니다. 실제 개발 작업을 시작하려면 항상 `git clone`이 필요합니다.
- 일반적인 오픈소스 기여 흐름은 다음과 같습니다: **Fork → (내 Fork를) Clone → 브랜치 생성 및 작업 → Origin(내 Fork)으로 push → 원본 저장소(Upstream)로 PR 생성**.
- 협업에서는 보통 **origin을 내 Fork**, **upstream을 원본 저장소**로 설정하여 두 개의 원격을 관리합니다. 원본의 변경 사항을 주기적으로 내 저장소와 로컬 환경에 동기화하는 루틴을 갖추는 것이 중요합니다.

---

## Fork와 Clone 비교

| 항목 | Fork | Clone |
|---|---|---|
| 개념 | GitHub 상의 **다른 사람 저장소를 내 계정으로 복제** | 원격 저장소(원본/포크)를 **내 로컬 디렉터리로 복제** |
| 수행 위치 | GitHub 웹사이트 또는 GitHub CLI | 로컬 터미널(Git 명령어) |
| 생성 결과물 | **내 소유의 새로운 원격 저장소(내 계정 내)** | **로컬 작업 디렉터리** |
| 권한 | 내 저장소이므로 자유롭게 **push 가능** | 복제 대상 원격 저장소의 권한에 따름(읽기 전용일 수 있음) |
| 주요 목적 | 오픈소스 기여, 원본에 PR 보내기, 내 계정에서 독립적으로 코드 운영 | 소스 코드를 내려받아 로컬에서 작업 시작 |
| 원본 저장소로 PR 보내기 | **가능**(일반적인 사용 목적) | clone만으로는 불충분하며, 보통 Fork를 기반으로 진행 |
| 동기화 관리 | **upstream** 원격을 추가하여 주기적으로 동기화 필요 | 보통 `origin` 하나만 존재하며, 필요 시 다중 원격 설정 가능 |

---

## Fork의 정의와 목적

Fork는 **GitHub 플랫폼 차원**에서 이루어지는 "계정 간 원격 저장소 복제" 작업입니다. 그 결과로 나의 GitHub 계정에 원본 저장소와 동일한 내용을 가진, 그러나 **완전히 내 소유인 독립 저장소**가 생성됩니다. 원본 저장소와의 연결 정보(예: 네트워크 그래프, Compare 기능)는 유지되며, 이를 통해 기여 내역을 추적할 수 있습니다.

**핵심 효과**: 원본 저장소에 대한 쓰기 권한이 없더라도, **내 Fork에 자유롭게 코드를 push하고, 원본 저장소로 Pull Request를 보낼 수 있는 통로**를 만들어 줍니다.

**주요 사용 사례**
- 오픈소스 프로젝트에 기여하고자 할 때(권한이 없는 경우)
- 외부 저장소를 내 계정의 정책이나 CI/CD 환경에서 실험하고 싶을 때
- 조직 내에서 권한 분리와 코드 검토 프로세스를 강화하기 위해(조직 정책에 따라 포크 사용 여부가 결정됨)

---

## Clone의 재정의

`git clone`은 **어떤 원격 저장소(원본이든 내 포크든)** 의 전체 내용과 이력을 **로컬 머신에 복제**하는 Git의 기본 명령입니다.
- clone을 실행하면 기본적으로 **origin** 이라는 이름의 원격 저장소 정보가 등록됩니다.
- **내 포크를 clone**한 경우, `origin`은 **내 포크 저장소**를 가리키게 되어, 나는 이 `origin`에 자유롭게 push할 수 있습니다.
- 반면, **원본 저장소를 직접 clone**하면, 해당 저장소에 대한 쓰기 권한이 없는 한 push는 불가능합니다. 이러한 경우 Fork를 통해 push 권한을 확보한 후 PR을 생성하는 것이 표준적인 방법입니다.

---

## 오픈소스 기여 기본 흐름: Fork, Clone, 작업, PR

### 1. GitHub 웹에서 Fork 생성
원본 저장소 페이지에서 **Fork** 버튼을 클릭한 후, 대상 계정을 선택하여 포크를 생성합니다.

### 2. 내 포크를 로컬로 Clone
```bash
git clone git@github.com:myusername/repo-name.git
cd repo-name
```

### 3. 원본 저장소를 Upstream으로 등록
```bash
git remote add upstream https://github.com/original-owner/repo-name.git
git remote -v
# 출력 결과 예시:
# origin   git@github.com:myusername/repo-name.git (fetch/push)
# upstream https://github.com/original-owner/repo-name.git (fetch)
```

### 4. 작업 브랜치 생성 및 개발
```bash
git checkout -b feature/add-cool-thing
# 코드 수정 작업 진행

git add .
git commit -m "feat: add cool thing"
git push -u origin feature/add-cool-thing
```

### 5. GitHub에서 Pull Request 생성
내 포크의 `feature/add-cool-thing` 브랜치와 원본 저장소의 `main` (또는 다른 대상) 브랜치를 비교(Compare)하여 Pull Request를 생성합니다.

---

## Fork를 최신 상태로 유지하기: Upstream 동기화 전략

원본 저장소는 다른 기여자들에 의해 계속 발전합니다. 내 포크와 로컬 작업을 최신 상태로 유지하려면 정기적인 동기화가 필수적입니다.

### 안전하고 일반적인 패턴: Fetch + Rebase
```bash
# 원본 저장소(upstream)의 최신 변경사항 가져오기
git fetch upstream

# 로컬의 main 브랜치로 이동
git checkout main

# upstream/main의 변경사항을 기반으로 로컬 main 브랜치 재배치 (선형 이력 유지)
git rebase upstream/main

# 변경된 이력을 내 포크(origin)의 main 브랜치에도 반영
git push -f origin main
```
- `-f`(force push)는 리베이스로 인해 커밋 이력이 변경되었을 때 필요할 수 있습니다. 팀 내에서 force push 허용 여부를 확인하는 것이 좋습니다.

### Merge 기반 동기화 (커밋 이력 보존 선호 시)
```bash
git fetch upstream
git checkout main
git merge upstream/main
git push origin main
```

### 작업 중인 특정 브랜치를 최신 main 위로 재배치
```bash
git checkout feature/add-cool-thing
git fetch upstream
git rebase upstream/main
# 충돌 발생 시 해결 진행
git push -f origin feature/add-cool-thing
```

---

## Fork vs Template Repository vs Mirror (개념 구분)

- **Fork**: 원본과의 **연결성**을 유지합니다. 네트워크 그래프와 PR을 통한 상호 작용이 가능하며, 오픈소스 기여의 표준 경로입니다.
- **Template Repository**: 프로젝트의 구조와 보일러플레이트 코드만을 복제하여 **완전히 새로운 저장소를 생성**합니다. 원본과의 연결은 보존되지 않으며, 별개의 프로젝트를 시작할 때 사용합니다.
- **Mirror**: `git clone --mirror` 명령으로 모든 브랜치, 태그, 참조까지 포함한 완벽한 복제본을 만듭니다. 주로 저장소 마이그레이션이나 백업 목적으로 사용됩니다.

---

## 권한, 보안 및 조직 정책 관련 주의사항

1.  **포크 허용 정책**: 조직이나 특정 저장소 설정에서 보안이나 지적 재산권 보호를 이유로 포크 기능을 비활성화할 수 있습니다.
2.  **프라이빗 저장소 포크**: 조직 정책에 따라 프라이빗 저장소의 포크를 동일 조직 내로만 제한하거나, 전면 금지할 수 있습니다.
3.  **GitHub Actions 보안**: 포크에서 생성된 PR에 대한 워크플로우 실행은 기본적으로 **저장소 시크릿에 접근할 수 없도록 제한**됩니다. `pull_request_target` 이벤트를 사용할 때는 특히 주의해야 하며, 신뢰할 수 없는 코드(외부 기여자의 코드)에서 시크릿을 사용하거나 배포 작업을 수행해서는 안 됩니다.
4.  **브랜치 보호 규칙**: 원본 저장소는 병합 전 필수 리뷰나 상태 확인을 요구하는 보호 규칙을 설정하는 경우가 많습니다. 포크에서 보낸 PR도 이러한 규칙을 모두 통과해야 병합될 수 있습니다.
5.  **라이선스 준수**: 포크한 저장소는 원본의 라이선스 조건을 따라야 합니다. 코드를 수정하거나 배포할 때는 관련 라이선스 조항을 반드시 확인해야 합니다.

---

## Fork 기반 협업을 위한 모범 사례

- **브랜치 네이밍**: `feat/…`, `fix/…`, `docs/…`와 같은 일관된 컨벤션을 사용합니다.
- **커밋 메시지**: Conventional Commits 규약을 따르는 것을 권장합니다.
- **PR 템플릿 활용**: 변경 사항 요약, 테스트 방법, 관련 이슈 등을 구조화하여 명확한 PR 작성을 돕습니다.

예시 `.github/pull_request_template.md`:
```markdown
## 변경 사항 요약
-

## 변경 유형
- [ ] 신규 기능
- [ ] 버그 수정
- [ ] 문서 수정
- [ ] 리팩토링

## 테스트 방법
-

## 관련 이슈
Closes #<이슈 번호>
```

---

## 다중 원격 저장소 관리

```bash
# 현재 등록된 원격 저장소 목록 확인
git remote -v

# 추가 원격 저장소 등록 (예: 개인 실험용 저장소)
git remote add personal git@github.com:myusername/experiment.git

# 특정 브랜치를 다른 원격 저장소로 푸시
git push personal my-prototype-branch:my-prototype-branch
```

---

## 포크 기반 작업에서 자주 마주치는 Git 작업

### 리베이스 중 충돌 해결
```bash
git rebase upstream/main
# <<<<<<<, >>>>>>> 충돌 마커가 표시된 파일들을 수동으로 수정합니다.

git add <수정된_파일>
git rebase --continue
```

### 커밋 이력 정리
```bash
git rebase -i HEAD~5
# 대화형 리베이스 모드에서 pick, squash, reword 등을 활용하여 커밋을 정리합니다.
```

### 특정 커밋만 적용하기 (체리픽)
```bash
git checkout feature/x
git cherry-pick <upstream이나-다른-브랜치의-커밋-SHA>
```

---

## GitHub CLI(`gh`)를 활용한 효율적인 작업

### 포크 생성 및 클론 (한 번에)
```bash
# 원본 저장소를 포크하고, 포크된 저장소를 바로 클론합니다.
gh repo fork original-owner/repo --clone
cd repo
gh repo set-default origin
```

### 브랜치 작업 및 PR 생성
```bash
git checkout -b feat/new-ui
# 작업 후 커밋

git add .
git commit -m "feat(ui): redesign header"
git push -u origin feat/new-ui

# PR 자동 생성 (제목과 내용을 커밋 메시지에서 가져옴)
gh pr create --fill --base main --head myusername:feat/new-ui
gh pr view --web # 브라우저에서 PR 확인
```

---

## 포크와 CI/CD: 보안 패턴

포크에서 온 PR에 대한 CI 실행은 보안상 제한됩니다. 일반적인 패턴은 다음과 같습니다.
- `pull_request` 이벤트: 포크의 코드 변경에 대해 **읽기 권한 수준**으로 워크플로우를 실행하며, 저장소 시크릿에는 접근할 수 없습니다.
- 배포나 민감한 작업은 **원본 저장소의 특정 브랜치(예: main)에 푸시되었을 때만** 실행되도록 조건을 걸어야 합니다.

예시 `.github/workflows/ci.yml`:
```yaml
name: CI
on:
  pull_request:
  push:
    branches: [ main ]

jobs:
  test:
    runs-on: ubuntu-latest
    permissions:
      contents: read
    steps:
      - uses: actions/checkout@v4
      - run: npm ci
      - run: npm test

  deploy:
    if: github.event_name == 'push' && startsWith(github.ref, 'refs/heads/main')
    runs-on: ubuntu-latest
    permissions:
      contents: write
      id-token: write
    steps:
      - uses: actions/checkout@v4
      - run: ./scripts/deploy.sh
```
- 위 예시에서 `deploy` 잡은 오직 원본 저장소의 `main` 브랜치에 푸시될 때만 실행되므로, 포크 PR에 의한 배포를 방지합니다.

---

## 실전 시나리오별 해결 스크립트

### 시나리오 1: "포크는 했는데, 원본의 최신 변경사항으로 내 포크를 업데이트하고 싶다"
```bash
# upstream 원격이 설정되어 있는지 확인
git remote -v
# 설정되지 않았다면 추가
git remote add upstream https://github.com/original-owner/repo.git

# 동기화 수행
git fetch upstream
git checkout main
git rebase upstream/main
git push -f origin main
```

### 시나리오 2: "작업 중인 브랜치를 최신 upstream main 브랜치 기반으로 재작업하고 싶다"
```bash
git checkout feature/my-work
git fetch upstream
git rebase upstream/main
# 충돌 해결 후
git push -f origin feature/my-work
```

### 시나리오 3: "원본 저장소를 직접 clone했는데 push 권한이 없다"
```bash
# 1. GitHub 웹사이트에서 원본 저장소를 내 계정으로 Fork합니다.
# 2. 로컬 저장소의 원격 URL을 내 포크의 URL로 변경합니다.
git remote set-url origin git@github.com:myusername/repo.git
# 또는, 내 포크를 새로 clone한 후 기존 로컬 변경사항을 옮기는 방법도 있습니다.
```

### 시나리오 4: "내 포크에서 원본 저장소로 PR을 보내는 방법은?"
1.  내 포크의 작업 브랜치를 origin(내 포크)에 푸시합니다.
    ```bash
    git push -u origin feature/x
    ```
2.  GitHub 웹사이트에서 내 포크 저장소로 이동합니다.
3.  방금 푸시한 `feature/x` 브랜치와 원본 저장소의 `main` 브랜치를 비교(Compare)하여 Pull Request를 생성합니다.

---

## 유지 관리자(원본 저장소 소유자)를 위한 팁

- **기여 가이드 제공**: `CONTRIBUTING.md` 파일에 포크 방법, 업스트림 동기화 절차, 테스트 실행 방법 등을 명확히 기재합니다.
- **PR 안전 장치 마련**: 브랜치 보호 규칙(필수 리뷰, 상태 확인), 자동 라벨 부여, 코드 오너(`CODEOWNERS` 파일) 지정 등을 활용합니다.
- **CI 보안 강화**: 포크 PR에는 시크릿 노출을 차단하고, `pull_request_target` 사용 시에는 권한 상승 공격에 각별히 주의합니다.
- **신규 기여자 배려**: 이슈/PR 템플릿과 명확한 라벨 체계를 도입하여 프로젝트 기여 진입 장벽을 낮춥니다.

---

## 전체 과정 예시: 첫 포크부터 PR 생성까지

```bash
# GitHub 웹에서 포크를 생성한 후, 터미널에서 진행

# 1. 내 포크를 클론
git clone git@github.com:myusername/awesome-lib.git
cd awesome-lib

# 2. 원본 저장소를 upstream으로 추가
git remote add upstream https://github.com/original-owner/awesome-lib.git
git fetch upstream

# 3. 로컬 main 브랜치를 upstream/main과 동기화
git checkout main
git rebase upstream/main
git push -f origin main

# 4. 새로운 작업 브랜치 생성
git checkout -b fix/typo-in-docs

# 5. 코드 수정 및 커밋
sed -i 's/Awseome/Awesome/g' docs/intro.md
git add docs/intro.md
git commit -m "docs: fix typo in intro"

# 6. 내 포크로 푸시
git push -u origin fix/typo-in-docs

# 7. PR 생성 (GitHub 웹사이트 또는 gh CLI 사용)
# gh pr create --fill --base main --head myusername:fix/typo-in-docs
```

---

## Fork와 Template Repository 선택 기준

- **목적이 기여(Contribution)라면** → **Fork**를 선택하세요. 원본과의 연결을 유지하며 PR을 보낼 수 있습니다.
- **목적이 새 프로젝트 시작(보일러플레이트 복제)이라면** → **Template Repository**를 활용하세요. 원본과의 연결 없이 독립적인 프로젝트를 시작합니다.
- **목적이 저장소의 완전한 복제 및 이전이라면** → **Mirror** 기능을 고려하세요.

---

## 자주 묻는 질문 (FAQ)

**Q1. 포크를 삭제해도 원본 저장소에 영향을 미치나요?**
A. 아닙니다. 포크는 내 계정의 독립된 저장소입니다. 포크를 삭제해도 원본 저장소에는 어떠한 영향도 없습니다.

**Q2. 포크한 저장소에서 생성한 이슈나 디스커션은 원본과 공유되나요?**
A. 일반적으로 공유되지 않습니다. 이슈와 디스커션은 각 저장소별로 독립적입니다. 다만, PR은 원본 저장소에 생성되므로 그곳의 논의 흐름에 포함됩니다.

**Q3. 포크한 저장소를 프라이빗으로 전환할 수 있나요?**
A. 가능합니다. 하지만 원본 저장소의 라이선스(예: 오픈소스 라이선스)와 조직의 정책을 반드시 확인해야 합니다. 일부 라이선스는 프라이빗 사용을 제한할 수 있습니다.

**Q4. 포크에서 실행되는 GitHub Actions가 실패하거나 시크릿에 접근하지 못합니다.**
A. 이는 의도된 보안 동작입니다. 포크에서 생성된 PR에 대해서는 저장소 시크릿이 공유되지 않습니다. 배포 등 시크릿을 필요로 하는 작업은 원본 저장소의 브랜치 푸시 시에만 실행되도록 워크플로우를 설계해야 합니다.

---

## 결론

GitHub의 Fork는 오픈소스 생태계의 협업 모델을 가능하게 하는 핵심 기능입니다. 원본 저장소에 대한 직접적인 쓰기 권한 없이도, 포크를 통해 개인적인 실험 공간을 만들고, 변경사항을 안전하게 원본에 제안할 수 있습니다. 효과적인 기여를 위해서는 `origin`(내 포크)과 `upstream`(원본)을 명확히 구분하고, 주기적인 동기화 작업을 통해 내 작업 환경을 최신 상태로 유지하는 습관이 중요합니다. 또한, 포크 PR에 대한 CI/CD 보안 모델을 이해하고, 브랜치 네이밍, 커밋 메시지, PR 템플릿과 같은 협업 규칙을 준수함으로써 원활한 코드 리뷰와 병합 프로세스를 도울 수 있습니다.

---

## 주요 참고 명령어 요약

```bash
# 포크한 저장소를 로컬에 클론
git clone git@github.com:myusername/forked-repo.git
cd forked-repo

# 원본 저장소를 upstream 원격으로 추가
git remote add upstream https://github.com/original-owner/repo.git
git remote -v # 확인

# 원본의 최신 변경사항으로 로컬 main 브랜치 업데이트 (리베이스 방식)
git fetch upstream
git checkout main
git rebase upstream/main
git push -f origin main

# 새로운 기능 브랜치 생성 및 작업
git checkout -b feature/x
git add .
git commit -m "feat: x"
git push -u origin feature/x

# GitHub CLI를 사용한 PR 생성
gh pr create --fill --base main --head myusername:feature/x
```
