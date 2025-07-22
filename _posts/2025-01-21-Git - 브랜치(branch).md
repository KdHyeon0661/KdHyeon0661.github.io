---
layout: post
title: Git - SSH 키
date: 2025-01-14 19:20:23 +0900
category: Git
---
# Git 브랜치 생성 및 관리  
> merge 없이 브랜치 생성과 이동, 삭제 중심으로

---

## 🌿 1. 브랜치란?

**브랜치(Branch)**는 코드 변경 이력을 나눠서 관리할 수 있는 **별도의 작업 공간**입니다.  
`main`(또는 `master`) 외에 기능 단위로 브랜치를 만들어 안전하게 실험하거나 개발할 수 있습니다.

---

## 🖥️ 2. GitHub (웹)에서 브랜치 생성하기

1. GitHub 저장소에 접속
2. 좌측 상단 **"main ▼"** 클릭
3. 브랜치 이름 입력 → **"Create branch: 새로운이름"** 클릭

> 💡 GitHub에서 브랜치를 생성하면 서버에서만 존재하며, 로컬에는 자동으로 생기지 않음

---

## 💻 3. 로컬에서 브랜치 생성 및 전환

### ✅ 브랜치 생성

```bash
git branch feature/login
```

> 현재 브랜치에 기반하여 `feature/login` 브랜치 생성 (자동 전환은 아님)

### 🔀 브랜치 전환

```bash
git checkout feature/login
```

또는 생성과 동시에 전환:

```bash
git checkout -b feature/login
```

---

## 🧭 4. 현재 브랜치 확인

```bash
git branch
```

- 현재 브랜치는 `*` 표시됨

---

## 🔄 5. 원격 브랜치 확인 및 추적 설정

```bash
git branch -r            # 원격 브랜치 목록 확인
git checkout -b dev origin/dev  # 원격 브랜치 추적 시작
```

> `origin/dev` 브랜치를 로컬에서 `dev`라는 이름으로 사용하며 추적하도록 설정

---

## 📤 6. 새 브랜치 푸시하기 (원격으로 업로드)

```bash
git push -u origin feature/login
```

- `-u` 옵션은 해당 브랜치를 원격과 연동(tracking)함  
- 이후부터는 `git push`만 입력해도 자동으로 푸시됨

---

## 🗑️ 7. 브랜치 삭제

### 로컬 브랜치 삭제

```bash
git branch -d feature/login      # 병합된 경우만 삭제됨
git branch -D feature/login      # 강제 삭제
```

### 원격 브랜치 삭제

```bash
git push origin --delete feature/login
```

---

## 🧠 8. 브랜치 네이밍 규칙 (권장)

| 용도        | 예시                      |
|-------------|---------------------------|
| 기능 개발   | `feature/login`, `feature/signup` |
| 버그 수정   | `bugfix/image-crash`      |
| 리팩토링    | `refactor/header-style`   |
| 실험용 브랜치 | `test/new-api`             |
| 릴리즈      | `release/v1.2.0`          |

---

## 🧪 9. 브랜치 관련 명령 요약

```bash
git branch                # 브랜치 목록 확인
git branch 새브랜치이름   # 브랜치 생성
git checkout 브랜치이름   # 브랜치 전환
git checkout -b 이름      # 생성 + 전환
git push -u origin 이름   # 브랜치 푸시
git branch -d 이름        # 브랜치 삭제 (병합된 경우만)
git branch -D 이름        # 브랜치 강제 삭제
git push origin --delete 이름  # 원격 브랜치 삭제
```

---

## 📎 참고

- 브랜치 관리 공식 문서: https://git-scm.com/docs/git-branch
- GitHub 브랜치 가이드: https://docs.github.com/en/get-started/quickstart/github-flow