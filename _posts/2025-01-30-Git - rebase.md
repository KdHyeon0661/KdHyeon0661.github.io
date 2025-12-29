---
layout: post
title: Git - rebase
date: 2025-01-30 19:20:23 +0900
category: Git
---
# Git Rebase 완전 가이드

## 핵심 개념

Git rebase는 브랜치의 커밋들을 새로운 기준점으로 재배치하여 깔끔한 선형 히스토리를 만드는 강력한 도구입니다. 개인 브랜치 정리와 코드 리뷰 전 히스토리 단순화에 특히 유용하지만, 공유 브랜치에서는 주의가 필요합니다.

---

## Rebase 기본 이해

### Rebase란 무엇인가?
Rebase는 현재 브랜치의 커밋들을 새로운 기준 커밋(base) 뒤로 이동시키고, 이동된 커밋들은 새로운 해시값을 갖는 새 커밋으로 다시 작성됩니다. 이로 인해 병합 커밋 없이 선형적인 히스토리를 만들 수 있습니다.

### 시각적 이해
```
rebase 전:
main:     A---B
               \
feature:        C---D

rebase 후:
main:     A---B
                   \
feature:            C'---D'
```
C와 D 커밋이 C'와 D'로 재작성되어 main 브랜치의 최신 커밋 B 뒤에 배치됩니다.

### 기본 사용법
```bash
git checkout feature/login
git rebase main
```

---

## Rebase vs Merge: 실무적 비교

| 항목 | Merge | Rebase |
|------|-------|--------|
| **히스토리 형태** | 병합 커밋 생성 (분기 흔적 보존) | 선형 히스토리 (분기 흔적 제거) |
| **가독성** | 브랜치 흐름 명확 | 히스토리 간결 |
| **충돌 처리** | 한 번의 충돌 해결 | 각 커밋별 충돌 해결 필요 |
| **협업 안정성** | 높음 (추천) | 공유 브랜치에서 위험 |
| **사용 시기** | 협업 브랜치, 기록 보존 | 개인 브랜치 정리 |
| **FF 병합** | 필요 없음 | 쉽게 가능 |

---

## Rebase 작업 흐름

### 1. 기본 Rebase 실행
```bash
# 대상 브랜치로 이동
git checkout feature/login

# main 브랜치 기준으로 rebase
git rebase main
```

### 2. 충돌 해결 과정
Rebase 중 충돌이 발생하면 Git은 작업을 일시 중지합니다:
```bash
# 충돌 메시지 예시:
CONFLICT (content): Merge conflict in index.html

# 충돌 파일 수정 후
git add index.html

# rebase 계속 진행
git rebase --continue

# rebase 중단 및 원복
git rebase --abort

# 현재 커밋 건너뛰기 (주의: 커밋 내용 손실)
git rebase --skip
```

### 3. 자동 충돌 해결 기록 (rerere)
동일한 충돌 패턴을 반복해서 해결해야 할 경우:
```bash
# rerere 기능 활성화
git config --global rerere.enabled true
```

---

## Interactive Rebase: 히스토리 정리

### 인터랙티브 모드 실행
```bash
# 최근 5개 커밋 정리
git rebase -i HEAD~5
```

### 액션 명령어
편집기에서 사용 가능한 주요 액션:

| 액션 | 설명 |
|------|------|
| `pick` | 커밋을 그대로 유지 |
| `reword` | 커밋 메시지만 수정 |
| `edit` | 커밋 내용 수정 (커밋 후 `git commit --amend`) |
| `squash` | 이전 커밋과 합치고 메시지 병합 |
| `fixup` | 이전 커밋과 합치되 메시지 무시 |
| `drop` | 커밋 완전 제거 |
| `exec` | 해당 시점에서 쉘 명령 실행 |

### 예시 구성
```
pick a1b2c3d Login 기능 구현
reword e4f5g6h 불필요한 로그 제거
fixup i7j8k9l 코드 포맷팅 수정
exec npm test
```

### 자동 스쿼시 기능
`fixup!`이나 `squash!`로 시작하는 커밋 메시지를 사용하면:
```bash
git rebase -i --autosquash origin/main
```

---

## 고급 Rebase 기법

### 1. 특정 커밋 범위 이동 (`--onto`)
특정 커밋 범위만 새로운 베이스로 이동시킬 때 사용:
```bash
# A 커밋 이후부터 B 커밋까지를 newbase 위로 이동
git rebase --onto newbase A B
```

**실용적 예시**:
```bash
# 이동할 커밋 범위 확인
git log --oneline feature-start..feature-end

# 실제 이동 실행
git rebase --onto new-base feature-start feature-end
```

### 2. 병합 커밋 구조 보존 (`--rebase-merges`)
복잡한 브랜치 구조를 유지하면서 rebase:
```bash
git rebase --rebase-merges origin/main
```

### 3. 작업 디렉토리 자동 저장 (`--autostash`)
Rebase 시작 전 변경사항이 있을 경우:
```bash
# 일회성 사용
git rebase --autostash origin/main

# 전역 설정
git config --global rebase.autoStash true
```

---

## Pull 시 Rebase 활용

### 기본 설정
```bash
# pull 시 항상 rebase 사용
git config --global pull.rebase true

# 또는 개별 실행
git pull --rebase
```

### 고급 옵션
```bash
# 병합 구조 보존하며 rebase
git pull --rebase=merges

# 인터랙티브 모드로 rebase
git pull --rebase=interactive
```

---

## Rebase 후 변경사항 비교

### 변경 전후 비교 (`range-diff`)
Rebase 전후의 패치 시리즈 변화 확인:
```bash
# main 브랜치 기준, topic 브랜치 변경 비교
git range-diff main...topic
```

---

## 안전한 푸시 전략

Rebase는 커밋 해시를 변경하므로 강제 푸시가 필요합니다:
```bash
# 안전한 강제 푸시 (동료 작업 보호)
git push --force-with-lease origin feature/login
```

**중요**: `--force-with-lease`는 로컬이 알고 있는 원격 상태와 실제 원격 상태가 다를 경우 푸시를 거부하여 동료의 작업을 보호합니다.

---

## 사고 대비 및 복구 방법

### 작업 전 백업
```bash
# rebase 전 백업 브랜치 생성
git branch backup/feature-login-$(date +%Y%m%d)

# 또는 태그 사용
git tag before-rebase
```

### 복구 방법
```bash
# HEAD 이동 기록 확인
git reflog

# 직전 작업 상태로 복구
git reset --hard ORIG_HEAD

# 특정 시점으로 새 브랜치 생성
git switch -c rescued-branch HEAD@{3}
```

---

## 특수 상황 처리

### 서브모듈이 있는 경우
서브모듈 포인터가 rebase 과정에서 변경될 수 있습니다:
```bash
# 서브모듈 디렉토리에서 원하는 커밋으로 체크아웃
cd submodule-directory
git checkout desired-commit

# 상위 저장소에서 변경사항 커밋
cd ..
git add submodule-directory
git commit -m "Update submodule pointer"
```

### Git LFS와 함께 사용
LFS 포인터 파일 확인 필요:
```bash
# LFS 설치 확인
git lfs install

# 포인터 파일 체크아웃
git checkout -- .
```

---

## 팀 협업을 위한 규칙

### 기본 원칙
1. **공유 브랜치 rebase 금지**: main, release, hotfix 브랜치는 rebase하지 않음
2. **개인 브랜치만 rebase**: 푸시 전에 개인 브랜치 정리용으로 사용
3. **강제 푸시 주의**: 항상 `--force-with-lease` 사용
4. **사전 동의**: 팀 내 rebase 정책 수립 및 준수

### PR 병합 정책
- **Rebase and Merge**: 선형 히스토리 유지 (개인 브랜치에 적합)
- **Squash and Merge**: 단일 커밋으로 병합 (기능 단위 정리)
- **Merge Commit**: 전체 히스토리 보존 (협업 브랜치에 적합)

---

## 실전 시나리오

### 시나리오 1: 개인 브랜치 정리 및 FF 병합
```bash
# 원격 최신 상태 동기화
git fetch origin

# 기능 브랜치로 이동
git checkout feature/payment

# main 기준으로 rebase
git rebase origin/main

# 충돌 해결 후 계속
# (반복: 충돌 수정 → git add → git rebase --continue)

# main 브랜치로 이동 및 FF 병합
git checkout main
git merge --ff-only feature/payment
```

### 시나리오 2: 특정 커밋 범위 이전
```bash
# topic 브랜치에서 start-commit과 end-commit 사이를 new-base로 이동
git checkout topic
git rebase --onto new-base start-commit end-commit
```

### 시나리오 3: 자동화된 테스트와 함께 rebase
```bash
git rebase -i HEAD~4
# 편집기에서:
# pick abc1234 Initial implementation
# exec npm test
# pick def5678 Refactor component
# exec npm run lint
```

---

## 문제 해결 가이드

### 일반적인 문제 및 해결책

**문제 1**: `Permission denied` 또는 푸시 거부
- **원인**: Rebase로 인한 해시 변경
- **해결**: `git push --force-with-lease origin 브랜치명`

**문제 2**: 반복적인 충돌
- **원인**: 여러 커밋에 동일한 충돌 발생
- **해결**: `git config --global rerere.enabled true` 설정

**문제 3**: 작업 디렉토리 변경사항 손실
- **원인**: Rebase 전 stash하지 않음
- **해결**: `--autostash` 옵션 사용 또는 수동 `git stash`

**문제 4**: 서브모듈 포인터 오류
- **원인**: Rebase 중 서브모듈 참조 변경
- **해결**: 서브모듈 디렉토리에서 적절한 커밋 체크아웃

### 디버깅 도구
```bash
# 상세한 rebase 진행 상황 확인
GIT_TRACE=1 git rebase main

# 충돌 상태 확인
git status
git diff
```

---

## 모범 사례 요약

### 안전한 Rebase 원칙
1. **개인 브랜치 한정**: 공유 브랜치는 rebase하지 않기
2. **최신 상태 유지**: rebase 전 `git fetch`로 원격 상태 동기화
3. **백업 습관**: 대규모 rebase 전 백업 브랜치 생성
4. **점진적 접근**: 많은 커밋을 한 번에 rebase하기보다 작은 단위로 나누기
5. **테스트 통합**: `exec` 명령으로 rebase 중 자동 테스트 실행

### 협업 환경 고려사항
1. **팀 정책 수립**: rebase 사용 규칙 명확히 정의
2. **커뮤니케이션**: 공유 브랜치 rebase 시 팀원 사전 공지
3. **도구 활용**: `range-diff`로 변경사항 시각화하여 리뷰 지원
4. **자동화**: CI/CD 파이프라인과 rebase 워크플로우 통합

### 생산성 팁
1. **에일리어스 설정**: 자주 사용하는 rebase 명령 단축어 설정
2. **GUI 도구 활용**: 복잡한 인터랙티브 rebase는 소스트리 등 GUI 도구 활용
3. **연습 환경**: 실제 프로젝트보다 연습 저장소에서 먼저 익히기
4. **문서화**: 팀 내 rebase 체크리스트 및 가이드라인 공유

---

## 학습을 위한 실습 환경

```bash
# 실습용 저장소 생성
mkdir rebase-practice && cd rebase-practice
git init

# 기본 파일 생성 및 초기 커밋
echo "Initial content" > file.txt
git add file.txt
git commit -m "Initial commit"
git branch -M main

# 기능 브랜치 생성 및 작업
git checkout -b feature/add-menu
echo "Menu item 1" >> file.txt
git commit -am "Add menu item 1"
echo "Menu item 2" >> file.txt
git commit -am "Add menu item 2"

# main 브랜치에서도 작업
git checkout main
echo "Footer content" >> file.txt
git commit -am "Add footer"

# Rebase 실습
git checkout feature/add-menu
git rebase main

# 충돌 발생 시험 (의도적으로 충돌 만들기)
# ... 충돌 해결 과정 연습

# 인터랙티브 rebase 연습
git rebase -i HEAD~3
```

---

## 결론

Git rebase는 개발 워크플로우에서 히스토리를 깔끔하게 관리할 수 있는 강력한 도구입니다. 적절히 사용하면 선형적인 커밋 히스토리로 코드 리뷰와 디버깅을 용이하게 만들 수 있습니다.

그러나 rebase의 힘은 신중한 사용에서 비롯됩니다. 공유 브랜치에서의 rebase는 팀 협업에 심각한 문제를 초래할 수 있으므로, 반드시 개인 브랜치에서만 사용하고 팀원들과의 명확한 합의 하에 진행해야 합니다.

가장 중요한 것은 상황에 맞는 도구 선택입니다. 히스토리의 완전성을 유지해야 하는 경우에는 merge를, 깔끔한 선형 히스토리가 더 중요한 경우에는 rebase를 선택하세요. 두 방법 모두 장단점이 있으며, 프로젝트의 성격과 팀의 워크플로우에 가장 적합한 방식을 채택하는 것이 핵심입니다.

Rebase를 마스터하는 것은 Git을 전문적으로 사용하는 개발자에게 필수적인 스킬입니다. 처음에는 조금 복잡해 보일 수 있지만, 체계적인 학습과 꾸준한 실습을 통해 자연스럽게 익힐 수 있습니다. 작은 프로젝트에서 시작해 점차 복잡한 시나리오로 확장해가며 rebase의 진정한 힘을 경험해 보세요.