---
layout: post
title: Git - pull --rebase & squash merge
date: 2025-02-06 19:20:23 +0900
category: Git
---
# `git pull --rebase`와 GitHub `Squash and merge` 완벽 가이드

## 두 기능의 공통점: 깔끔한 히스토리

`git pull --rebase`와 GitHub의 `Squash and merge`는 모두 Git 히스토리를 정리하고 단순화하는 데 사용됩니다. 각각 다른 상황에서 활용되지만, 목표는 동일합니다: 복잡한 브랜치 구조를 정리하여 프로젝트의 변경 이력을 이해하기 쉽게 만드는 것입니다.

---

# Part 1: `git pull --rebase` 상세 가이드

## 기본 개념

`git pull --rebase`는 원격 저장소의 최신 변경사항을 가져온 후, 로컬에서 작성한 커밋을 그 위에 "재배치"하는 방식으로 동작합니다. 이는 일반적인 `git pull`(merge 방식)과 다른 접근법을 제공합니다.

```bash
git pull --rebase origin main
```

**작동 원리**:
1. 원격 저장소(`origin/main`)에서 최신 커밋을 가져옵니다(fetch).
2. 로컬의 커밋들을 일시적으로 제거합니다.
3. 가져온 최신 커밋 위에 로컬 커밋들을 순서대로 다시 적용합니다.
4. 충돌이 없으면 선형적인 히스토리가 생성됩니다.

## 일반 pull vs pull --rebase 비교

| 특징 | `git pull` (Merge 방식) | `git pull --rebase` |
|------|------------------------|---------------------|
| **히스토리 형태** | 분기와 병합이 표시됨 | 선형적이고 깔끔함 |
| **병합 커밋** | 생성됨 | 생성되지 않음 |
| **충돌 처리** | 한 번에 처리 | 각 커밋별로 처리 가능 |
| **협업 영향** | 안전함 | 주의 필요(커밋 해시 변경) |

## 실전 사용법

### 기본 설정 (매번 옵션 입력 방지)
```bash
# pull 기본 동작을 rebase로 설정
git config --global pull.rebase true

# rebase 시 자동으로 변경사항 임시 저장
git config --global rebase.autoStash true

# 반복되는 충돌 해결 자동화
git config --global rerere.enabled true
```

### 충돌 해결 과정
rebase 도중 충돌이 발생하면 다음과 같이 처리합니다:
```bash
# 충돌 발생 시
git pull --rebase

# 충돌이 발생한 파일을 편집하여 해결
# 파일 수정 후...
git add 수정한파일.txt

# rebase 계속 진행
git rebase --continue

# 현재 커밋 건너뛰기
# git rebase --skip

# rebase 중단 (원래 상태로 복구)
# git rebase --abort
```

### 빠른 해결 옵션
```bash
# 현재 브랜치의 버전 선택
git checkout --ours 파일명

# 원격 브랜치의 버전 선택
git checkout --theirs 파일명

# 시각적 병합 도구 사용
git mergetool
```

## 주의사항과 모범 사례

### 언제 사용해야 할까요?
- **개인 작업 브랜치**에서 메인 브랜치와 동기화할 때
- 깔끔한 선형 히스토리를 유지하고 싶을 때
- 불필요한 병합 커밋을 피하고 싶을 때

### 피해야 할 상황
- **공유 브랜치**에서는 사용하지 마세요 (다른 사람의 작업에 영향을 줄 수 있음)
- 이미 원격에 푸시된 커밋을 rebase하면 다른 협업자에게 문제를 일으킬 수 있습니다

### 안전한 사용을 위한 팁
```bash
# 강제 푸시 시 다른 사람의 작업을 보호
git push --force-with-lease

# 실수했을 때 복구 방법
git reflog  # 작업 기록 확인
git reset --hard ORIG_HEAD  # rebase 이전 상태로 복구
```

---

# Part 2: GitHub `Squash and merge` 심화 가이드

## 기본 개념

GitHub의 `Squash and merge` 기능은 Pull Request의 모든 커밋을 하나의 커밋으로 압축하여 메인 브랜치에 병합합니다. 이는 기능 개발 과정에서 생성된 여러 개의 작은 커밋을 논리적인 단위로 통합하는 데 유용합니다.

## 사용 방법

1. GitHub에서 Pull Request 페이지로 이동합니다.
2. **Squash and merge** 버튼을 클릭합니다.
3. GitHub가 모든 커밋 메시지를 하나로 합친 편집창을 보여줍니다.
4. 최종 커밋 메시지를 적절히 정리합니다.
5. **Confirm squash and merge**를 클릭하여 병합을 완료합니다.

### 커밋 메시지 작성 예시

**병합 전 개별 커밋**:
```
feat: 로그인 폼 UI 구현
fix: 버튼 색상 오류 수정
refactor: 컴포넌트 구조 개선
```

**Squash 후 최종 메시지**:
```
feat(auth): 로그인 화면 구현 완료

- 기본 로그인 폼 UI 구현
- 버튼 색상 오류 수정
- 컴포넌트 구조 리팩토링

Co-authored-by: 동료이름 <email@example.com>
```

### 중요한 메타데이터 유지

Squash and merge 시에도 중요한 정보는 유지할 수 있습니다:
- **Co-authored-by**: 여러 사람이 함께 작업한 경우
- **Signed-off-by**: DCO(Developer Certificate of Origin) 정책이 있는 경우
- **Issue 참조**: `Closes #123` 같은 이슈 링크

## 장단점 분석

### 장점
- **히스토리 단순화**: 기능 단위로 명확한 커밋 히스토리 유지
- **가독성 향상**: 메인 브랜치의 히스토리가 깔끔해짐
- **리뷰 단위 명확화**: 하나의 PR이 하나의 논리적 변경으로 표시

### 단점
- **세부 이력 손실**: 개별 커밋의 맥락이 사라짐
- **디버깅 어려움**: `git bisect` 등으로 특정 변경을 추적하기 어려울 수 있음
- **기여도 추적**: 개별 커밋별 기여도 파악이 어려움

## 실무 적용 가이드

### 적합한 상황
- 기능 개발 단위가 명확한 프로젝트
- 오픈소스 프로젝트에서 외부 기여자 PR 처리
- 작은 규모의 팀이나 프로젝트

### 부적합한 상황
- 디버깅과 회귀 테스트가 빈번한 프로젝트
- 규제 준수를 위해 모든 변경 사항을 추적해야 하는 경우
- 대규모 엔터프라이즈 프로젝트

---

# Part 3: 두 기능의 통합 활용 전략

## 로컬 개발 워크플로우 예시

개발자가 로컬에서 작업할 때의 이상적인 워크플로우는 다음과 같을 수 있습니다:

```bash
# 1. 메인 브랜치에서 시작
git checkout main
git pull --rebase origin main

# 2. 기능 브랜치 생성
git checkout -b feature/new-login

# 3. 여러 커밋으로 작업
# ... 작업 중 ...
git commit -m "feat: 로그인 폼 기본 구조"
git commit -m "fix: 입력 유효성 검사"
git commit -m "style: 반응형 디자인 개선"

# 4. PR 생성 전 메인 브랜치와 동기화
git pull --rebase origin main

# 5. GitHub에 PR 생성
git push origin feature/new-login
```

## GitHub에서의 처리

PR 리뷰가 완료되면:
1. 팀원들이 코드 리뷰를 완료합니다.
2. 모든 CI 테스트가 통과합니다.
3. **Squash and merge**를 선택하여 기능을 통합합니다.
4. 깔끔한 단일 커밋으로 메인 브랜치에 병합됩니다.

## 팀 정책 수립

효과적인 협업을 위해 팀 단위의 정책을 수립하는 것이 중요합니다:

### 권장 정책
1. **개인 브랜치에서는 `pull --rebase` 사용**: 로컬 히스토리 정리
2. **공유 브랜치에서는 일반 `pull` 사용**: 안정성 유지
3. **PR 병합 전략 통일**: 팀 합의하에 Squash or Merge 결정
4. **커밋 메시지 컨벤션**: Conventional Commits 등 표준 채택

### GitHub 설정
저장소 설정에서 다음을 구성할 수 있습니다:
- 허용할 병합 방법 선택
- 기본 병합 방법 설정
- 브랜치 보호 규칙 설정

---

# 마무리

`git pull --rebase`와 GitHub의 `Squash and merge`는 현대적인 Git 워크플로우의 핵심 도구입니다. 이 둘을 적절히 활용하면 프로젝트의 히스토리를 깔끔하게 유지하면서도 효율적인 협업을 이어갈 수 있습니다.

## 핵심 요약

1. **`git pull --rebase`**는 로컬 작업 흐름을 개선하는 도구입니다. 개인 브랜치에서 사용하면 선형적이고 이해하기 쉬운 히스토리를 만들 수 있습니다.

2. **GitHub `Squash and merge`**는 팀 협업을 개선하는 도구입니다. PR 단위로 논리적인 변경을 기록하여 메인 브랜치의 가독성을 높입니다.

3. **적절한 균형**이 중요합니다. 히스토리의 깔끔함과 디버깅을 위한 세부 정보 사이에서 팀의 요구에 맞는 균형점을 찾으세요.

4. **팀 합의**가 필수입니다. 어떤 상황에서 어떤 도구를 사용할지 팀원들과 논의하고 문서화하세요.

이 도구들을 마스터하는 것은 단순한 기술 습득 이상의 의미가 있습니다. 이는 더 나은 협업 문화, 더 명확한 코드 히스토리, 그리고 궁극적으로 더 높은 코드 품질로 이어집니다. 처음에는 조금 복잡하게 느껴질 수 있지만, 일단 익숙해지면 개발 워크플로우의 효율성을 크게 향상시킬 수 있는 강력한 도구가 됩니다.