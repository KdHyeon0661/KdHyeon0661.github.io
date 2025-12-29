---
layout: post
title: Git - cherry-pick vs rebase
date: 2025-01-30 20:20:23 +0900
category: Git
---
# Git Cherry-pick vs Rebase

## 핵심 개념 비교

**Cherry-pick**과 **Rebase**는 Git에서 커밋을 관리하는 두 가지 강력한 도구입니다. 각각의 특징과 적절한 사용 시기를 이해하면 작업 효율성을 크게 높일 수 있습니다.

### 간단 정의
- **Cherry-pick**: 특정 브랜치의 개별 커밋(또는 연속된 몇 개의 커밋)을 현재 브랜치로 복사하여 적용
- **Rebase**: 현재 브랜치의 전체 커밋(또는 특정 범위)을 다른 베이스 커밋 위로 재배치하여 선형적인 히스토리 생성

### 선택 가이드라인
- **버그 수정이나 특정 기능만 다른 브랜치에 적용** → Cherry-pick
- **개인 브랜치의 히스토리를 깔끔하게 정리** → Rebase
- **릴리스 브랜치에 핫픽스 백포트** → Cherry-pick
- **리뷰 전에 커밋 히스토리 정리** → Rebase

---

## Git Cherry-pick 상세 설명

### 기본 개념
Cherry-pick은 다른 브랜치의 특정 커밋을 선택적으로 현재 브랜치에 적용하는 명령입니다. 원본 커밋을 복사하여 새로운 커밋을 생성하므로 커밋 해시는 변경됩니다.

### 기본 사용법
```bash
# 단일 커밋 적용
git cherry-pick abc1234

# 여러 커밋 적용 (순서대로 적용됨)
git cherry-pick abc1234 def5678 ghi9012

# 커밋 범위 적용 (주의: 아래 범위 표기법 참고)
git cherry-pick start-commit^..end-commit
```

### 범위 표기법 주의사항
Git에서 범위 표기법은 혼동하기 쉽습니다:
- `A..B`: 커밋 B에는 있지만 A에는 없는 변경사항 (그래프 차이)
- `A^..B`: 커밋 A의 직전 커밋부터 B까지의 연속된 커밋들

**실무 팁**: 적용할 범위를 먼저 확인하세요:
```bash
# 적용할 커밋 범위 확인
git log start-commit..end-commit --oneline

# 확인 후 cherry-pick 실행
git cherry-pick start-commit^..end-commit
```

### 실용적인 옵션들
```bash
# 원본 커밋 해시를 메시지에 기록 (추적성 향상)
git cherry-pick -x abc1234

# 커밋 메시지 편집기 열기
git cherry-pick -e abc1234

# 변경사항만 스테이징하고 커밋은 하지 않음
git cherry-pick -n abc1234
# 이후 git commit 으로 직접 커밋

# 서명 커밋 생성
git cherry-pick -S abc1234

# Signed-off-by 라인 추가
git cherry-pick -s abc1234
```

### 조직별 추천 설정
- **추적성 중요**: 항상 `-x` 옵션 사용
- **법적 준수 필요**: `-s` 옵션으로 Signed-off-by 추가
- **보안 정책**: `-S` 옵션으로 서명 커밋 생성

---

## Cherry-pick 충돌 처리

### 충돌 발생 시 작업 흐름
```bash
# cherry-pick 실행
git cherry-pick abc1234

# 충돌 발생 시 파일 수정
# CONFLICT (content): Merge conflict in filename

# 충돌 해결 후
git add filename
git cherry-pick --continue

# 작업 취소
git cherry-pick --abort

# 현재 커밋 건너뛰기
git cherry-pick --skip
```

### 반복 충돌 최소화
동일한 충돌 패턴이 반복될 경우:
```bash
# rerere 기능 활성화
git config --global rerere.enabled true
```

이 기능은 이전에 해결한 충돌 패턴을 기억하여 자동으로 적용해줍니다.

---

## Git Rebase 상세 설명

### 기본 개념
Rebase는 현재 브랜치의 커밋들을 다른 베이스 커밋 위로 재배치하는 명령입니다. 커밋들이 재작성되어 새로운 해시를 가지며, 결과적으로 선형적인 히스토리가 만들어집니다.

### 기본 사용법
```bash
# 현재 브랜치를 main 브랜치 기준으로 rebase
git checkout feature-branch
git rebase main

# 원격 브랜치 기준으로 rebase
git fetch origin
git rebase origin/main
```

### 충돌 처리
```bash
# rebase 중 충돌 발생 시
# 1. 충돌 파일 수정
# 2. 수정된 파일 스테이징
git add filename
# 3. rebase 계속
git rebase --continue

# rebase 취소
git rebase --abort

# 현재 커밋 건너뛰기
git rebase --skip
```

---

## 고급 기능 비교

### Cherry-pick의 고급 기능

#### 병합 커밋 처리
```bash
# 병합 커밋을 cherry-pick (첫 번째 부모 기준)
git cherry-pick -m 1 merge-commit-sha
```

#### 여러 커밋을 하나로 묶기
```bash
# 변경사항만 스테이징
git cherry-pick -n commit1 commit2 commit3

# 필요한 추가 수정 후
git commit -m "Squashed fixes: 설명"
```

### Rebase의 고급 기능

#### 특정 커밋 범위 이동 (`--onto`)
```bash
# A 커밋 이후부터 B 커밋까지를 newbase 위로 이동
git rebase --onto newbase A B
```

#### 인터랙티브 Rebase (`-i`)
```bash
# 최근 5개 커밋 정리
git rebase -i HEAD~5
```

편집기에서 사용 가능한 액션:
- `pick`: 커밋 유지
- `reword`: 커밋 메시지 수정
- `edit`: 커밋 내용 수정
- `squash`: 이전 커밋과 합치기
- `fixup`: 이전 커밋과 합치기 (메시지 무시)
- `drop`: 커밋 삭제

#### 자동 스쿼시 (`--autosquash`)
```bash
# fixup! 또는 squash!로 시작하는 커밋 자동 정렬
git rebase -i --autosquash origin/main
```

#### 병합 구조 보존 (`--rebase-merges`)
```bash
# 병합 커밋 구조를 유지하면서 rebase
git rebase --rebase-merges origin/main
```

---

## 비교 테이블: Cherry-pick vs Rebase

| 항목 | Cherry-pick | Rebase |
|------|------------|--------|
| **목적** | 특정 커밋 선택적 적용 | 브랜치 전체 재배치 |
| **결과** | 선택한 커밋이 복사됨 | 모든 커밋이 재작성됨 |
| **히스토리** | 필요한 변경만 적용 | 선형적이고 깔끔한 히스토리 |
| **충돌 처리** | 각 cherry-pick 시점에서 | 각 재배치 커밋에서 |
| **협업 안전성** | 비교적 안전 | 공유 브랜치에서 위험 |
| **사용 예시** | 버그 수정 백포트 | 리뷰 전 히스토리 정리 |
| **기록 추적** | `-x` 옵션으로 원본 추적 | reflog로 복구 가능 |

---

## 실무 시나리오

### 시나리오 1: 릴리스 브랜치에 핫픽스 적용
```bash
# main 브랜치에서 버그 수정
git checkout main
# ... 버그 수정 작업 ...
git commit -m "fix: critical bug resolved"

# 릴리스 브랜치에 해당 수정만 적용
git checkout release/1.2
git cherry-pick -x abc1234  # main의 수정 커밋
git push origin release/1.2
```

### 시나리오 2: 개인 브랜치 정리 및 병합
```bash
# 최신 main 상태 동기화
git fetch origin

# 기능 브랜치 정리
git checkout feature/login
git rebase origin/main

# 충돌 해결 후
git add .
git rebase --continue

# Fast-forward 병합
git checkout main
git merge --ff-only feature/login
```

### 시나리오 3: 특정 커밋 범위 이동
```bash
# topic 브랜치의 특정 범위를 새로운 베이스로 이동
git rebase --onto new-base start-commit end-commit
```

### 시나리오 4: 여러 브랜치의 수정사항 통합
```bash
# 다른 브랜치의 여러 핵심 수정사항만 현재 브랜치에 적용
git cherry-pick -x abc123 def456 ghi789
```

---

## 안전한 작업을 위한 팁

### 작업 전 백업
```bash
# 중요한 작업 전에 백업 브랜치 생성
git branch backup/$(date +%Y%m%d-%H%M)
# 또는 태그 사용
git tag before-operation
```

### 복구 방법
```bash
# 작업 기록 확인
git reflog

# 직전 작업 상태로 복구
git reset --hard ORIG_HEAD

# 특정 시점으로 새 브랜치 생성
git switch -c rescued-branch HEAD@{3}
```

### 협업 규칙
1. **공유 브랜치 rebase 금지**: main, release 등 공유 브랜치는 rebase하지 않음
2. **강제 푸시 주의**: rebase 후 푸시 시 `--force-with-lease` 사용
3. **커뮤니케이션**: 팀원과의 작업 조율 필수
4. **문서화**: 팀 내 Git 워크플로우 명확히 정의

---

## 문제 해결 가이드

### 일반적인 문제들

**문제 1**: Cherry-pick 충돌 반복 발생
- **해결**: `git config --global rerere.enabled true` 설정

**문제 2**: 이미 적용된 변경사항 다시 cherry-pick
- **해결**: `git cherry-pick --skip` 또는 `--allow-empty` 옵션 사용

**문제 3**: Rebase 후 푸시 거부
- **해결**: `git push --force-with-lease` 사용

**문제 4**: 잘못된 범위 cherry-pick
- **해결**: `git cherry-pick --abort` 후 올바른 범위로 재시도

### 디버깅 도구
```bash
# 상세한 실행 로그 확인
GIT_TRACE=1 git cherry-pick abc1234

# 변경사항 미리보기
git show abc1234
git diff start-commit..end-commit
```

---

## 모범 사례

### Cherry-pick 모범 사례
1. **추적성 유지**: 항상 `-x` 옵션 사용하여 원본 커밋 추적
2. **범위 확인**: 적용 전 `git log`로 커밋 범위 확인
3. **테스트**: cherry-pick 후 기능 테스트 필수
4. **문서화**: 왜 해당 커밋을 선택했는지 기록

### Rebase 모범 사례
1. **개인 브랜치 한정**: 공유되지 않은 브랜치에서만 사용
2. **최신 상태 동기화**: rebase 전 `git fetch` 실행
3. **작은 단위 작업**: 많은 커밋을 한 번에 rebase하기보다 분할
4. **백업 습관**: 중요한 rebase 전에 백업 브랜치 생성

### 공통 모범 사례
1. **충돌 해결 능력**: 충돌 해결에 익숙해지기
2. **팀 정책 준수**: 조직의 Git 정책 이해 및 준수
3. **도구 활용**: GUI 도구로 복잡한 작업 시각화
4. **지속적 학습**: 새로운 Git 기능과 모범 사례 학습

---

## 학습을 위한 실습 예제

```bash
# 실습 환경 설정
mkdir git-practice && cd git-practice
git init

# 기본 파일 생성
echo "Initial content" > file.txt
git add file.txt
git commit -m "Initial commit"
git branch -M main

# 기능 브랜치 생성
git checkout -b feature/add-header
echo "Header section" >> file.txt
git commit -am "Add header"
echo "Improved header" >> file.txt
git commit -am "Improve header styling"

# 다른 기능 브랜치 생성
git checkout -b feature/add-footer main
echo "Footer section" >> file.txt
git commit -am "Add footer"

# Cherry-pick 실습
git checkout main
# feature/add-header의 첫 번째 커밋만 적용
git cherry-pick feature/add-header~1

# Rebase 실습
git checkout feature/add-footer
git rebase main
# 충돌 발생 시 해결 과정 연습
```

---

## 결론

Git의 cherry-pick과 rebase는 각각 고유한 강점을 가진 강력한 도구입니다. 이들을 효과적으로 사용하려면 각 도구의 특징, 장단점, 적절한 사용 시기를 이해하는 것이 중요합니다.

### 핵심 요약
1. **Cherry-pick**은 정밀한 선택이 필요할 때, **Rebase**는 전체적인 정리가 필요할 때 사용하세요.
2. **안전성**을 고려하여 공유 브랜치에서는 cherry-pick을, 개인 브랜치에서는 rebase를 선호하세요.
3. **복구 전략**을 항상 마련해두고, 중요한 작업 전에는 백업을 생성하세요.
4. **팀 협업**을 위해 Git 워크플로우와 정책을 명확히 정의하고 준수하세요.

### 최종 조언
두 도구 모두 마스터하기 위해 꾸준한 실습이 필요합니다. 처음에는 작은 프로젝트나 실습 환경에서 시작하여 점차 복잡한 시나리오로 확장해나가세요. 각 도구의 미묘한 차이와 함정을 이해하면 실제 프로젝트에서 더 자신감 있게 사용할 수 있습니다.

가장 중요한 것은 상황에 맞는 도구 선택입니다. 때로는 cherry-pick이 더 적합하고, 때로는 rebase가 더 나은 선택일 수 있습니다. 프로젝트의 요구사항, 팀의 워크플로우, 개인의 선호도를 종합적으로 고려하여 최적의 결정을 내리세요.

Git은 강력한 도구이지만, 그 힘은 올바른 사용법을 아는 데서 비롯됩니다. cherry-pick과 rebase를 효과적으로 활용하면 더 깔끔한 코드 히스토리와 효율적인 협업 환경을 만들 수 있습니다.