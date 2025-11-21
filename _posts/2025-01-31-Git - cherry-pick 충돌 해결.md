---
layout: post
title: Git - cherry-pick 충돌 해결
date: 2025-01-31 21:20:23 +0900
category: Git
---
# Rebase & Cherry-pick **충돌 해결 예제**

## 전제: “충돌”이란?

두 브랜치가 **같은 파일의 같은(혹은 겹치는) 영역**을 서로 다르게 수정하면 Git이 자동 병합하지 못하고 **충돌(conflict)** 으로 분류합니다.
충돌은 rebase/merge/cherry-pick 등 “패치 적용”이 일어나는 모든 상황에서 발생할 수 있습니다.

---

## 빠른 맛보기 — 사용자의 기본 예제(복습)

### Rebase 충돌 예제(요지)

- `main`: `header.html`에 `<h1>홈페이지</h1>`
- `feature/banner`: 같은 줄을 `<h1>배너 페이지</h1>`로 수정
- `feature/banner`에서 `git rebase main` → 충돌
- 해결: 수동 수정 → `git add` → `git rebase --continue`

### Cherry-pick 충돌 예제(요지)

- 현재 `main`
- `dev`의 커밋 `bcd5678`를 `git cherry-pick bcd5678`
- `footer.html`의 같은 줄이 다름 → 충돌
- 해결: 수동 수정 → `git add` → `git cherry-pick --continue`

본 글은 위 흐름을 **확장**하여, 실제로 **로컬에서 그대로 재현** 가능하도록 **스크립트**부터 제공합니다.

---

## 재현 가능한 실습 레포 — 스크립트 한 번으로 환경 구성

아래 명령을 새 폴더에서 그대로 실행하면, rebase/cherry-pick 충돌을 순차 실습할 수 있는 레포가 만들어집니다.

```bash
# 실습 0: 깨끗한 작업 디렉터리 준비

rm -rf rbc-lab && mkdir rbc-lab && cd rbc-lab
git init
git config user.name  "lab"
git config user.email "lab@example.com"
git branch -M main

# 공통 파일 초기화

echo '<h1>홈페이지</h1>' > header.html
echo '<p>© 2025 MyCompany</p>' > footer.html
git add . && git commit -m "init: header/homepage, footer/company"

# feature/banner 브랜치에서 header 충돌 유발

git checkout -b feature/banner
# 같은 줄을 다른 값으로 수정

echo '<h1>배너 페이지</h1>' > header.html
git commit -am "feat(banner): header as banner page"

# main에서도 header 같은 줄을 바꿔 충돌 만들기

git checkout main
echo '<h1>홈페이지 메인</h1>' > header.html
git commit -am "feat(main): header homepage main"

# ---------- Rebase 충돌 테스트 ----------

git checkout feature/banner
git rebase main || true    # 충돌 발생을 위해 실패 허용
# 충돌 확인

git status
# 이 상태에서 수동 해결 연습(본문 §3 참고)

# ---------- Cherry-pick 충돌 준비 ----------
# dev 브랜치에서 footer를 devTeam으로 변경

git checkout -b dev main
echo '<p>© 2025 DevTeam</p>' > footer.html
git commit -am "feat(dev): footer devteam"

# main에서 cherry-pick 시 충돌 만들기(사전: main의 footer 수정)

git checkout main
echo '<p>© 2025 MyCompany</p>' > footer.html
git commit -am "feat(main): footer mycompany confirm"

# ---------- Cherry-pick 충돌 테스트 ----------

git cherry-pick dev~0 || true
# 충돌 확인

git status
```

> 위 스크립트 실행 후, 이어지는 절차(§3, §4)를 따라 충돌을 해결하세요.

---

## Rebase 충돌 해결 — 단계별 상세 가이드

### 충돌 감지와 파일 열기

```bash
git rebase main
# ...
# CONFLICT (content): Merge conflict in header.html

git status
```

`header.html` 을 열면 충돌 마커를 확인할 수 있습니다.

```html
<<<<<<< HEAD
<h1>홈페이지 메인</h1>
=======
<h1>배너 페이지</h1>
>>>>>>> abc1234 (feat(banner): header as banner page)
```

- `<<<<<<< HEAD` ~ `=======` 는 현재 커밋(적용 대상 경로, rebase 시 “새 베이스 기준 상태”)
- `=======` ~ `>>>>>>> <commit>` 는 가져오려는 커밋(적용 중인 패치의 변경)

### 원하는 결과로 수동 통합

예1) 배너 문구를 유지:
```html
<h1>배너 페이지</h1>
```

예2) 혼합/합성:
```html
<h1>홈페이지 메인 | 배너 페이지</h1>
```

### 스테이징 후 계속

```bash
git add header.html
git rebase --continue
```

반복해서 다른 커밋에서 또 충돌이 나면 **같은 절차**를 반복합니다.

### 중단/취소/건너뛰기

```bash
git rebase --abort    # rebase 시작 전으로 복귀
git rebase --skip     # 현재 커밋을 건너뛰고 다음으로
```

> 실수 방지: rebase는 **여러 커밋**을 순차 적용하므로, 충돌은 **여러 번** 발생할 수 있습니다.

---

## Cherry-pick 충돌 해결 — 단계별 상세 가이드

### 충돌 감지와 파일 열기

```bash
git cherry-pick <sha>   # 예: dev~0
# ...
# CONFLICT (content): Merge conflict in footer.html

git status
```

`footer.html`:

```html
<<<<<<< HEAD
<p>© 2025 MyCompany</p>
=======
<p>© 2025 DevTeam</p>
>>>>>>> bcd5678 (feat(dev): footer devteam)
```

### 통합 예시

예1) 병합:
```html
<p>© 2025 MyCompany | DevTeam</p>
```

예2) 회사 표기로 단일화:
```html
<p>© 2025 MyCompany</p>
```

### 스테이징 후 계속/중단

```bash
git add footer.html
git cherry-pick --continue

# 또는

git cherry-pick --abort
git cherry-pick --skip
```

> cherry-pick은 **선택한 커밋**의 적용 시점에서만 충돌합니다.

---

## 충돌 유형별 정밀 대응

### 충돌

- 동일 파일 동일(혹은 겹치는) 범위 충돌 → **가장 흔함**
- 해결: 파일 열어 수동 통합 → `git add` → `continue`

### 충돌

- 한쪽 브랜치가 파일명을 변경, 다른 쪽이 동일 파일을 수정한 경우
- 전략:
  1) 최종 파일명을 선택(팀 규칙/컨벤션)
  2) 변경 내용 손실 없이 통합
- 도구:
  ```bash
  git status
  git diff --name-status --diff-filter=R
  git mergetool
  ```

### 충돌

- 한쪽은 파일 삭제, 다른 쪽은 수정
- 전략:
  - 파일 유지: 삭제를 되돌리고 수정 내용 반영
  - 파일 삭제 유지: 수정사항 폐기
- 명령:
  ```bash
  # 현재(ours)를 채택
  git checkout --ours   path/to/file
  # 상대(theirs)를 채택
  git checkout --theirs path/to/file
  git add path/to/file
  ```

### 충돌

- 한쪽은 `path/foo` 파일, 다른 쪽은 `path/`를 디렉터리로 변경(혹은 반대)
- 전략:
  - 구조를 일관되게 정하고 이름을 재배열(예: 파일을 `path/index.html`로 이동)
  - 수동으로 파일/폴더 재조정 후 add

### 충돌

- 이미지/디자인 등 자동 머지 불가
- 선택지:
  ```bash
  git checkout --ours   assets/logo.psd
  # 또는
  git checkout --theirs assets/logo.psd
  git add assets/logo.psd
  ```
- `.gitattributes`로 미리 정책 명시:
  ```
  *.psd -merge -diff
  *.png binary
  ```

### 충돌

- 서로 다른 커밋 포인터를 가리킴
- 절차:
  ```bash
  cd submodules/libA
  git checkout <원하는 커밋>
  cd ../..
  git add submodules/libA   # 상위 리포에서 포인터 커밋
  git commit -m "fix(submodule): point libA to <sha>"
  ```

---

## 시각 도구와 자동화 — `mergetool`, `rerere`

### mergetool

선호하는 GUI/UI 도구로 충돌 지점을 비교/선택:

```bash
git config --global merge.tool meld      # 예: meld
git mergetool
```

### rerere(Resolve REcorded)

이전 충돌 해결 패턴을 기억 → 유사 충돌이 반복될 때 자동 제안:

```bash
git config --global rerere.enabled true
```

> rebase로 같은 충돌이 여러 번 발생할 때 **시간 절약** 가능.

---

## 부분 선택/빠른 선택 — `--ours` / `--theirs`

충돌 파일을 전체적으로 한쪽 버전으로 빠르게 선택:

```bash
# 선택

git checkout --ours path/to/file

# 선택

git checkout --theirs path/to/file

git add path/to/file
```

> 주의: 파일 전체를 한쪽으로 선택합니다. 라인 단위 수동 통합이 더 안전할 때가 많습니다.

---

## 빈 커밋/중복, 건너뛰기, 중단/복구

### 빈 커밋/중복

- 이미 동일 변경이 반영되어 **empty** 상태가 될 수 있음
- 처리:
  ```bash
  # skip 해서 건너뜀
  git cherry-pick --skip
  git rebase --skip

  # 혹은 메시지만 남기고 싶다면
  git commit --allow-empty -m "doc: already applied (note)"
  ```

### 중단/전면 취소

```bash
git cherry-pick --abort
git rebase --abort
```

### 비상 복구 — `reflog`와 스냅샷

```bash
git reflog
git reset --hard ORIG_HEAD
# 또는 특정 시점으로 브랜치 구출

git switch -c rescue HEAD@{3}
```

> 위험 작업 전에는 **백업 브랜치/태그**를 남겨두세요.

---

## Rebase vs Cherry-pick 충돌 — 확장 비교

| 항목 | Rebase 충돌 | Cherry-pick 충돌 |
|---|---|---|
| 발생 빈도 | **여러 커밋**을 순차 적용하므로 반복 발생 가능 | **선택한 커밋** 적용 시점에서만 |
| 해결 난이도 | 반복/누적 가능 → `rerere` 효과 큼 | 단발성인 경우가 많음 |
| 명령어 | `rebase --continue/--abort/--skip` | `cherry-pick --continue/--abort/--skip` |
| 쓰임새 | 브랜치 **정리/선형화** | 특정 수정 **선택적 반영** |

---

## 실전 팁(체크리스트)

- [ ] 충돌 후 **반드시 conflict 마커**(`<<<<<<<`, `=======`, `>>>>>>>`)를 제거하고 저장
- [ ] `git status`로 충돌 파일 목록 확인 → **수정 파일만** `git add`
- [ ] 같은 충돌 반복 시 `rerere`를 켜서 시간 절약
- [ ] 대형/복합 충돌은 `git mergetool`로 시각 비교
- [ ] 바이너리는 **한쪽 선택** 후 팀과 재합의
- [ ] rebase는 **개인 브랜치**에서만, 이후 푸시는 `--force-with-lease`
- [ ] 사전 **백업 브랜치/태그**로 보험
- [ ] 서브모듈은 **포인터 커밋**을 상위에서 명시적으로 커밋

---

## 확장 실습 — 충돌 유형 모두 경험해보기

### 리네임 충돌 재현

```bash
# 준비

rm -rf rename-lab && mkdir rename-lab && cd rename-lab
git init
git config user.name  "lab"
git config user.email "lab@example.com"
git branch -M main

echo 'v1' > file.txt
git add . && git commit -m "init"

# branch A: 파일명 변경 + 내용 수정

git checkout -b A
git mv file.txt main.txt
echo 'A-change' >> main.txt
git add . && git commit -m "A: rename to main.txt + change"

# main: 같은 파일 내용만 수정

git checkout main
echo 'main-change' >> file.txt
git commit -am "main: change file.txt"

# A를 main 위로 rebase → 리네임/수정 충돌 가능

git checkout A
git rebase main || true
git status
# mergetool로 이름/내용 충돌 해소, or 수동 통합

```

### 삭제-수정 충돌 재현

```bash
rm -rf delmod-lab && mkdir delmod-lab && cd delmod-lab
git init && git branch -M main
echo 'keep-me' > note.md
git add . && git commit -m "init"

git checkout -b X
# X에서는 note.md 삭제

git rm note.md
git commit -m "X: delete note.md"

git checkout main
# main에서는 note.md 수정

echo 'main edits' >> note.md
git commit -am "main: edit note.md"

git checkout X
git rebase main || true
git status
# 전략 선택:
#   파일 유지 → git checkout --theirs note.md; git add note.md
#   삭제 유지 → git checkout --ours  note.md; git add note.md
# → git rebase --continue

```

---

## 명령어 치트시트

```bash
# 공통 관찰

git status
git diff
git diff --name-status
git log --oneline --graph --decorate --all

# Rebase 충돌 루틴

git rebase <base>
# 충돌 → 파일 편집

git add <files>
git rebase --continue
git rebase --skip
git rebase --abort

# Cherry-pick 충돌 루틴

git cherry-pick <sha1> [<sha2>...]
# 충돌 → 파일 편집

git add <files>
git cherry-pick --continue
git cherry-pick --skip
git cherry-pick --abort

# 빠른 선택(파일 단위)

git checkout --ours   <path>
git checkout --theirs <path>

# 시각 도구/자동화

git config --global merge.tool meld
git mergetool
git config --global rerere.enabled true

# 복구

git reflog
git reset --hard ORIG_HEAD
git switch -c rescue <reflog-entry>
```

---

## 결론

- **Rebase** 충돌은 “여러 커밋”을 순차 적용하므로 반복될 수 있어 **rerere** 효과가 크고,
- **Cherry-pick** 충돌은 “선택한 커밋” 단위로 발생해 단발성인 경우가 많습니다.
- 유형별(내용/리네임/삭제-수정/디렉터리-파일/바이너리/서브모듈) 해결 전술을 숙지하고, **mergetool**과 **rerere**를 병행하면 생산성이 크게 향상됩니다.
- 언제든 실수/후퇴가 필요할 수 있으므로 **`--abort`/`reflog`/백업 브랜치**를 생활화하세요.

---

## 참고

- Git Conflict resolution: https://git-scm.com/docs/git-merge#_how_conflicts_are_presented
- Git cherry-pick: https://git-scm.com/docs/git-cherry-pick
- Git rebase: https://git-scm.com/docs/git-rebase
