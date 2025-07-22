---
layout: post
title: Git - SSH 키
date: 2025-01-14 19:20:23 +0900
category: Git
---
# Rebase & Cherry-pick 충돌 해결 예제

---

## 🎯 전제 조건: 충돌이란?

두 브랜치가 **같은 파일의 같은 라인**을 서로 다르게 수정한 경우, Git은 자동 병합을 하지 못하고 충돌로 간주합니다.

---

## 🌿 1. Rebase 충돌 해결 예제

### 📁 상황:

- `main` 브랜치: `header.html`에 `<h1>홈페이지</h1>`
- `feature/banner` 브랜치: 같은 라인을 `<h1>배너 페이지</h1>`로 수정
- `main`의 최신 커밋을 기준으로 `feature/banner`를 rebase

---

### 🧪 시나리오 실행:

```bash
git checkout feature/banner
git rebase main
```

### ⚠️ 충돌 발생:

```bash
Auto-merging header.html
CONFLICT (content): Merge conflict in header.html
error: could not apply abc1234... 배너 제목 수정
```

---

### 🧹 해결 절차:

1. 충돌 파일 열기 (`header.html`):

```html
<<<<<<< HEAD
<h1>홈페이지</h1>
=======
<h1>배너 페이지</h1>
>>>>>>> abc1234 (배너 제목 수정)
```

2. 원하는 내용으로 직접 수정:

```html
<h1>배너 페이지</h1>
```

3. 변경 사항 스테이징 후 rebase 계속:

```bash
git add header.html
git rebase --continue
```

---

### 🔁 기타 명령어:

- `git rebase --abort`: rebase 중단
- `git status`: 충돌 상태 확인

---

## 🍒 2. Cherry-pick 충돌 해결 예제

### 📁 상황:

- 현재 브랜치: `main`
- `dev` 브랜치에 있는 특정 커밋을 가져오려 함
- `dev`에서 `footer.html`의 동일한 라인이 `main`과 다르게 수정됨

---

### 🧪 실행:

```bash
git checkout main
git cherry-pick bcd5678
```

### ⚠️ 충돌 발생:

```bash
Auto-merging footer.html
CONFLICT (content): Merge conflict in footer.html
error: could not apply bcd5678... 푸터 문구 변경
```

---

### 🧹 해결 절차:

1. 충돌 파일 열기 (`footer.html`):

```html
<<<<<<< HEAD
<p>© 2025 MyCompany</p>
=======
<p>© 2025 DevTeam</p>
>>>>>>> bcd5678 (푸터 문구 변경)
```

2. 수정 (선택 예시):

```html
<p>© 2025 MyCompany | DevTeam</p>
```

3. 변경 사항 스테이징 후 cherry-pick 계속:

```bash
git add footer.html
git cherry-pick --continue
```

---

## 🔍 Rebase vs Cherry-pick 충돌 비교

| 항목 | Rebase | Cherry-pick |
|------|--------|-------------|
| 충돌 발생 시점 | **모든 커밋마다 개별로 충돌 발생** | 선택한 **특정 커밋에서만 발생** |
| 해결 명령어 | `rebase --continue`, `--abort` | `cherry-pick --continue`, `--abort` |
| 일반 사용 흐름 | 브랜치 히스토리를 정리할 때 | 특정 커밋만 가져올 때 |

---

## 📎 실전 팁

- `git status` 명령으로 충돌 파일을 쉽게 확인할 수 있음
- 충돌 수정 시 **에디터에서 conflict 마커 (`<<<<<<<`, `=======`, `>>>>>>>`) 제거 필수**
- 충돌 난 파일만 `git add`, 나머지는 그대로 둬도 OK
- 충돌 해결 후 커밋 메시지는 cherry-pick/rebase가 자동 생성

---

## 🔗 참고 링크

- Git Conflict resolution 공식: https://git-scm.com/docs/git-merge#_how_conflicts_are_presented
- Git cherry-pick 문서: https://git-scm.com/docs/git-cherry-pick
- Git rebase 문서: https://git-scm.com/docs/git-rebase