---
layout: post
title: Git - Fork vs Clone
date: 2025-01-19 20:20:23 +0900
category: Git
---
# GitHub의 Fork란 무엇인가?

## 0. 핵심 요약(한눈에 정리)

- **Fork**: 다른 사람/조직의 GitHub 저장소를 **내 GitHub 계정 아래 독립 복제**로 만들어 **내가 push 가능한** 저장소를 얻는 행위. 보통 **오픈소스 기여**를 위해 사용.
- **Clone**: 원격 저장소(원본이든, 내 포크든)를 **내 로컬 디렉터리**로 복제하는 Git 명령. **로컬 작업**을 시작하려면 언제나 `git clone`이 필요.
- 일반적인 오픈소스 기여 플로우: **Fork → (내 Fork를) Clone → 브랜치 생성/작업 → Origin(Fork)으로 push → 원본 저장소(Upstream)로 PR 생성**.
- 협업 실무에서는 **origin = 내 Fork**, **upstream = 원본** 으로 **원격 두 개**를 설정하고, 주기적으로 **upstream 변경 사항을 내 Fork/로컬에 반영**하는 루틴을 갖춘다.

---

## 1. Fork vs Clone 비교(기존 표 확장)

| 항목 | Fork | Clone |
|---|---|---|
| 개념 | GitHub 상의 **다른 사람 저장소를 내 계정으로 복제** | 원격 저장소(원본/포크)를 **내 로컬 디렉터리로 복제** |
| 수행 위치 | GitHub 웹(또는 GitHub CLI) | 로컬 터미널(Git) |
| 생성물 | **내 소유의 원격 저장소(내 계정)** | **로컬 작업 디렉터리** |
| 권한 | 내 저장소이므로 자유롭게 **push 가능** | 원격 권한에 따라 다름(읽기 전용일 수 있음) |
| 목적 | 오픈소스 기여, 원본에 PR, 내 계정에서 독립 운영 | 소스 내려받아 로컬 작업 시작 |
| PR(원본으로) | **가능**(일반적 목적) | clone만으로는 불충분(보통 Fork 기반이 필요) |
| 동기화 | **upstream** 원격을 추가, 주기적으로 동기화 필요 | 보통 `origin`만 존재; 필요 시 다중 원격 설정 가능 |

---

## 2. Fork란 무엇인가? (기존 정의 심화)

**정의**: Fork는 **GitHub 레벨**에서 수행하는 “원격 저장소의 계정 간 복제”입니다. 결과물은 **내 GitHub 계정에 속한 독립 저장소**이며, 원본과 **연결 정보(네트워크 탭 히스토리, Compare 기능 등)** 는 유지됩니다.
**효과**: 원본 저장소에 직접 권한이 없어도, **내 Fork에 자유롭게 push** → **원본에 PR**을 보낼 수 있게 됩니다.

**언제 쓰나**
- 오픈소스 프로젝트 기여(권한 없음)
- 외부 저장소를 **내 계정 정책/CI** 하에 실험하고 싶을 때
- 사내에서도 **권한 분리/검토 프로세스**가 필요한 경우(조직 정책에 따라 포크 허용/비허용 가능)

---

## 3. Clone이란 무엇인가? (기존 설명 보강)

`git clone`은 **어떤 원격 저장소(원본이든 포크든)** 를 **로컬에 복제**합니다.
- 보통 clone 후에는 **origin** 이라는 원격 이름으로 복제 대상이 등록됩니다.
- **내 포크를 clone**하면, `origin`은 **내 포크**를 가리키게 됩니다(이 경우 push 가능).
- 원본을 바로 clone하면, 권한이 없다면 **push 불가**입니다(읽기 전용). 이때는 Fork를 만들어 내 포크로 push한 뒤 PR을 생성합니다.

---

## 4. 기본 흐름: 오픈소스 기여(포크 → 클론 → 작업 → PR)

### 4.1 GitHub 웹에서 Fork
- 원본 저장소 페이지 → **Fork** 버튼 클릭 → 내 계정 선택 → 포크 생성

### 4.2 내 포크를 로컬로 clone
```bash
git clone git@github.com:myusername/repo-name.git
cd repo-name
```

### 4.3 원본 저장소를 upstream 으로 등록
```bash
git remote add upstream https://github.com/original-owner/repo-name.git
git remote -v
# origin   git@github.com:myusername/repo-name.git (fetch/push)
# upstream https://github.com/original-owner/repo-name.git (fetch)
```

### 4.4 작업 브랜치 생성 후 개발
```bash
git checkout -b feature/add-cool-thing
# 코드 수정
git add .
git commit -m "feat: add cool thing"
git push -u origin feature/add-cool-thing
```

### 4.5 GitHub에서 Pull Request 생성
- 내 포크의 `feature/add-cool-thing` → **Compare & pull request** → 원본 저장소의 대상 브랜치(보통 `main`)로 PR

---

## 5. Fork를 최신 상태로 유지(Upstream 동기화 전략)

원본 저장소는 활발히 변경됩니다. 내 포크/로컬을 최신으로 유지하려면 **정기 동기화**가 필요합니다.

### 5.1 안전하고 흔한 패턴: fetch + rebase
```bash
# 1. 원본 최신을 가져옴
git fetch upstream

# 2. 내 로컬 main을 체크아웃
git checkout main

# 3. upstream/main 위로 rebase (선형 이력 유지)
git rebase upstream/main

# 4. 내 포크(origin)에도 최신 main 반영
git push -f origin main
```
- `-f`(force)는 리베이스로 이력이 바뀌었을 때 필요할 수 있습니다. 팀에서 허용되는지 확인하십시오.

### 5.2 merge 기반(이력 보존 선호 시)
```bash
git fetch upstream
git checkout main
git merge upstream/main
git push origin main
```

### 5.3 특정 브랜치만 최신화(작업 브랜치 rebase)
```bash
git checkout feature/add-cool-thing
git fetch upstream
git rebase upstream/main
# 충돌 해결 후
git push -f origin feature/add-cool-thing
```

---

## 6. Fork vs Template Repository vs Mirror(혼동 주의)

- **Fork**: 원본과 **연결**(네트워크·PR) 유지. 오픈소스 기여 플로우 표준.
- **Template Repository**: 템플릿을 기반으로 **새 저장소 생성**. 원본과 연결 보존 X(별개 프로젝트 출발용).
- **Mirror**: `--mirror` 로 모든 참조(브랜치/태그/refs)까지 포함 복제. 주로 **레거시→GitHub 이전** 등에 사용.

---

## 7. 권한·보안·조직 정책(실무 주의점)

1) **포크 허용/금지**: 조직/저장소 설정에 포크 비활성화가 있을 수 있음(특허/비공개 코드 유출 방지).
2) **프라이빗 저장소 포크**: 조직 정책에 따라 제한될 수 있음(동일 조직 내만 허용 등).
3) **Actions 보안**: 포크에서 보낸 PR은 기본적으로 **시크릿 노출 방지**를 위해 제한됨. `pull_request_target` 이벤트 사용 시 **권한 상승 위험**이 있으므로 주의(신뢰할 수 없는 코드에 **절대 시크릿 사용/배포 수행 금지**).
4) **브랜치 보호**: 원본은 보통 보호 규칙(리뷰/체크 필수)을 둠. 포크 PR이 이를 충족해야 머지 가능.
5) **라이선스 준수**: 포크는 원본 라이선스를 따라야 함. 변경/배포 시 라이선스 조건을 확인.

---

## 8. Fork 기반 협업: 브랜치/커밋 메시지/PR 템플릿

- 브랜치 네이밍: `feat/…`, `fix/…`, `docs/…` 등 일관되게
- 커밋 메시지: Conventional Commits 권장
- PR 템플릿을 사용해 변경 요약/테스트/이슈 링크를 구조화

`.github/pull_request_template.md`
```markdown
## 요약
-

## 변경 유형
- [ ] 기능 추가
- [ ] 버그 수정
- [ ] 문서
- [ ] 리팩터링

## 테스트
-

## 관련 이슈
Closes #<id>
```

---

## 9. 다중 원격 시나리오: origin(내 포크) + upstream(원본) + 또 다른 remote

```bash
# 현재 원격 확인
git remote -v

# 추가 원격(예: 개인 실험용 레포)
git remote add personal git@github.com:myusername/experiment.git

# 특정 브랜치만 다른 원격으로 push
git push personal my-prototype-branch:my-prototype-branch
```

---

## 10. 충돌 해결/리베이스/체리픽(포크 워크플로우에서 자주 발생)

### 10.1 리베이스 중 충돌 처리
```bash
git rebase upstream/main
# <<<<<<<, >>>>>>> 마커가 있는 파일들 수동 수정
git add <file>
git rebase --continue
```

### 10.2 커밋 정리(히스토리 청결)
```bash
git rebase -i HEAD~5
# pick/squash/reword 활용
```

### 10.3 특정 커밋만 가져오기(체리픽)
```bash
git checkout feature/x
git cherry-pick <commit-sha-from-upstream-or-other-branch>
```

---

## 11. gh(GitHub CLI)로 빠르게 Fork/PR 자동화

### 11.1 포크 + 클론(한 번에)
```bash
# 원본 저장소 디렉터리 없이 바로 실행 가능
gh repo fork original-owner/repo --clone
cd repo
gh repo set-default origin
```

### 11.2 브랜치/PR
```bash
git checkout -b feat/new-ui
# 작업 후
git add .
git commit -m "feat(ui): redesign header"
git push -u origin feat/new-ui

gh pr create --fill --base main --head myusername:feat/new-ui
gh pr view --web
```

### 11.3 upstream 동기화(gh 확장 기능 또는 수동)
```bash
git fetch upstream
git checkout main
git rebase upstream/main
git push -f origin main
```

---

## 12. 포크에서 CI 실행 패턴과 보안

- PR 이벤트: `pull_request`는 포크에서 온 변경에 대해 **읽기 권한** 수준으로 워크플로우 실행(시크릿 접근 제한).
- 배포/민감 작업 필요 시: **메인 저장소 브랜치**에서만 실행되도록 조건부 처리.

예시: `.github/workflows/ci.yml`
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
- 포크 PR에는 `deploy` 가 실행되지 않도록 조건을 둔다.

---

## 13. 실전 시나리오별 “정답 스크립트”

### 13.1 “포크는 했고, 내 포크를 최신 upstream으로 맞추고 싶다”
```bash
git remote -v
# upstream 없으면 등록
git remote add upstream https://github.com/original-owner/repo.git

# 최신 동기화
git fetch upstream
git checkout main
git rebase upstream/main
git push -f origin main
```

### 13.2 “작업 브랜치를 upstream 최신 main 위로 올리고 싶다”
```bash
git checkout feature/my-work
git fetch upstream
git rebase upstream/main
# 충돌 해결 후
git push -f origin feature/my-work
```

### 13.3 “원본 저장소를 바로 clone했는데 push 권한이 없다”
```bash
# 1. GitHub에서 원본을 Fork
# 2. 로컬 원격 교체(내 포크로)
git remote set-url origin git@github.com:myusername/repo.git
# 또는 새로 내 포크를 clone한 뒤 기존 작업을 옮긴다(브랜치/패치 등)
```

### 13.4 “내 포크에서 원본으로 PR 보내려면?”
- 내 포크 브랜치 `feature/x`를 origin에 push
```bash
git push -u origin feature/x
```
- GitHub에서 **Compare & pull request** → base: `original-owner/repo` 의 `main`, compare: `myusername/repo` 의 `feature/x`

---

## 14. 고급: 유지보수자 관점(원본 저장소 운영 팁)

- **포크 대상 가이드**: `CONTRIBUTING.md`에 포크/업스트림 동기화/테스트 지침 명확화
- **PR 안전 가드**: 브랜치 보호(리뷰/필수 체크), 자동 라벨링, 코드 오너(CODEOWNERS)
- **CI 보안**: 포크 PR은 시크릿 차단, 배포 금지. `pull_request_target` 사용 시는 **권한 상승 공격 주의**
- **초보 기여자 배려**: 이슈 템플릿/PR 템플릿/라벨 정의로 진입장벽 완화

---

## 15. 예제: 최초 Fork부터 첫 PR까지 전체 로그

```bash
# 0. GitHub 웹에서 Fork 수행 후
git clone git@github.com:myusername/awesome-lib.git
cd awesome-lib

# 1. upstream 추가
git remote add upstream https://github.com/original-owner/awesome-lib.git
git fetch upstream

# 2. 로컬 main을 upstream/main으로 맞춤
git checkout main
git rebase upstream/main
git push -f origin main

# 3. 작업 브랜치 생성
git checkout -b fix/typo-in-docs

# 4. 수정 및 커밋
sed -i 's/Awseome/Awesome/g' docs/intro.md
git add docs/intro.md
git commit -m "docs: fix typo in intro"

# 5. 내 포크로 push
git push -u origin fix/typo-in-docs

# 6. GitHub에서 PR 생성(또는 gh CLI)
# gh pr create --fill --base main --head myusername:fix/typo-in-docs
```

---

## 16. Template와의 선택 기준(실무 결론)

- **기여**가 목적 → **Fork**
- **새 프로젝트 출발**(구조/보일러플레이트만 복사) → **Template repository**
- **온전한 이력/권한/참조 포함 이전** → **Mirror**

---

## 17. 자주 묻는 질문(FAQ)

**Q1. 포크를 삭제해도 원본에는 영향이 없나?**
A. 없다. 포크는 **내 계정의 독립 저장소**일 뿐, 원본과 느슨하게 연결된 상태다.

**Q2. 포크한 저장소에서 만든 이슈/디스커션은 원본과 공유되나?**
A. 일반적으로 **각 저장소 별도**. 다만 PR은 원본에 생성하므로 원본의 워크플로우와 규칙을 따른다.

**Q3. 포크한 저장소를 프라이빗으로 바꿀 수 있나?**
A. 가능하나, **라이선스/정책**을 확인해야 한다. 조직 정책에 따라 제한될 수 있다.

**Q4. 포크에서 Actions가 안 돌아가거나 시크릿에 접근 못 한다**
A. 정상이다. 포크 PR에는 시크릿이 공유되지 않는다. 배포/시크릿 요구 작업은 메인 저장소에서만 수행되도록 설계해야 한다.

---

## 18. 요약 체크리스트

- **origin=내 포크**, **upstream=원본** 으로 원격을 분리한다.
- 작업 전/PR 전에는 **항상 upstream/main → 내 main 리베이스/머지**로 최신화한다.
- 포크 PR의 CI/보안 정책을 이해하고(시크릿 차단), 배포는 메인 저장소에서만 실행되게 한다.
- 브랜치/커밋/PR 템플릿을 일관되게 사용한다.

---

## 19. 참고 명령어(기존 목록 확장)

```bash
# Fork 후 로컬 clone
git clone git@github.com:myusername/forked-repo.git
cd forked-repo

# 원본 저장소 remote 등록 (업스트림 추가)
git remote add upstream https://github.com/original-owner/repo.git
git remote -v

# 원본 최신 반영(rebase)
git fetch upstream
git checkout main
git rebase upstream/main
git push -f origin main

# 작업 브랜치
git checkout -b feature/x
git add .
git commit -m "feat: x"
git push -u origin feature/x

# PR 생성(웹 또는 gh)
# gh pr create --fill --base main --head myusername:feature/x
```

---

## 20. 참고 자료

- GitHub 공식: Fork a repo
  https://docs.github.com/en/get-started/quickstart/fork-a-repo
- Git: git-clone
  https://git-scm.com/docs/git-clone
- GitHub Actions 보안 모범 사례(포크 PR 관련)
  https://docs.github.com/actions/security-guides
