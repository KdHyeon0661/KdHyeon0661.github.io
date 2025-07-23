---
layout: post
title: Git - rebase and merge & 자동 병합 정책 설정
date: 2025-02-06 19:20:23 +0900
category: Git
---
# 1️⃣ GitHub의 `Rebase and merge` 완전 정리

---

## 🧠 개념

GitHub에서 PR(Pull Request)을 병합할 때 사용할 수 있는 병합 방식 중 하나입니다:

```
🔘 Create a merge commit  
🔘 Squash and merge  
✅ Rebase and merge
```

---

## 🔁 Rebase and merge란?

- PR 브랜치의 커밋들을 **base 브랜치(main)의 끝으로 순서대로 붙이는 방식**
- 병합 커밋(merge commit)이 **생기지 않음**
- 히스토리가 **선형(linear)** 하게 유지됨

---

## 🧱 예시

기준 브랜치(main):

```
A---B---C  (main)
```

PR 브랜치(feature):

```
       D---E  (feature)
```

### ✅ rebase and merge 결과:

```
A---B---C---D'---E'  (main)
```

- D, E가 `main` 위로 새롭게 이어짐 (`D'`, `E'`는 새로운 커밋으로 재작성됨)

---

## 📦 사용법

1. PR 화면으로 이동
2. 아래쪽 병합 방식 중 **`Rebase and merge`** 선택
3. `Confirm rebase and merge` 클릭

---

## 📊 장단점

| 항목 | 장점 | 단점 |
|------|------|------|
| 히스토리 | 깔끔하고 선형 | 커밋 해시 변경됨 |
| 협업 | 추적과 디버깅 쉬움 | 커밋 서명이 사라질 수 있음 |
| 병합 커밋 없음 | ✔️ | 커밋 간 경계가 모호해질 수 있음 |

---

## 🚫 주의

- PR 브랜치가 최신 main과 충돌 시 자동 병합 불가
- 충돌 해결은 웹 또는 로컬에서 수동 처리 필요
- 로컬에서 rebase 후 PR 업데이트 필요할 수도 있음

---

# 2️⃣ GitHub 자동 병합 정책 설정 (Branch Protection Rules)

---

## 🧠 개요

GitHub에서는 브랜치 보호 규칙(`Branch protection rules`)을 통해 **PR 병합 방식**, **CI 성공 여부**, **리뷰 요구사항** 등을 설정할 수 있습니다.

> `main` 브랜치에 임의로 push 하지 못하게 하고, 반드시 PR 리뷰를 거치도록 강제할 수 있음

---

## 🛠️ 설정 위치

1. GitHub 저장소 → `Settings` → `Branches`
2. `Branch protection rules` 섹션
3. `Add rule` 버튼 클릭
4. 보호할 브랜치 이름 (예: `main`) 지정

---

## 🔐 주요 설정 항목

| 항목 | 설명 |
|------|------|
| ✅ Require pull request reviews | PR 병합 전에 리뷰 필수 |
| ✅ Require status checks to pass | CI 테스트 성공 후 병합 허용 |
| ✅ Require linear history | **merge commit 금지** → `rebase` or `squash`만 허용됨 |
| ✅ Require signed commits | GPG 서명된 커밋만 병합 허용 |
| ✅ Restrict who can push | 특정 사람만 푸시 가능 |
| ✅ Dismiss stale reviews | 새 커밋 푸시 시 기존 리뷰 무효화 |

---

## 💡 예: Rebase and Merge만 허용하고 싶을 때

1. `Require linear history` 체크 ✅  
   → Merge commit 금지, Rebase & Squash만 허용

2. `Allow merge commits` 옵션 해제  
3. `Allow squash merges`, `Allow rebase merges`만 활성화

이 설정을 하면 PR을 머지할 때 "Rebase and merge" 또는 "Squash and merge"만 선택 가능하고, 일반 Merge는 비활성화됩니다.

---

## 🧪 실무 적용 팁

| 상황 | 추천 설정 |
|------|-----------|
| 오픈소스 | Require PR + Require CI + Require reviews |
| 기업 협업 | PR 필수 + 병합은 squash or rebase |
| 자동화된 배포 | CI 성공 후만 병합 허용 |
| Merge 커밋 방지 | Require linear history 활성화 |

---

## 🔗 참고 링크

- [GitHub Rebase and Merge 공식 문서](https://docs.github.com/en/pull-requests/collaborating-with-pull-requests/merging-a-pull-request#rebase-and-merge)
- [GitHub Branch Protection 설정](https://docs.github.com/en/repositories/configuring-branches-and-merges-in-your-repository/managing-a-branch-protection-rule)

---

## ✅ 마무리 요약

| 항목 | 설명 |
|------|------|
| `Rebase and merge` | PR 커밋을 main 끝으로 이어붙임 (히스토리 깔끔) |
| `Branch Protection Rules` | 병합 방식 제한, 리뷰/CI 강제, push 통제 가능 |
| `Require linear history` | 병합 커밋 방지, rebase/squash만 허용 |