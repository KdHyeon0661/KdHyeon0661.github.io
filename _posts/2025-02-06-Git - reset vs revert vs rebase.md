---
layout: post
title: Git - reset vs revert vs rebase
date: 2025-02-06 19:20:23 +0900
category: Git
---
# Git: Reset, Revert, Rebase 비교 가이드

## Git의 세 가지 상태 이해

Git 작업을 효과적으로 관리하려면 다음 세 가지 상태를 이해하는 것이 중요합니다:

1. **HEAD**: 현재 체크아웃된 커밋을 가리키는 포인터 (현재 작업 위치)
2. **Index (Staging Area)**: 다음 커밋에 포함될 변경사항들이 모이는 공간
3. **Working Tree**: 실제 파일들이 존재하는 작업 디렉토리

이 세 도구(reset, revert, rebase)는 각각 이 상태들을 다르게 조작합니다.

---

## Git Reset: 히스토리 포인터 재설정

### 기본 개념
`git reset`은 브랜치 포인터(HEAD)를 과거의 특정 커밋으로 이동시키고, 선택한 옵션에 따라 Index와 Working Tree의 상태를 조정합니다.

### 세 가지 주요 모드

#### 1. `--soft` (가장 안전)
```bash
git reset --soft HEAD~1
```
- **동작**: HEAD만 이전 커밋으로 이동
- **영향**: Index와 Working Tree는 변경되지 않음
- **사용처**: 최근 커밋 메시지 수정, 커밋 분할을 위한 준비

#### 2. `--mixed` (기본값)
```bash
git reset --mixed HEAD~1
```
- **동작**: HEAD와 Index를 이전 커밋 상태로 재설정
- **영향**: 변경사항은 Working Tree에 남아 있음 (unstaged 상태)
- **사용처**: 커밋을 재구성하거나 부분적으로 다시 커밋할 때

#### 3. `--hard` (가장 위험)
```bash
git reset --hard HEAD~1
```
- **동작**: HEAD, Index, Working Tree 모두를 이전 커밋 상태로 재설정
- **영향**: 모든 변경사항이 완전히 삭제됨
- **사용처**: 실험적인 변경사항 완전히 폐기

### 시각적 이해
```
초기 상태:
A---B---C---D (main, HEAD)

--soft 사용 후:
A---B---C (main, HEAD)
         \
          D (변경사항은 Index에 남음)

--mixed 사용 후:
A---B---C (main, HEAD)
         \
          D (변경사항은 Working Tree에만 남음)

--hard 사용 후:
A---B---C (main, HEAD)  # D의 모든 변경사항 완전 삭제
```

### 실용적 예제

**커밋 메시지 수정하기:**
```bash
# 최근 커밋 취소 (변경사항 유지)
git reset --soft HEAD~1

# 새 메시지로 커밋
git commit -m "수정된 커밋 메시지"
```

**커밋을 여러 개로 분할하기:**
```bash
# 최근 커밋을 unstaged 상태로 분해
git reset --mixed HEAD~1

# 변경사항을 선택적으로 스테이징
git add -p

# 각 부분을 별도로 커밋
git commit -m "첫 번째 부분"
git commit -m "두 번째 부분"
```

### 위험성과 주의사항
- **공유 브랜치 금지**: 이미 푸시된 브랜치에서 reset을 사용하면 협업자들의 히스토리가 손상됩니다.
- **복구 가능성**: `--hard`는 Working Tree 변경까지 삭제하므로 복구가 어려울 수 있습니다.
- **백업 습관**: 중요한 작업 전에는 항상 백업 브랜치를 생성하세요.

### 실수 복구 방법
```bash
# 최근 작업 기록 확인
git reflog

# 직전 상태로 복구
git reset --hard ORIG_HEAD

# 특정 시점으로 복구
git reset --hard HEAD@{3}

# 안전한 복구 (새 브랜치 생성)
git switch -c rescued-branch HEAD@{2}
```

---

## Git Revert: 안전한 변경 취소

### 기본 개념
`git revert`는 특정 커밋의 변경사항을 반대로 적용하는 새로운 커밋을 생성합니다. 기존 히스토리는 그대로 유지되므로 협업 환경에서 가장 안전한 방법입니다.

### 기본 사용법
```bash
# 단일 커밋 취소
git revert abc1234

# 여러 커밋 범위 취소
git revert HEAD~3..HEAD
```

### 병합 커밋 되돌리기
병합 커밋을 되돌릴 때는 어느 부모 브랜치를 기준으로 할지 지정해야 합니다:
```bash
# 첫 번째 부모(main 브랜치) 기준으로 revert
git revert -m 1 merge-commit-id
```

**중요**: 복잡한 병합 상황에서는 테스트 브랜치에서 먼저 시도해보는 것이 좋습니다.

### 충돌 처리
```bash
# revert 실행 중 충돌 발생 시
# 1. 충돌 파일 수정
# 2. 수정사항 스테이징
git add filename
# 3. revert 계속 진행
git revert --continue

# revert 취소
git revert --abort
```

### Revert 되돌리기
revert 커밋 자체를 되돌려 원래 변경사항을 복원할 수 있습니다:
```bash
git revert revert-commit-id
```

---

## Git Rebase: 히스토리 재배치

### 기본 개념
`git rebase`는 현재 브랜치의 커밋들을 다른 브랜치의 최신 커밋 위로 재배치합니다. 이를 통해 병합 커밋 없이 선형적인 히스토리를 만들 수 있습니다.

### 기본 사용법
```bash
# 현재 브랜치를 main 브랜치 기준으로 rebase
git checkout feature-branch
git rebase main
```

### 인터랙티브 Rebase
히스토리를 정리하는 강력한 방법:
```bash
# 최근 5개 커밋 정리
git rebase -i HEAD~5
```

편집기에서 사용 가능한 명령어:
- `pick`: 커밋 유지
- `reword`: 커밋 메시지 수정
- `edit`: 커밋 내용 수정
- `squash`: 이전 커밋과 합치기
- `fixup`: 이전 커밋과 합치기 (메시지 무시)
- `drop`: 커밋 삭제

### 자동 스쿼시
```bash
# fixup! 또는 squash! 접두사가 있는 커밋 자동 정렬
git rebase -i --autosquash HEAD~10
```

### 고급 기능

#### 특정 커밋 범위 이동
```bash
# A 커밋 이후부터 B 커밋까지를 new-base 위로 이동
git rebase --onto new-base A B
```

#### 병합 구조 보존
```bash
# 병합 커밋 구조를 유지하면서 rebase
git rebase --rebase-merges main
```

### Pull 시 Rebase 설정
```bash
# pull 시 항상 rebase 사용
git config --global pull.rebase true

# 또는 수동 실행
git pull --rebase
```

### 충돌 처리
```bash
# rebase 중 충돌 발생 시
# 1. 충돌 파일 수정
# 2. 수정사항 스테이징
git add filename
# 3. rebase 계속 진행
git rebase --continue

# rebase 취소
git rebase --abort

# 현재 커밋 건너뛰기
git rebase --skip
```

---

## 세 도구 비교 분석

### 목적과 특징

| 항목 | Reset | Revert | Rebase |
|------|-------|--------|--------|
| **주요 목적** | 히스토리 포인터 재설정 | 변경사항 반전 커밋 생성 | 커밋 재배치 및 히스토리 정리 |
| **히스토리 변경** | 예 (포인터 이동) | 아니오 (새 커밋 추가) | 예 (커밋 재작성) |
| **협업 안전성** | 낮음 (로컬 전용) | 높음 (공유 브랜치 적합) | 중간 (공유 브랜치 주의) |
| **변경 영향** | 옵션에 따라 다양 | Working Tree 영향 없음 | 파일 재적용 과정 필요 |
| **사용 예시** | 로컬 커밋 수정 | 운영 중 문제 커밋 취소 | PR 전 히스토리 정리 |

### 상황별 추천 도구

| 상황 | 추천 도구 | 이유 |
|------|-----------|------|
| **로컬 커밋 메시지 수정** | `reset --soft` | 변경사항 유지하면서 메시지만 수정 |
| **커밋을 여러 부분으로 분할** | `reset --mixed` | 변경사항을 선택적으로 재커밋 가능 |
| **실험적 변경 완전 폐기** | `reset --hard` | 모든 변경사항 완전 삭제 |
| **공유 브랜치의 문제 커밋 취소** | `revert` | 히스토리 보존하며 안전하게 취소 |
| **개인 브랜치 히스토리 정리** | `rebase -i` | 커밋 정리 및 선형 히스토리 생성 |
| **기능 브랜치 최신 상태 유지** | `rebase` | main 브랜치 변경사항 반영 |
| **복잡한 병합 커밋 취소** | `revert -m 1` | 특정 부모 기준으로 안전하게 취소 |

---

## 충돌 해결 모범 사례

### 충돌 마커 이해
충돌이 발생한 파일에는 다음과 같은 마커가 포함됩니다:
```
<<<<<<< HEAD
현재 브랜치의 내용
=======
다른 브랜치의 내용
>>>>>>> branch-name
```

### 충돌 해결 절차
1. 충돌 파일 열기 및 내용 검토
2. 적절한 내용으로 통합 (마커 제거)
3. 수정사항 스테이징: `git add filename`
4. 작업 계속: 상황에 따라 `--continue`, `--abort`, `--skip`

### 빠른 해결 옵션
```bash
# 현재 브랜치 버전으로 덮어쓰기
git checkout --ours filename

# 다른 브랜치 버전으로 덮어쓰기
git checkout --theirs filename

# 병합 도구 사용
git mergetool
```

### 반복 충돌 방지
```bash
# rerere 기능 활성화
git config --global rerere.enabled true
```

---

## 협업 환경 고려사항

### 보호 브랜치 규칙
많은 조직에서는 보호 브랜치 규칙을 설정합니다:
- **리뷰 요구사항**: PR 승인 필요
- **상태 체크**: CI/CD 테스트 통과 필수
- **선형 히스토리**: 병합 커밋 금지
- **서명 커밋**: GPG 서명 필수
- **강제 푸시 제한**: 히스토리 재작성 방지

이러한 규칙은 `reset`과 `rebase` 사용에 제약을 줄 수 있습니다.

### 서명 커밋과의 상호작용
```bash
# GPG 서명 설정
git config --global user.signingkey KEYID
git config --global commit.gpgsign true
```

**중요**: `rebase`는 커밋을 재작성하므로 서명이 무효화될 수 있습니다. 조직 정책을 확인하세요.

---

## 고급 작업 시나리오

### PR 전 히스토리 정리
```bash
# 원격 최신 상태 동기화
git fetch origin

# 기능 브랜치 정리
git rebase origin/main
git rebase -i HEAD~10 --autosquash

# 안전하게 푸시
git push --force-with-lease
```

### 복잡한 변경 재구성
```bash
# 최근 커밋을 작업 상태로 분해
git reset --mixed HEAD~1

# 변경사항 선택적 커밋
git add -p
git commit -m "주요 기능"
git add -p
git commit -m "보조 수정"

# 히스토리 최종 정리
git rebase -i HEAD~3
```

---

## 모범 사례 요약

### 안전 작업 원칙
1. **공유 브랜치 주의**: 이미 푸시된 브랜치에서는 `revert`를 우선 고려
2. **백업 습관**: 중요한 변경 전에 태그나 백업 브랜치 생성
3. **점진적 접근**: 큰 변경을 한 번에 하지 말고 작은 단위로 나누기
4. **테스트**: 변경 후 기능 테스트 필수
5. **커뮤니케이션**: 팀원과의 작업 조율

### 도구별 권장 사항
- **Reset**: 로컬 작업, 개인 브랜치, 실험적 변경에 사용
- **Revert**: 공유 브랜치, 운영 환경, 감사 추적 필요 시 사용
- **Rebase**: 개인 브랜치 정리, 선형 히스토리 유지, PR 준비 시 사용

### 복구 전략
1. **Reflog 활용**: `git reflog`로 작업 기록 확인
2. **ORIG_HEAD**: 위험 작업 후 즉시 복구 가능
3. **백업 브랜치**: 중요한 작업 전에 생성
4. **원격 백업**: 로컬 실수 대비 원격 브랜치 유지

---

## 학습을 위한 실습 예제

### Reset 실습
```bash
# 실습 환경 설정
mkdir git-reset-practice && cd git-reset-practice
git init
echo "Version 1" > file.txt
git add file.txt && git commit -m "Initial commit"

# 여러 커밋 생성
echo "Version 2" >> file.txt
git commit -am "Add feature A"
echo "Version 3" >> file.txt
git commit -am "Add feature B"

# 다양한 reset 옵션 실습
git reset --soft HEAD~1
git status  # 변경사항 확인
git reset --mixed HEAD~1
git status  # unstaged 상태 확인
git reset --hard HEAD~1
git status  # 변경사항 완전 삭제 확인
```

### Revert 실습
```bash
# 실습 환경 설정
mkdir git-revert-practice && cd git-revert-practice
git init
echo "Line 1" > document.txt
git add document.txt && git commit -m "Add line 1"

# 문제가 될 수 있는 변경사항 추가
echo "Problematic line" >> document.txt
git commit -am "Add problematic line"

# Revert로 안전하게 취소
git revert HEAD
cat document.txt  # 문제 라인 제거 확인
```

### Rebase 실습
```bash
# 실습 환경 설정
mkdir git-rebase-practice && cd git-rebase-practice
git init
echo "Base content" > app.js
git add app.js && git commit -m "Initial app"
git branch -M main

# 기능 브랜치 생성
git checkout -b feature/login
echo "Login function" >> app.js
git commit -am "Add login"
echo "Auth check" >> app.js
git commit -am "Add auth"

# 메인 브랜치 업데이트
git checkout main
echo "Header" >> app.js
git commit -am "Add header"

# Rebase 실행
git checkout feature/login
git rebase main
git log --oneline --graph  # 선형 히스토리 확인
```

---

## 자주 묻는 질문 (FAQ)

**Q: 이미 푸시한 커밋을 수정해야 할 때 무엇을 사용해야 하나요?**
A: 공유 브랜치에서는 `git revert`를 사용하세요. 이는 기존 히스토리를 보존하면서 변경사항을 취소하는 새로운 커밋을 생성합니다.

**Q: Reset과 Revert 중 어떤 것이 더 안전한가요?**
A: 협업 환경에서는 `revert`가 더 안전합니다. `reset`은 히스토리를 변경하여 다른 협업자들에게 문제를 일으킬 수 있습니다.

**Q: Rebase 중 충돌이 너무 많아요. 어떻게 해결하나요?**
A: 충돌이 많은 경우 `git rebase --abort`로 중단한 후, 작은 단위로 나누어 작업하거나, merge를 고려해보세요.

**Q: 개인 브랜치에서는 어떤 도구를 주로 사용하나요?**
A: 개인 브랜치에서는 `rebase`를 사용하여 히스토리를 깔끔하게 유지하는 것이 좋습니다. PR 전에 인터랙티브 rebase로 정리하면 리뷰어들이 변경사항을 이해하기 쉽습니다.

**Q: 실수로 `reset --hard`를 사용했어요. 복구할 수 있나요?**
A: 네, `git reflog`를 사용하여 이전 상태를 찾고 `git reset --hard HEAD@{번호}`로 복구할 수 있습니다. 다만, 변경사항이 아직 스테이징되지 않았다면 복구가 어려울 수 있습니다.

---

## 결론

Git의 `reset`, `revert`, `rebase`는 각각 고유한 목적과 사용 시나리오를 가진 강력한 도구들입니다. 이들을 효과적으로 활용하려면 각 도구의 특징과 적절한 사용 상황을 이해하는 것이 중요합니다.

### 핵심 포인트 정리

1. **적절한 도구 선택**: 상황과 목적에 맞는 도구를 선택하세요.
   - 로컬 작업 정리 → `reset`
   - 공유 브랜치 변경 취소 → `revert`
   - 히스토리 정리 및 재배치 → `rebase`

2. **안전성 우선**: 협업 환경에서는 항상 안전성을 고려하세요.
   - 공유 브랜치에서는 히스토리 변경을 최소화
   - 중요한 작업 전에는 백업 생성
   - 팀 정책과 워크플로우 준수

3. **점진적 학습**: 처음에는 작은 프로젝트에서 각 도구를 실습해보세요.
   - 실수해도 괜찮은 환경에서 연습
   - 각 옵션의 효과를 직접 확인
   - 충돌 해결 경험 축적

4. **팀 협업**: 팀의 Git 워크플로우와 정책을 이해하고 준수하세요.
   - 코드 리뷰 문화와 조화
   - CI/CD 파이프라인과의 통합
   - 문서화된 워크플로우 따르기

이 세 도구를 마스터하면 Git을 더 효율적으로 사용할 수 있을 뿐만 아니라, 더 깔끔한 코드 히스토리와 원활한 팀 협업을 이끌어낼 수 있습니다. 각 도구의 강점을 이해하고 상황에 맞게 적용하는 것이 전문 개발자로서의 중요한 역량입니다.