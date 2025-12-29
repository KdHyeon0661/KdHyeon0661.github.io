---
layout: post
title: Git - GitHub 웹에서 PR 충돌 해결
date: 2025-02-06 19:20:23 +0900
category: Git
---
# Git 충돌 해결 가이드

## 충돌이란 무엇인가?

Git에서 충돌은 두 개 이상의 브랜치가 동일한 파일의 동일한 부분을 서로 다르게 수정했을 때 발생합니다. 이 경우 Git은 어떤 변경사항을 선택해야 할지 판단할 수 없으므로, 사용자가 직접 해결해야 합니다.

GitHub에서 Pull Request를 생성할 때 충돌이 있으면 다음과 같은 메시지를 볼 수 있습니다:
```
This branch has conflicts that must be resolved
```

이 상태에서는 자동 병합이 불가능하며, 사용자가 충돌을 해결해야만 PR을 병합할 수 있습니다.

---

## GitHub 웹사이트에서 충돌 해결하기

GitHub의 웹 인터페이스를 사용하면 간단한 충돌을 쉽게 해결할 수 있습니다.

### 단계별 해결 과정

1. **PR 페이지 접속**: 충돌이 발생한 Pull Request 페이지로 이동합니다.

2. **충돌 확인**: 상단에 "This branch has conflicts that must be resolved" 메시지가 표시됩니다. **Resolve conflicts** 버튼을 클릭합니다.

3. **충돌 파일 편집**:
   충돌이 발생한 파일은 다음과 같은 마커로 표시됩니다:
   ```html
   <<<<<<< HEAD
   <!-- 메인 브랜치의 내용 -->
   =======
   <!-- PR 브랜치의 내용 -->
   >>>>>>> feature/header-update
   ```

4. **수동으로 해결**:
   원하는 코드로 수정하고 모든 충돌 마커(`<<<<<<<`, `=======`, `>>>>>>>`)를 삭제합니다.

   예시:
   ```html
   <!-- 수정 전 -->
   <<<<<<< HEAD
   <h1>Welcome to Home Page</h1>
   =======
   <h1>Welcome to Our Service</h1>
   >>>>>>> feature/header-update
   
   <!-- 수정 후 -->
   <h1>Welcome to Our Home Page</h1>
   ```

5. **변경사항 저장**:
   - **Mark as resolved** 버튼을 클릭합니다.
   - 커밋 메시지를 확인하고 **Commit merge** 버튼을 클릭합니다.

6. **병합 완료**:
   충돌이 해결되면 PR을 병합할 수 있는 상태가 됩니다.

### 권한 주의사항
- PR 브랜치에 쓰기 권한이 있어야 웹에서 충돌을 해결할 수 있습니다.
- fork에서 온 PR의 경우, **Allow edits by maintainers** 옵션이 활성화되어 있어야 저장소 관리자가 충돌을 해결할 수 있습니다.

---

## 로컬에서 충돌 해결하기

복잡한 충돌이나 대량의 충돌 파일이 있는 경우 로컬에서 해결하는 것이 더 효율적입니다.

### 기본 해결 방법 (Merge 방식)

```bash
# 메인 브랜치를 최신 상태로 업데이트
git checkout main
git pull origin main

# 작업 브랜치로 전환
git checkout feature/header-update

# 메인 브랜치를 병합 (충돌 발생)
git merge main

# 충돌이 발생한 파일을 편집기로 열어 수정
# <<<<<<< HEAD ... >>>>>>> main 마커 사이의 내용을 적절히 통합

# 수정 완료 후
git add 수정한파일.html
git commit -m "resolve: 헤더 충돌 해결"
git push origin feature/header-update
```

### Rebase 방식으로 해결하기 (선형 히스토리 유지)

```bash
# 작업 브랜치로 전환
git checkout feature/header-update

# 메인 브랜치 기준으로 리베이스
git fetch origin
git rebase origin/main

# 충돌 발생 시 각 커밋별로 해결
# 충돌 파일 수정 → git add → git rebase --continue
# 반복...

# 원격 저장소에 푸시 (강제 푸시 필요)
git push --force-with-lease origin feature/header-update
```

**주의**: 리베이스는 커밋 히스토리를 재작성하므로, 이미 다른 사람이 가져간 브랜치에서는 사용을 피해야 합니다.

### 유용한 도구와 명령어

```bash
# 충돌 상태 확인
git status

# 한쪽 버전 선택하기
git checkout --ours 파일명    # 현재 브랜치 버전 선택
git checkout --theirs 파일명  # 병합 대상 브랜치 버전 선택

# 시각적 병합 도구 사용
git mergetool

# 병합 중단하기
git merge --abort
git rebase --abort
```

---

## 다양한 충돌 유형과 해결 전략

### 1. 일반 텍스트 파일 충돌
가장 흔한 유형으로, 동일한 파일의 동일한 줄을 수정했을 때 발생합니다.
**해결**: 수동으로 코드를 통합하고 충돌 마커를 삭제합니다.

### 2. 파일 이름 변경 충돌
한 브랜치에서는 파일 이름을 변경하고, 다른 브랜치에서는 같은 파일을 수정했을 때 발생합니다.
**해결**: 최종 파일명을 결정하고 내용을 통합합니다. 로컬에서 해결하는 것이 더 쉽습니다.

### 3. 바이너리 파일 충돌
이미지, PDF, 컴파일된 파일 등의 충돌은 자동 해결이 불가능합니다.
**해결**: 한쪽 버전을 선택하거나, 새로운 버전의 파일을 생성합니다.
```bash
# 한쪽 버전 선택
git checkout --ours image.png
# 또는
git checkout --theirs image.png
```

### 4. 삭제-수정 충돌
한 브랜치에서는 파일을 삭제하고, 다른 브랜치에서는 같은 파일을 수정했을 때 발생합니다.
**해결**: 파일을 유지할지 삭제할지 결정합니다.
```bash
# 파일 유지 (수정된 내용을 채택)
git checkout --theirs 파일명
git add 파일명

# 파일 삭제
git rm 파일명
```

---

## 실전 연습: 로컬에서 충돌 시뮬레이션

로컬에서 충돌 상황을 재현해보며 실습할 수 있습니다.

```bash
# 새로운 디렉토리 생성 및 Git 초기화
mkdir conflict-practice
cd conflict-practice
git init
git branch -M main

# 초기 파일 생성
echo "첫 번째 줄" > test.txt
echo "두 번째 줄" >> test.txt
git add test.txt
git commit -m "초기 파일 생성"

# 기능 브랜치 생성 및 수정
git checkout -b feature-branch
echo "수정된 첫 번째 줄 (기능)" > test.txt
echo "두 번째 줄" >> test.txt
git commit -am "기능 브랜치에서 수정"

# 메인 브랜치에서 다른 수정
git checkout main
echo "수정된 첫 번째 줄 (메인)" > test.txt
echo "두 번째 줄" >> test.txt
git commit -am "메인 브랜치에서 수정"

# 충돌 발생 (병합 시도)
git merge feature-branch

# 충돌 해결
# test.txt 파일을 열어 충돌을 수동으로 해결
# <<<<<<< HEAD ... >>>>>>> feature-branch 사이의 내용 통합

# 해결 완료
git add test.txt
git commit -m "충돌 해결 완료"
```

---

## 문제 해결 팁

### 자주 발생하는 문제들

1. **"Resolve conflicts" 버튼이 보이지 않음**
   - 원인: PR 브랜치에 대한 쓰기 권한이 없음
   - 해결: 저장소 관리자에게 권한 요청 또는 로컬에서 해결

2. **충돌 해결 후에도 병합이 안 됨**
   - 원인: 브랜치 보호 규칙(필수 리뷰, 상태 검사) 미충족
   - 해결: 필요한 리뷰를 받고 CI 테스트가 통과되도록 함

3. **복잡한 충돌로 웹 편집이 어려움**
   - 원인: 여러 파일의 충돌 또는 바이너리 파일 충돌
   - 해결: 로컬에서 mergetool 등의 도구를 사용

4. **리베이스 후 푸시가 거부됨**
   - 원인: 히스토리 재작성으로 원격과 불일치
   - 해결: `git push --force-with-lease` 사용

### 생산성 향상을 위한 팁

```bash
# 반복되는 충돌 해결을 자동화
git config --global rerere.enabled true

# 선호하는 병합 도구 설정
git config --global merge.tool vscode
git config --global mergetool.vscode.cmd 'code --wait $MERGED'

# 충돌 해결 도움말 보기
git help merge
git help mergetool
```

---

## 마무리

Git 충돌은 협업 개발에서 피할 수 없는 부분이지만, 올바른 방법으로 접근하면 두려울 필요가 없습니다. 충돌을 효과적으로 관리하려면 다음 원칙을 기억하세요:

**핵심 원칙**

1. **작은 단위로 자주 병합하세요**: 큰 변경사항보다 작은 변경사항을 자주 병합하면 충돌 규모와 해결 난이도가 줄어듭니다.

2. **팀 규칙을 정하세요**: 어떤 상황에서 merge를 사용하고, 어떤 상황에서 rebase를 사용할지 팀원들과 합의하세요.

3. **커뮤니케이션이 중요합니다**: 충돌이 발생했을 때 관련 팀원들과 소통하며 해결 방안을 논의하세요.

4. **도구를 활용하세요**: 웹 인터페이스는 간단한 충돌에, 로컬 도구(mergetool)는 복잡한 충돌에 적합합니다.

5. **연습이 완벽을 만듭니다**: 충돌 해결에 익숙해지려면 실제로 경험해보는 것이 가장 좋습니다. 연습용 저장소를 만들어 다양한 충돌 시나리오를 경험해보세요.

충돌은 실수가 아니라, 팀원들이 각자 열심히 작업하고 있다는 증거입니다. 적절한 도구와 명확한 프로세스를 통해 충돌을 효율적으로 관리하면, 개발 팀의 생산성과 코드 품질을 동시에 향상시킬 수 있습니다.