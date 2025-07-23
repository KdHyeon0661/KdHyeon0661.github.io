---
layout: post
title: Git - GitHub Flow vs Git Flow
date: 2025-02-11 20:20:23 +0900
category: Git
---
# 🌊 GitHub Flow vs Git Flow 완전 정리

---

## 📌 개요

| 구분 | Git Flow | GitHub Flow |
|------|----------|-------------|
| 스타일 | 릴리스 주기 기반 | 지속 배포(CD) 기반 |
| 복잡도 | 높음 | 낮음 |
| 주요 브랜치 | `main`, `develop`, `feature`, `release`, `hotfix` | `main`, `feature` |
| 릴리스 전략 | 버전 단위 배포 | 배포 자동화, 빠른 병합 |
| 사용 예 | 대규모 프로젝트, 앱 | 웹 서비스, 스타트업, DevOps |

---

# 1️⃣ Git Flow

---

## 🧠 개념

**Vincent Driessen**가 제안한 브랜치 전략으로,  
**릴리스 단위로 계획된 개발과 병합을 명확히 나눕니다.**

---

## 🔧 주요 브랜치 구성

| 브랜치 | 용도 |
|--------|------|
| `main` | 실제 배포 중인 안정 버전 |
| `develop` | 다음 릴리스를 위한 통합 브랜치 |
| `feature/xxx` | 기능 개발용 |
| `release/xxx` | 배포 준비 및 버그 픽스 |
| `hotfix/xxx` | 배포 후 긴급 수정 |

---

## 🔁 흐름

```
main ← release ← develop ← feature
main ← hotfix
```

1. 기능 개발:  
   `git checkout -b feature/login develop`

2. 기능 완료 → develop 병합

3. 배포 준비:  
   `git checkout -b release/1.0 develop`  
   테스트 및 문서 수정

4. 배포:  
   `release → main → tag v1.0`  
   그리고 `release → develop`

5. 긴급 수정 발생 시:  
   `git checkout -b hotfix/1.0.1 main`

---

## ✅ 장점

- 역할이 명확한 브랜치 구분
- 대규모 릴리스, 다수의 기능 동시 개발에 적합
- 릴리스 관리가 용이

---

## ❗ 단점

- 브랜치가 많고 복잡함
- 단일 배포 트리거로는 비효율적
- 실시간 배포 환경에 부적합

---

# 2️⃣ GitHub Flow

---

## 🧠 개념

**GitHub에서 공식적으로 권장**하는 간단하고 직관적인 브랜치 전략.  
**지속 배포(CD)** 환경에 매우 적합하며, 단순히 `main`과 `feature`만 사용합니다.

---

## 🔧 주요 브랜치 구성

| 브랜치 | 용도 |
|--------|------|
| `main` | 항상 배포 가능한 코드 |
| `feature/xxx` | 기능 개발, PR 후 `main`으로 병합 |

---

## 🔁 흐름

```text
main ← feature/login
```

1. `main`에서 브랜치 생성:  
   `git checkout -b feature/login main`

2. 기능 개발 → 커밋 → 푸시

3. GitHub에 PR 생성 + 코드 리뷰

4. CI 테스트 통과 후 → `main`에 병합  
   (보통 `Squash and merge` or `Rebase and merge`)

5. 자동 배포

---

## ✅ 장점

- 브랜치 간소화
- 자동화(CI/CD)와 궁합 좋음
- 변경 사항 추적과 리뷰 중심 문화 강화
- 빠른 피드백, 빠른 배포

---

## ❗ 단점

- 릴리스 준비 기간 필요 시 불편
- 병렬 기능 관리가 많을 경우 혼잡 가능
- 버전 태그 관리 수동 필요

---

# 🎯 어떤 상황에서 어떤 플로우를 쓸까?

| 상황 | 추천 전략 |
|------|-----------|
| 웹 앱, 빠른 배포, DevOps 환경 | GitHub Flow ✅ |
| 사내 시스템, 버전 관리 중심 | Git Flow ✅ |
| 오픈소스 협업 (단순화된 전략) | GitHub Flow |
| 릴리스와 패치가 동시에 필요 | Git Flow |

---

## 📊 요약 비교표

| 항목 | Git Flow | GitHub Flow |
|------|----------|-------------|
| 복잡도 | 높음 | 낮음 |
| 주요 브랜치 | 5개 이상 | 2개 |
| 배포 방식 | 수동 릴리스 | 지속 배포 |
| 병합 방식 | 릴리스 준비 → 배포 | PR 병합 후 자동 배포 |
| 릴리스 전 QA 단계 | 명확히 분리 | 별도 브랜치 없음 |
| CI/CD 적합도 | 낮음 | 매우 높음 |
| 히스토리 관리 | 분기 기반 | 선형 중심 |

---

## 💡 Git Flow를 GitHub Flow처럼 간소화하는 팁

- `develop` 브랜치를 없애고 `main`을 주 통합 브랜치로 사용
- `release`, `hotfix`는 태그 + PR 방식으로 대체
- `feature` 브랜치 → PR → main 병합 → 자동 배포로 통일

---

## 🔗 참고 링크

- [Git Flow 원본 글 (nvie)](https://nvie.com/posts/a-successful-git-branching-model/)
- [GitHub Flow 공식 가이드](https://docs.github.com/en/get-started/quickstart/github-flow)
- [Git Flow vs GitHub Flow 비교 문서](https://www.atlassian.com/git/tutorials/comparing-workflows)