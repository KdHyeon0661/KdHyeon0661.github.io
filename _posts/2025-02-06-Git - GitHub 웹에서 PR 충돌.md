---
layout: post
title: Git - GitHub 웹에서 PR 충돌 해결
date: 2025-02-06 19:20:23 +0900
category: Git
---
# GitHub 웹에서 **PR( Pull Request ) 충돌 해결**

## 기본 개념 정리(복습)

- **PR 충돌(merge conflict)**: PR의 **base(기준)** 브랜치와 **compare(비교)** 브랜치가 **같은 파일의 같은/겹치는 영역**을 서로 다르게 수정한 경우 자동 병합이 불가능한 상태.
- GitHub는 PR 상단에 다음과 같은 경고를 표시합니다:

```
This branch has conflicts that must be resolved
```

- 이때 **Merge pull request** 버튼이 비활성화되며, **Resolve conflicts** 버튼 또는 **Update branch** 버튼(설정/권한/정책에 따라 다름)이 노출될 수 있습니다.

---

## 권한과 정책(중요)

웹에서 충돌을 해결하려면 다음 중 하나가 필요합니다.

1. **PR 브랜치에 쓰기 권한**
   - 같은 저장소 내 브랜치인 경우: 보통 **Write** 권한이 있으면 웹에서 **Resolve conflicts** 가능.
   - **fork에서 온 PR**: PR 창 우측 사이드바의 **Allow edits by maintainers** 체크가 활성화되어 있어야, 저장소 유지보수자(Maintainer)가 웹에서 해당 브랜치를 편집 가능.

2. **보호 브랜치 규칙(Protect Branch Rules)**
   - base 브랜치가 보호되는 경우(예: `main`):
     - Required reviews, Required status checks, Linear history(rules), Force-push 금지 등으로 인해 **웹에서 바로 병합/수정이 제한**될 수 있음.
     - 이때는 **로컬에서 충돌 해결 → 푸시 → 상태체크 통과 → PR 병합** 순서로 진행하거나, 정책에 맞춰 **Update branch**(merge commit) 또는 **rebase and merge** 전략을 택해야 함.

3. **필요 권한이 없다면**
   - 본인이 PR 작성자라면 **Allow edits by maintainers** 체크.
   - 아니면 **로컬에서 자신의 fork 브랜치**를 고쳐 다시 푸시(권한 = 자기 fork 에 대한 push).

---

## GitHub 웹에서 충돌 해결 — 단계별 절차(확장)

### Step 1. PR 페이지 접속

- 저장소 → **Pull requests** 탭 → 충돌 표시된 PR 클릭.

### Step 2. 충돌 메시지 확인

- 상단 배너: `This branch has conflicts that must be resolved`
- 폴더별/파일별 충돌 목록 확인
- 버튼:
  - **Resolve conflicts**: 웹 에디터를 열어 라인 충돌 해결.
  - **Update branch**: GitHub가 base를 compare에 **자동 merge commit**으로 가져와 업데이트(설정/권한/정책에 따라 표시·비표시).

### Step 3. 충돌 파일 웹 편집기로 열기

- **Resolve conflicts** 클릭 → GitHub 가 충돌난 파일을 “마커” 포함 상태로 엽니다.
- 충돌 마커 구문:

```text
<<<<<<< HEAD
# base(현재 기준)의 내용

=======
# compare(PR 브랜치)의 내용

>>>>>>> feature/header-update
```

HTML 예시:

```html
<<<<<<< HEAD
<h1>Welcome to Home Page</h1>
=======
<h1>Welcome to Our Service</h1>
>>>>>>> feature/header-update
```

### Step 4. 수동 통합(마커 제거 필수)

- 원하는 최종 결과로 **수정**하고, 모든 충돌 마커(`<<<<<<<`, `=======`, `>>>>>>>`)를 **삭제**해야 합니다.
- 통합 예시:

```html
<h1>Welcome to Our Home Page</h1>
```

### Step 5. 변경 저장·해결 표시

- 페이지 하단 커밋 창에서 메시지를 확인/수정(기본: `Resolve merge conflict`)
- **Mark as resolved** 클릭
- **Commit merge** 버튼 클릭

> **주의**: 보호 브랜치 규칙(Required status checks 등)으로 인해 커밋 후에도 **즉시 병합 불가**일 수 있습니다. CI 통과, 리뷰 승인 등 정책을 충족해야 합니다.

### Step 6. Merge 가능 상태 확인

- PR 화면으로 돌아오면 충돌 배너가 사라지고,
  - `This branch has no conflicts`
  - **Merge pull request** 버튼 활성화
- 팀 정책에 맞는 병합 전략(merge commit / squash and merge / rebase and merge)을 선택해 병합합니다.

---

## 웹 편집기의 **한계**와 대안

웹으로 바로 해결하기 곤란한 상황:

- **리네임(Rename) + 내용 변경**이 얽힌 복합 충돌
- **대량 파일** 충돌(스크롤·편집 어려움)
- **바이너리(이미지/디자인/문서)** 충돌: 라인 병합 불가
- **서브모듈** 포인터 충돌: 상위 리포에서 포인터 커밋을 업데이트해야 함
- **보호 브랜치 정책**으로 웹에서 해결/커밋이 차단되는 경우

이때는 **로컬에서 충돌 해결 후 푸시**가 권장됩니다(다음 장 참조).

---

## 로컬에서 충돌 해결 후 푸시(대안 절차 강화)

### merge-기반 업데이트(가장 단순)

```bash
# 기준 브랜치 업데이트

git checkout main
git pull origin main

# PR 브랜치로 전환

git checkout feature/header-update

# 기준 브랜치를 merge하여 충돌 확인

git merge main

# 충돌 파일 수정 → 마커 삭제

git add <수정파일들>
git commit

# PR 브랜치로 푸시

git push
```

- GitHub PR 화면에서 자동으로 “conflict resolved” 상태가 반영됩니다.

### rebase-기반 업데이트(선형 이력 선호 시)

```bash
git checkout feature/header-update
git fetch origin
git rebase origin/main
# 충돌 시 파일 수정 → git add → git rebase --continue 반복

git push --force-with-lease
```

- **주의**: rebase는 해시 재작성 → 원격에는 **강제 푸시** 필요.
- PR이 **fork에서 온 경우**에도 동일. 단, 팀 정책/권한에 따라 rebase가 금지될 수 있음.

### 부분 채택(ours / theirs) 단축 키

```bash
# 현재 브랜치(ours) 버전 채택

git checkout --ours   path/to/file
# 가져오는 쪽(theirs) 버전 채택

git checkout --theirs path/to/file
git add path/to/file
```

### 시각 편집 도구(mergetool)

```bash
git config --global merge.tool meld     # 예: meld
git mergetool
```

### 반복 충돌 자동 학습(rerere)

```bash
git config --global rerere.enabled true
```

---

## 다양한 충돌 유형 예시와 해결 전략

### 내용(content) 충돌

- 동일 파일 동일/겹치는 줄 충돌
- 해결: **라인 단위** 수동 통합 → 마커 삭제 → add → 커밋

### 리네임(rename) + 수정 충돌

- 한쪽에서 파일명 변경, 다른 쪽에서 같은 파일을 수정
- 웹에서는 맥락 파악이 힘들 수 있어 **로컬 권장**
- 전략: 최종 파일명 선택 → 내용 합치기 → **이력 검증**

### 삭제-수정(delete/modify) 충돌

- 한쪽은 파일 삭제, 다른 쪽은 수정
- 유지 여부 결정 후 `--ours/--theirs` 또는 수동 복원

### 바이너리(binary) 충돌

- 이미지/모델/디자인 등은 라인 병합 불가
- 선택지: ours 혹은 theirs로 **전체 파일 단위** 채택 → 팀 재합의
- `.gitattributes`로 병합/비교 정책 명시:
  ```
  *.psd -merge -diff
  *.png binary
  ```

### 서브모듈(submodule) 포인터 충돌

- 상위 리포에서 서브모듈 폴더가 가리키는 커밋 포인터를 **명시적으로 갱신** 후 커밋

---

## 조직/팀 설정과 “Update branch”

- **Update branch** 버튼은 GitHub가 **자동으로 base를 compare에 merge**하여 업데이트하는 기능입니다(merge commit 생성).
- 저장소/조직 정책(Linear history, Required status checks)에 따라:
  - 버튼이 **노출되지 않거나**,
  - 클릭 시 특정 정책 위반으로 **거부**될 수 있습니다.
- 선형 이력 강제(Require linear history) 환경에서는 **merge commit 금지** → 로컬에서 **rebase 후 푸시**가 필요.

---

## 완전 재현 가능한 **로컬 시뮬레이션**(학습용)

아래 스크립트는 로컬에서 “PR 충돌” 상황을 **모사**하고, 웹/로컬 해결 과정을 **연습**하기 위한 것입니다(실제 GitHub PR은 gh나 웹으로 생성).

```bash
# 깨끗한 디렉터리

rm -rf pr-conflict-lab && mkdir pr-conflict-lab && cd pr-conflict-lab
git init
git config user.name  "lab"
git config user.email "lab@example.com"
git branch -M main

# 초기 파일

mkdir src
echo '<h1>Welcome to Home Page</h1>' > src/header.html
echo '<p>© 2025 MyCompany</p>'        > src/footer.html
git add . && git commit -m "init: header/home, footer/company"

# feature 브랜치 생성 및 변경

git checkout -b feature/header-update
echo '<h1>Welcome to Our Service</h1>' > src/header.html
git commit -am "feat: service heading"

# main에서도 같은 라인을 바꿔 충돌 유발

git checkout main
echo '<h1>Welcome to Home Page v2</h1>' > src/header.html
git commit -am "feat: home v2"

# PR을 올렸다고 가정하고(웹에서), 이제 충돌을 해결해야 하는 상황
#    웹에서 Resolve conflicts(라인 통합)를 할 수도 있고,
#    로컬에서 아래처럼 해결할 수도 있음.

# --- 로컬 해결(merge 기반) ---

git checkout feature/header-update
git merge main || true     # 충돌 유발
git status                 # 충돌 파일 확인
# src/header.html 열어 수동 통합 후:
# <h1>Welcome to Our Home Page</h1> 같이 합의한 문구로 수정

git add src/header.html
git commit -m "resolve: header conflict"
git log --oneline --graph --decorate --all
# git push (원격이 있다고 가정)

# --- 또는 rebase 기반 ---

git checkout feature/header-update
git rebase main || true
# 파일 수정 → add → rebase --continue

git push --force-with-lease
```

> 실제 GitHub PR까지 자동화하려면 **GitHub CLI(gh)** 를 사용합니다:
> ```bash
> gh repo create <owner>/<repo> --public --source=. --push
> gh pr create --fill --base main --head feature/header-update
> ```

---

## 트러블슈팅

| 증상 | 원인 | 해결 |
|---|---|---|
| Resolve conflicts 버튼이 안 보임 | 쓰기 권한 없음 / fork PR에서 maintainer edit 허용 안됨 | PR 사이드바에서 **Allow edits by maintainers** 체크 또는 로컬에서 해결 후 푸시 |
| Commit merge 클릭 후에도 병합 불가 | 보호 브랜치 정책(리뷰/체크 미통과/선형 이력 강제) | 리뷰 통과/체크 통과 대기, rebase 후 `--force-with-lease`, 정책에 맞는 병합 전략 사용 |
| Update branch 버튼 없음 | 저장소 설정/정책상 merge 업데이트 비허용 | 로컬에서 merge/rebase 후 푸시 |
| 대량/복합 충돌로 웹 편집기에서 힘듦 | 리네임/바이너리/서브모듈/수십 파일 충돌 | 로컬에서 mergetool·ours/theirs·rerere 활용 |
| rebase 후 push 거부 | 해시 재작성으로 원격과 불일치 | `git push --force-with-lease` |
| 해결했는데도 CI가 실패 | 포맷터/테스트/린트 자동체크 실패 | 로컬에서 빌드/테스트/포맷 확인 후 수정 재푸시 |

---

## 체크리스트

- [ ] 라인 충돌 해결 후 **마커**(`<<<<<<<`, `=======`, `>>>>>>>`)는 반드시 제거
- [ ] **보호 브랜치 정책** 확인(필수 리뷰/체크/선형 이력/force-push 금지)
- [ ] fork PR이면 **Allow edits by maintainers** 사용 여부 확인
- [ ] 대량/바이너리/리네임 충돌은 **로컬**에서 mergetool·ours/theirs로 해결
- [ ] rebase 후 푸시는 **`--force-with-lease`** 사용(동료 커밋 보호)
- [ ] 반복 충돌은 **`rerere.enabled=true`** 로 시간 절약
- [ ] 병합 직전, CI/테스트/포맷 상태 확인

---

## 로컬 절차 요약(명령어 모음)

```bash
# 기준 업데이트

git checkout main
git pull origin main

# PR 브랜치 전환

git checkout feature-branch

# A) merge 기반

git merge main
# 충돌 해결 → 파일수정 → 마커 삭제

git add <files>
git commit
git push

# B) rebase 기반(선형 이력)

git fetch origin
git rebase origin/main
# 충돌 해결 루틴

git add <files>
git rebase --continue
git push --force-with-lease

# 도우미

git status
git diff --name-status
git checkout --ours   <path>
git checkout --theirs <path>
git mergetool
git config --global rerere.enabled true
```

---

## 결론

- GitHub **웹 편집기**만으로도 단순 라인 충돌은 빠르게 해결할 수 있지만,
  **리네임·바이너리·대량 충돌·보호 브랜치 정책**이 얽히면 **로컬에서 해결 후 푸시**가 안정적입니다.
- 팀 정책(Required reviews, Status checks, Linear history)을 미리 숙지하고, 필요 시 **Update branch** / **rebase** 전략을 적절히 선택하세요.
- 실수 방지를 위해 `--force-with-lease`, `mergetool`, `rerere`, 체크리스트를 습관화하면 PR 충돌 처리 속도와 안정성이 크게 향상됩니다.

---

## 참고

- GitHub 공식: Addressing merge conflicts in a pull request
  https://docs.github.com/en/pull-requests/collaborating-with-pull-requests/addressing-merge-conflicts/about-merge-conflicts

- Git(merge) — 충돌 표현
  https://git-scm.com/docs/git-merge#_how_conflicts_are_presented
