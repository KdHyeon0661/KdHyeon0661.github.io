---
layout: post
title: Git - GitHub 웹에서 PR 충돌 해결
date: 2025-02-06 19:20:23 +0900
category: Git
---
# 🔧 GitHub 웹에서 PR(Pull Request) 충돌 해결 가이드

---

## 🧠 1. PR 충돌이란?

Pull Request를 보낼 때, **기준 브랜치(base)**와 **비교 브랜치(compare)** 간에  
**동일 파일의 같은 부분이 다르게 변경**되었다면 GitHub는 **자동 병합이 불가능**하다고 판단하고 "Merge conflict"을 표시합니다.

---

## 🚨 2. 충돌 상태 인식

PR 화면 상단에 아래 메시지가 표시됩니다:

```
This branch has conflicts that must be resolved
```

그리고 "Merge pull request" 버튼이 비활성화됩니다.

---

## 🧭 3. 웹에서 충돌 해결: 단계별 절차

### ✅ Step 1. PR 페이지 접속

- GitHub 저장소 → **Pull requests** 탭 → 충돌 발생한 PR 클릭

### ✅ Step 2. 충돌 메시지 확인

- "This branch has conflicts that must be resolved"
- 아래에 `Resolve conflicts` 버튼이 보임 → 클릭

### ✅ Step 3. 웹 편집기에서 충돌 파일 수정

GitHub는 자동으로 충돌난 파일의 편집기를 열어줍니다.  
파일 내에 `<<<<<<<`, `=======`, `>>>>>>>` 마커가 표시됩니다.

#### 예시:

```html
<<<<<<< HEAD
<h1>Welcome to Home Page</h1>
=======
<h1>Welcome to Our Service</h1>
>>>>>>> feature/header-update
```

### ✅ Step 4. 충돌 수동 해결

원하는 내용으로 수정:

```html
<h1>Welcome to Our Home Page</h1>
```

**주의**: conflict 마커 (`<<<<<<<`, `=======`, `>>>>>>>`)를 **꼭 삭제**해야 합니다.

---

### ✅ Step 5. 변경 저장 및 커밋

- 수정 완료 후, 페이지 하단에 있는 커밋 창에서 메시지를 확인하거나 수정
- 기본 메시지: `"Resolve merge conflict"`
- ✅ `Mark as resolved` 클릭
- ✅ `Commit merge` 버튼 클릭

---

### ✅ Step 6. Merge 버튼 활성화 확인

- 충돌이 해결되면 다시 PR로 돌아가면 `"This branch has no conflicts"` 메시지로 변경됨
- `Merge pull request` 버튼이 활성화됨 → 병합 가능

---

## 🧑‍💻 4. 로컬에서 충돌 해결 후 푸시 (대안)

웹 에디터가 복잡한 경우, 로컬에서 수동 해결 후 푸시할 수 있습니다:

```bash
# 기준 브랜치 업데이트
git checkout main
git pull origin main

# PR 브랜치로 전환
git checkout feature/header-update

# 기준 브랜치를 merge하여 충돌 발생
git merge main
# 충돌 파일 수정
git add .
git commit
git push
```

→ 이후 GitHub에서 충돌이 해결된 상태로 표시됨

---

## 💡 5. 실전 팁

| 항목 | 설명 |
|------|------|
| conflict 해결 마커는 반드시 삭제 | `<<<<<<<`, `=======`, `>>>>>>>` |
| PR 브랜치에 직접 쓰기 권한 필요 | 아니면 로컬에서 fork 후 PR |
| commit 메시지 자동 생성 | `"Resolve conflict"` 또는 `"Merge branch 'main' into..."` |

---

## 🧪 6. conflict가 잘 생기는 대표 예시

- 같은 파일의 제목, 설명 등을 여러 브랜치에서 동시에 수정했을 때
- 코드 포맷터(Prettier 등)로 자동 정렬한 경우
- 동일한 목록, 배열, 객체를 각기 수정한 경우

---

## 🔗 참고 링크

- GitHub 공식 PR 충돌 해결 가이드:  
  https://docs.github.com/en/pull-requests/collaborating-with-pull-requests/addressing-merge-conflicts/about-merge-conflicts

- Git conflict 공식 문서:  
  https://git-scm.com/docs/git-merge#_how_conflicts_are_presented

---

## 📎 명령어 요약 (로컬 해결 시)

```bash
git checkout feature-branch
git merge main                     # 충돌 유발
# 파일 열어 수정
git add 수정한파일
git commit
git push
```

---

## ✅ 마무리 요약

1. PR에서 `Resolve conflicts` 클릭  
2. 마커(`<<<<<<<`) 삭제하고 수정  
3. `Mark as resolved` + `Commit merge`  
4. PR 화면으로 돌아가 `Merge pull request` 버튼 클릭  