---
layout: post
title: Git - 개요
date: 2025-01-14 19:20:23 +0900
category: Git
---
# GitHub의 Fork란 무엇인가?  
## 그리고 Fork vs Clone 비교

---

## 🪞 1. Fork란?

**Fork(포크)**는 **다른 사용자의 GitHub 저장소를 자신의 계정으로 복사해오는 기능**입니다.

> 즉, 원본 저장소를 그대로 복사하여 **내 계정에서 독립된 저장소로 관리**할 수 있게 합니다.

### ✅ 특징
- GitHub 상에서 버튼 하나로 수행
- 원본 저장소의 히스토리와 파일을 모두 복사
- 자신의 저장소이기 때문에 `push`가 자유로움
- 원본 저장소와 연결된 상태 유지 (Pull Request 가능)

---

## 📦 2. Fork 사용 예시

### 예: 오픈소스 프로젝트 기여 흐름

1. GitHub에서 오픈소스 저장소를 **Fork**
2. 내 저장소에서 코드 수정
3. 수정한 내용을 원본 저장소에 **Pull Request**로 제안

```bash
# 1. GitHub 웹에서 Fork 클릭
# 2. 내 계정으로 포크된 저장소 clone
git clone git@github.com:myusername/repo-name.git

# 3. 작업 후 커밋 & push
git add .
git commit -m "기능 추가"
git push origin main

# 4. GitHub에서 Pull Request 생성
```

---

## 🔁 3. Clone이란?

`git clone`은 **원격 저장소를 내 로컬 디렉토리로 복사**하는 명령어입니다.

- 로컬에서 Git 작업을 할 수 있도록 설정
- 복제한 저장소에는 `origin` 원격 주소가 연결됨
- clone만으로는 GitHub 저장소를 소유하지 않음 → `push` 불가능할 수도 있음

---

## ⚔️ 4. Fork vs Clone 차이

| 항목 | **Fork** | **Clone** |
|------|----------|-----------|
| 개념 | GitHub 저장소를 **내 계정으로 복사** | 원격 저장소를 **내 컴퓨터로 복사** |
| 위치 | GitHub 웹에서 사용 | 로컬에서 터미널로 사용 |
| 목적 | 오픈소스에 기여, 독립 저장소 생성 | 저장소 내려받아 작업 시작 |
| 권한 | 복사한 저장소의 **owner 권한** | 원본 저장소에 따라 **읽기 전용일 수도 있음** |
| Pull Request | 원본 저장소에 기여 가능 | PR 보내려면 별도로 Fork 필요 |
| 변경 반영 | GitHub 상에 존재 | 로컬에서만 존재 |
| push 가능 여부 | 가능 (내 저장소니까) | 원격 저장소 권한에 따라 다름 |

---

## 🧭 5. 언제 Fork, 언제 Clone?

| 상황 | Fork | Clone |
|------|------|-------|
| 오픈소스에 기여하고 싶다 | ✅ 사용 | ❌ 단독 사용 불가 |
| 개인 프로젝트를 시작한다 | ❌ 불필요 | ✅ 로컬에만 필요 |
| 회사 프로젝트, 팀 저장소 접근 | ❌ (소유자라면 불필요) | ✅ 권한이 있다면 clone만 해도 충분 |
| 다른 사람 저장소를 고치고 PR 보내고 싶다 | ✅ 필수 | ❌ clone만으로는 PR 불가 |
| 저장소를 백업/분석하려고 한다 | ❌ (내 계정 필요 없음) | ✅ clone만 해도 충분 |

---

## 🔗 6. 참고 명령어

```bash
# Fork 후 로컬 clone
git clone git@github.com:myusername/forked-repo.git

# 원본 저장소 remote 등록 (업스트림 추가)
git remote add upstream https://github.com/original-owner/repo.git

# 원본 저장소 최신 내용 받아오기
git fetch upstream
git merge upstream/main
```

---

## 📎 참고

- GitHub Fork 공식 문서: https://docs.github.com/en/get-started/quickstart/fork-a-repo
- Git 공식 clone 문서: https://git-scm.com/docs/git-clone