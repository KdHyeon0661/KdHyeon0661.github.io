---
layout: post
title: Git - cherry-pick vs rebase
date: 2025-01-30 20:20:23 +0900
category: Git
---
# Git cherry-pick vs rebase  
> cherry-pick 먼저 설명하고 차이점 비교

---

## 🍒 1. git cherry-pick 이란?

**`git cherry-pick`은 특정 커밋만 선택해서 다른 브랜치에 적용하는 기능**입니다.

- 여러 커밋 중 **딱 하나 또는 몇 개**만 옮기고 싶을 때 사용
- 전체 브랜치를 병합하지 않고 **특정 기능/수정만 반영** 가능

---

## ✅ 2. 기본 사용법

### ① 커밋 ID 확인

```bash
git log --oneline
```

예시 출력:

```
a3f5d19 로그인 기능 추가
e4c1a7b 배너 디자인 수정
9b2e0f2 footer 수정
```

### ② cherry-pick 실행

```bash
# 예: a3f5d19 커밋만 현재 브랜치에 적용
git cherry-pick a3f5d19
```

> 현재 브랜치로 해당 커밋의 변경 내용을 가져옵니다.

---

## 🧱 3. 여러 커밋 cherry-pick

```bash
# 커밋 여러 개 선택
git cherry-pick a3f5d19 e4c1a7b
```

또는 범위 지정:

```bash
git cherry-pick a3f5d19^..e4c1a7b
```

> `^`는 a3f5d19의 이전 커밋까지 포함

---

## 🔀 4. cherry-pick 충돌 발생 시

```bash
Auto-merging app.js
CONFLICT (content): Merge conflict in app.js
```

### 해결 절차:

1. 충돌난 파일 열어 수정
2. `git add 수정한파일`
3. `git cherry-pick --continue`

중단하려면:

```bash
git cherry-pick --abort
```

---

## 🧹 5. cherry-pick 응용 예시

### ✅ 예: `hotfix`만 `main`에 적용하고 싶을 때

```bash
# hotfix 브랜치의 특정 커밋만 main에 적용
git checkout main
git cherry-pick abc1234  # hotfix 커밋 ID
```

---

## 🧠 6. cherry-pick vs rebase 차이

| 항목 | `cherry-pick` | `rebase` |
|------|----------------|-----------|
| 목적 | **특정 커밋만 복사** | 브랜치 전체를 **재배치** |
| 대상 커밋 | 하나 또는 일부 커밋 | 여러 커밋 또는 전체 커밋 |
| 작업 위치 | **현재 브랜치에 커밋을 덧붙임** | **브랜치 기반 변경 (위치 이동)** |
| 히스토리 | 복사된 커밋만 추가됨 | 히스토리 재작성 |
| 커밋 해시 | 변경됨 (복사이기 때문에) | 변경됨 (기존 커밋 새로 생성됨) |
| 사용 예 | - 버그 픽스만 main에 적용  
          - 실험 커밋 중 일부만 적용 | - 기능 브랜치를 정리  
          - merge 없이 병합 |
| 협업 안전성 | 안전 (정확한 커밋만 사용) | 위험할 수 있음 (강제 push 필요) |

---

## 🎯 7. 선택 기준

| 상황 | 추천 |
|------|------|
| 오픈소스에서 특정 커밋만 가져오고 싶다 | `cherry-pick` |
| 기능 브랜치 정리해서 깔끔히 병합하고 싶다 | `rebase` |
| 커밋 히스토리를 깔끔히 선형으로 만들고 싶다 | `rebase` |
| 브랜치 병합 없이 일부 기능만 반영하고 싶다 | `cherry-pick` |

---

## 📎 명령어 요약

```bash
# 특정 커밋만 현재 브랜치에 적용
git cherry-pick <commit-id>

# 여러 개 커밋 선택
git cherry-pick <id1> <id2>

# 범위 지정
git cherry-pick <start>^..<end>

# 충돌 해결 후 계속
git cherry-pick --continue

# 중단
git cherry-pick --abort
```

---

## 🔗 참고 링크

- Git cherry-pick 공식 문서: https://git-scm.com/docs/git-cherry-pick
- Git rebase 문서: https://git-scm.com/docs/git-rebase
