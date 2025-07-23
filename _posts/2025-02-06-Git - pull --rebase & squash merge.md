---
layout: post
title: Git - pull --rebase & squash merge
date: 2025-02-06 19:20:23 +0900
category: Git
---
# 1️⃣ `git pull --rebase`: 깔끔한 로컬 병합 방식

---

## 🧠 개념

`git pull --rebase`는 원격 브랜치의 최신 커밋을 병합할 때 **`merge` 대신 `rebase`** 방식으로 처리합니다.

```bash
git pull --rebase origin main
```

이 명령은 다음과 같은 흐름입니다:

1. 원격 브랜치(main)의 최신 커밋 가져오기
2. 내 로컬 커밋들을 **일시적으로 꺼냄**
3. 원격 커밋 위에 **내 커밋을 깔끔히 이어붙임**
4. 충돌이 없다면 커밋 히스토리가 **선형(linear)** 으로 정리됨

---

## 🔍 비교: 일반 pull vs pull --rebase

| 항목 | `git pull` | `git pull --rebase` |
|------|------------|----------------------|
| 병합 방식 | `merge` 사용 | `rebase` 사용 |
| 커밋 히스토리 | 병합 커밋(Merge commit)이 생김 | 선형으로 정리됨 |
| 협업 시 | 히스토리 복잡해질 수 있음 | 깔끔하지만 충돌 처리 필요 |
| 예시 | `A—B—C` ← `D—E` → `merge` → `F` | `A—B—C—D'—E'` (D, E는 재배치됨)

---

## ✅ 사용법

```bash
# 일반적인 상황
git checkout feature/login
git pull --rebase origin main
```

### ⚠️ 충돌 시:

1. 파일 수정
2. `git add 파일명`
3. `git rebase --continue`

중단하려면: `git rebase --abort`

---

## 📎 자동 설정 방법

매번 `--rebase` 쓰기 귀찮다면 기본 설정 가능:

```bash
git config --global pull.rebase true
```

---

# 2️⃣ GitHub: Squash and merge

---

## 🧠 개념

Pull Request를 병합할 때 GitHub는 3가지 병합 방식을 지원합니다:

1. ✅ **Create a merge commit**
2. ✅ **Squash and merge** ← 오늘의 주제!
3. ✅ **Rebase and merge**

---

## 🍡 Squash and merge란?

PR에 있는 **여러 커밋을 하나의 커밋으로 "압축(squash)"하여** 병합하는 방식입니다.

> 커밋 로그가 깨끗해지고, 기능 단위로 관리하기 좋아짐

---

## 📦 예시

PR에 다음 커밋이 있다고 가정:

```
feat: 로그인 폼 생성
fix: 버튼 스타일 수정
refactor: 네이밍 변경
```

`Squash and merge`를 선택하면 GitHub에서 **1개의 커밋 메시지 입력 창**이 뜨고,  
최종적으로 다음처럼 병합됨:

```
feat: 로그인 화면 기능 추가
```

---

## ✅ 사용법

1. PR을 엽니다.
2. 아래 병합 옵션 중 `Squash and merge` 선택
3. 메시지를 적절히 수정
4. `Confirm squash and merge` 클릭

---

## 🧹 커밋 메시지 정리 팁

- PR의 커밋이 너무 많거나 불필요한 메시지가 많은 경우에 유용
- 기능 단위로 하나의 커밋을 남기면 히스토리 추적이 쉬움
- 예:

```text
feat: 사용자 로그인 구현
- 로그인 폼 생성
- 스타일 개선
- input 이벤트 처리
```

---

## ✅ 장점 vs 단점

| 항목 | 장점 | 단점 |
|------|------|------|
| 히스토리 | 깔끔하고 읽기 쉬움 | 커밋 단위 디버깅 어려움 |
| 기능 단위 병합 | 가능 | 중간 변경사항 누락될 수 있음 |
| 협업 관리 | 좋아짐 | 커밋 로그 세부 정보 사라짐 |

---

## 💡 실무에서의 사용 팁

| 상황 | 병합 방식 추천 |
|------|----------------|
| 개인 프로젝트 | Squash and merge ✔️ |
| 오픈소스 PR 관리 | Squash and merge ✔️ |
| 상세한 커밋이 필요한 협업 | 일반 Merge or Rebase 병합 ✔️ |
| 리뷰 단위 커밋 정리하고 싶을 때 | Squash and merge ✔️ |

---

## 🔗 참고 링크

- GitHub Docs (Squash and Merge):  
  https://docs.github.com/en/pull-requests/collaborating-with-pull-requests/merging-a-pull-request#squash-and-merge

- Git Docs (pull.rebase):  
  https://git-scm.com/docs/git-pull#Documentation/git-pull.txt---rebase
  
---

## ✅ 마무리 요약

| 주제 | 요약 |
|------|------|
| `git pull --rebase` | 로컬에서 최신 커밋을 깔끔히 이어붙이기 |
| GitHub squash merge | PR을 단일 커밋으로 병합, 히스토리 간결 |