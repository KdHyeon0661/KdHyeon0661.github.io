---
layout: post
title: Git - Rebase + Autosquash
date: 2025-02-26 20:20:23 +0900
category: Git
---
# Git Rebase + Autosquash로 커밋 정리하는 법

## 핵심 요약

Git의 `rebase`와 `autosquash` 기능을 결합하면, 개발 중 생성된 자잘한 커밋들을 효과적으로 정리하여 깔끔한 프로젝트 히스토리를 유지할 수 있습니다. 이 방식은 `fixup!` 또는 `squash!` 접두어가 붙은 커밋을 `git rebase -i --autosquash` 명령 실행 시 **자동으로 원본 커밋 바로 아래로 재배치하고, 적절한 동작(`fixup` 또는 `squash`)으로 설정**해 줍니다.

`git commit --fixup=<커밋>` 또는 `git commit --squash=<커밋>` 명령을 사용하면 정리 대상이 될 원본 커밋을 해시 등을 통해 정확히 지정할 수 있습니다. 작업 시에는 인터랙티브 리베이스의 범위를 충분히 넓게 잡으면서도, 충돌 발생에 대비해 단계적으로 진행하고, `git reflog`를 통해 언제든 복구할 수 있도록 하는 것이 안전합니다. 리베이스로 커밋 히스토리가 변경된 후에는 `git push --force-with-lease` 명령을 사용하여 원격 저장소를 안전하게 업데이트하세요.

반복적인 작업을 줄이려면 `rebase.autosquash=true`, `pull.rebase=true` 등의 설정을 글로벌(`--global`)로 활성화하는 것이 좋습니다.

---

## 커밋 정리가 필요한 이유

기능 하나를 개발하는 과정에서는 수많은 작은 수정—오타 교정, 디버그 로그 제거, 변수명 변경 등—이 발생하며, 이로 인해 **의미가 중복되거나 중요도가 낮은 자잘한 커밋**들이 히스토리에 쌓이게 됩니다. 이러한 혼란스러운 히스토리는 PR 리뷰어의 이해를 어렵게 하고, 미래에 버그를 추적하거나 코드 변경 사항을 파악하려는 개발자에게도 장애물이 됩니다.

따라서 커밋을 정리하는 목표는 다음과 같습니다:
- **의도가 명확한 커밋**만 남겨, 기능 추가(`feat`), 버그 수정(`fix`), 리팩토링(`refactor`) 등의 맥락을 쉽게 파악할 수 있게 합니다.
- **리뷰의 단위**와 **릴리스 노트에 기재될 변경 사항의 단위**를 일치시킵니다.
- 회귀 버그 분석 시 특정 변경 사항을 빠르게 찾아내고 **범위를 한정**(양자화)할 수 있도록 합니다.

---

## `--autosquash` 동작 원리 이해

### 매칭 규칙
`git rebase -i`(인터랙티브 리베이스)에 `--autosquash` 옵션을 함께 사용하면, Git은 리베이스 대상 범위 내에서 특별한 형식의 커밋 메시지를 찾습니다.
- **`fixup! <원본 커밋 제목>`**
- **`squash! <원본 커밋 제목>`**

이러한 메시지를 가진 커밋을 발견하면, Git은 동일한 제목을 가진 **원본 커밋(보통 가장 가까운 조상 커밋)** 을 찾아, 리베이스 수행 순서 목록(To-Do list)에서 해당 원본 커밋 바로 아래로 자동 재배치합니다. 또한, 접두어에 따라 자동으로 적절한 동작을 지정합니다.
- `fixup!` → `fixup` (원본 커밋 메시지를 유지하고, 본 커밋의 메시지는 버림)
- `squash!` → `squash` (원본 커밋과 병합하며, 커밋 메시지를 통합할 수 있는 편집기를 엶)

### 동작 시점
자동 정렬과 동작 지정은 `git rebase -i --autosquash <범위>` 명령을 실행하여 **리베이스 To-Do 리스트를 생성하는 시점에 일어납니다**. 이미 생성된 편집 화면에서 사용자가 수동으로 순서나 동작을 변경할 수도 있지만, 이 기능의 본래 목적은 자동화를 통해 수작업을 최소화하는 데 있습니다.

---

## 실전 워크플로우: 가장 많이 쓰는 3가지 패턴

### 패턴 A: 소규모 기능 개발 후 정리
하나의 기능 개발과 관련된 몇 개의 커밋만 정리하는 경우입니다.
```bash
# 1. 핵심 기능 커밋과 자잘한 수정 커밋을 순서대로 생성
git commit -m "feat: 게시글 작성 기능"
git commit -m "fixup! feat: 게시글 작성 기능"  # 오타 수정 등
git commit -m "squash! feat: 게시글 작성 기능" # 설명 보완 등

# 2. 최근 3개 커밋을 대상으로 자동 정리 실행
git rebase -i --autosquash HEAD~3
```
실행 후 열리는 편집기에는 다음과 유사한 목록이 자동으로 배치됩니다.
```
pick   a1b2c3 feat: 게시글 작성 기능
fixup  d4e5f6 fixup! feat: 게시글 작성 기능
squash e7f8g9 squash! feat: 게시글 작성 기능
```
저장하고 나가면, `fixup` 커밋은 자동으로 병합되고, `squash` 커밋에 대해서는 메시지 통합을 위한 편집기가 추가로 열릴 수 있습니다.

### 패턴 B: 정확한 커밋 해시를 사용한 안전한 정리
동일한 제목의 커밋이 여러 개 있을 경우를 대비해, 정리 대상이 되는 원본 커밋을 해시로 정확히 지정하는 방법입니다.
```bash
# 원본 커밋의 해시를 사용하여 fixup/squash 커밋 생성
git commit --fixup=a1b2c3d4
git commit --squash=e5f6a7b8

# 넉넉한 범위(예: 최근 15개 커밋)를 설정하여 리베이스 실행
git rebase -i --autosquash HEAD~15
```

### 패턴 C: 중규모 분기 작업 중 간헐적 정리
긴 기능 브랜치에서 작업하면서 중간중간 정리를 하고 싶을 때 유용합니다.
```bash
# 작업 중, 이전에 만든 특정 기능 커밋(해시: FEA1234)을 계속 보완
git commit --fixup=FEA1234
git commit --fixup=FEA1234
git commit --squash=FEA1234

# 넓은 범위(예: origin/main 기준)로 리베이스하여 전체 흐름 정리
git rebase -i --autosquash origin/main
# 또는 HEAD~N 과 같은 상대적 범위도 가능
```

---

## `git commit --fixup`과 `--squash` 명령 상세

이 명령어들은 정리 대상을 명시적으로 지정하면서 `fixup!`/`squash!` 접두어가 붙은 커밋을 생성하는 편리한 방법입니다.

```bash
# 가장 안전한 방법: 원본 커밋 해시 직접 지정
git commit --fixup=a1b2c3d4
git commit --squash=e5f6a7b8

# (주의) 커밋 메시지로 대상 지정 (중복 제목이 있을 경우 매칭 오류 가능)
git commit --fixup="feat: 게시글 작성 기능"
```
- `--fixup`: 생성된 커밋은 나중에 리베이스 시 **원본 커밋에 자동으로 병합되며, 원본 커밋의 메시지가 유지**됩니다. 본 커밋의 메시지는 사용되지 않습니다.
- `--squash`: 생성된 커밋은 리베이스 시 원본 커밋과 병합되지만, **커밋 메시지를 통합할 수 있는 편집기가 열립니다**. 이를 통해 변경 사항에 대한 설명을 다듬을 기회가 주어집니다.

> 최신 버전의 Git에서는 `--fixup=amend:<커밋>` 또는 `--fixup=reword:<커밋>`과 같은 확장 옵션을 제공하기도 합니다. 팀 전체가 사용하는 Git 버전을 확인하고, 이러한 고급 기능을 사용할 때는 문서화하여 혼선을 방지하세요.

---

## 인터랙티브 리베이스의 범위 설정 전략

적절한 범위를 선택하는 것은 안전성과 효율성에 중요합니다.
- **소규모 안전 범위**: `HEAD~3`, `HEAD~5`처럼 최근 몇 개의 커밋만 대상으로 합니다. 충돌 가능성이 낮고 관리가 쉽습니다.
- **기능 브랜치 전체 정리**: 현재 브랜치의 모든 변경 사항을 정리하고 싶다면, 베이스 브랜치(예: `main`)와의 공통 조상(merge base)을 기준으로 범위를 설정합니다.
  ```bash
  # 현재 브랜치와 origin/main의 공통 조상 커밋을 찾아 그 이후부터 리베이스
  BASE_COMMIT=$(git merge-base HEAD origin/main)
  git rebase -i --autosquash $BASE_COMMIT
  ```
- **PR 기준 정리**: "이번 PR에 포함될 모든 커밋"을 정리 대상으로 삼습니다. 이는 리뷰어에게 깔끔한 변경 내역을 제공합니다.

---

## 리베이스 중 충돌 발생 시 대처법

리베이스는 커밋을 순차적으로 재적용하는 과정이므로, 중간에 충돌이 발생할 수 있습니다.
```bash
git rebase -i --autosquash HEAD~20
# ... 충돌 발생 메시지 출력 ...
```
1.  `git status` 명령으로 충돌이 발생한 파일을 확인합니다.
2.  해당 파일들을 열어 `<<<<<<<`, `=======`, `>>>>>>>` 마커를 제거하고 원하는 코드로 수동 통합합니다.
3.  충돌 해결이 완료된 파일을 스테이징합니다.
    ```bash
    git add <수정한-파일-경로>
    ```
4.  리베이스 작업을 계속 진행합니다.
    ```bash
    git rebase --continue
    ```
5.  만약 충돌 해결이 너무 복잡하거나 중단하고 싶다면, 다음 명령으로 리베이스를 완전히 취소하고 원래 상태로 돌아갈 수 있습니다.
    ```bash
    git rebase --abort
    ```

**팁**: 많은 커밋을 한 번에 리베이스할 때 여러 번 연속으로 충돌이 발생할 수 있습니다. 이 경우 범위를 줄이거나, 기능별로 나누어 여러 번에 걸쳐 리베이스를 수행하는 전략을 고려하세요.

---

## 리베이스 후 안전하게 원격 저장소에 푸시하기

리베이스는 커밋의 해시를 재생성하여 히스토리를 변경합니다. 따라서 기존 원격 브랜치의 히스토리와 일치하지 않게 되어, 일반적인 `git push`는 실패합니다. 이때 **`--force-with-lease`** 옵션을 사용하는 것이 안전합니다.

```bash
git push --force-with-lease origin feature/my-feature
```
- `--force-with-lease`는 단순한 `--force`와 달리, **로컬이 인지하고 있는 원격 브랜치의 상태가 실제 원격 상태와 동일한지 확인한 후에만 강제 푸시를 수행**합니다. 이는 다른 협업자가 동시에 푸시한 내용을 실수로 덮어쓰는 위험을 줄여줍니다.
- GitHub 등의 호스트 서비스에서는 브랜치 보호 규칙(`Require linear history` 등)을 설정하여 히스토리 재작성을 제어할 수 있습니다.

---

## 반복 작업을 줄이는 Git 설정

매번 `--autosquash` 옵션을 입력하지 않도록, 또는 풀(pull) 시 자동으로 리베이스되도록 전역 설정을 할 수 있습니다.

```bash
# 리베이스 시 자동으로 --autosquash 동작 활성화
git config --global rebase.autosquash true

# git pull 명령 실행 시 기본 동작을 merge 대신 rebase로 설정
git config --global pull.rebase true

# 리베이스 실행 전, 작업 중인 변경사항을 자동으로 stash 했다가 완료 후 복구
git config --global rebase.autoStash true

# 푸시 시 현재 브랜치만 푸시하도록 안전 설정 (기본값)
git config --global push.default simple
```

이러한 설정은 팀의 공통 개발 환경(예: Dev Container, 공유 셋업 스크립트)에 포함시켜 모든 구성원이 일관된 환경에서 작업할 수 있도록 하는 것이 좋습니다.

---

## 대규모 PR 정리를 위한 체계적인 접근법

### Conventional Commits와의 조화
커밋 메시지를 `feat:`, `fix:`, `refactor:`와 같은 규칙(Conventional Commits)으로 작성하면, 리베이스 시 **의도별로 커밋을 그룹화**하기가 훨씬 수월해집니다. 자잘한 수정은 항상 `--fixup` 커밋으로 생성하고, 리베이스 시 `squash` 또는 `fixup` 처리하세요.

### 대상 베이스 커밋 고정 및 반복
정리의 중심이 될 "대표 커밋"의 해시를 찾아 고정하고, 모든 관련 수정 사항을 이 커밋을 대상으로 하는 `--fixup` 커밋으로 만듭니다.
```bash
# 핵심 기능 커밋 해시가 FEA1234 라고 가정
git commit --fixup=FEA1234
git commit --fixup=FEA1234
git commit --squash=FEA1234
# 충분한 범위로 리베이스 실행
git rebase -i --autosquash HEAD~20
```

### 단계적 분할 정리
한 번에 50개의 커밋을 정리하려다 보면 복잡성과 충돌 위험이 급증합니다. 대신 기능 단위나 모듈 단위로 구분하여 **여러 번의 작은 리베이스로 나누어 실행**하는 것이 안전하고 심리적 부담도 적습니다.

---

## 팀 협업 및 CI/CD 파이프라인과의 통합

- **커밋 메시지 린트(Commitlint)**: Conventional Commits 규칙을 자동으로 검사하여 팀의 표준을 유지합니다.
- **PR 병합 정책**: GitHub 등에서 `Require linear history` 규칙을 활성화하면 병합 커밋 생성을 방지하고, 깔끔한 선형 히스토리를 강제할 수 있습니다. 팀 문화에 맞게 `Squash and merge` 또는 `Rebase and merge` 방식을 선택합니다.
- **자동 변경 로그 생성**: `conventional-changelog`나 `Release Drafter` 같은 도구는 정리된 커밋 히스토리를 기반으로 릴리스 노트를 자동 생성하는 데 매우 효과적입니다.

---

## 보안 및 감사 추적 관점에서의 고려사항

리베이스는 **히스토리를 재작성**하는 행위입니다. 따라서 다음 사항에 주의해야 합니다.
- **배포된/공개된 커밋 히스토리**: 이미 태그가 붙어 릴리스되었거나, 공개 저장소에 푸시되어 다른 사람이 참조하고 있을 가능성이 있는 커밋은 리베이스로 정리해서는 안 됩니다. 이러한 경우에는 새로운 커밋을 추가하는 방식으로 수정하세요.
- **감사 추적이 필요한 환경**: 금융, 의료 등 규제가 엄격한 환경에서는 커밋 히스토리의 불변성(Immutability)이 중요할 수 있습니다. **개인 작업 브랜치에서만 리베이스를 수행**하고, 메인 브랜치(`main`, `develop`)에 병합된 후에는 히스토리를 변경하지 않는 정책을 고려하세요.

---

## 실패 시 복구 방법: `git reflog`

리베이스 중 문제가 발생하거나 결과가 마음에 들지 않을 때, `git reflog`는 강력한 안전망 역할을 합니다.
```bash
# HEAD와 브랜치가 이동했던 모든 기록 확인
git reflog
# 출력 예: 'HEAD@{5}: rebase -i (start): checkout origin/main'

# 원하는 시점(예: HEAD@{3})으로 완전히 되돌리기
git reset --hard HEAD@{3}

# 또는, 해당 시점의 상태를 기반으로 새 브랜치를 생성하여 내용을 살리기
git checkout -b rescue-branch HEAD@{5}
```
`reflog`는 대략 90일 동안의 참조 로그를 보관합니다. 중요한 정리 작업 전에 **임시 브랜치를 하나 생성해 두는 것**도 심리적 안정감과 실수 방지에 도움이 됩니다.

---

## 실용 예제 컬렉션

### 예제 1: 파일 변경 기반으로 대상 커밋 자동 찾기
특정 파일을 수정한 가장 최근 커밋을 자동으로 찾아 그 커밋을 대상으로 `fixup` 커밋을 생성할 수 있습니다.
```bash
# path/to/file 을 마지막으로 수정한 커밋의 해시 찾기
BASE_COMMIT=$(git log -n1 --pretty=format:%h -- path/to/file)
# 해당 커밋을 대상으로 fixup 커밋 생성
git commit --fixup=$BASE_COMMIT
# 범위를 설정하여 리베이스 실행
git rebase -i --autosquash HEAD~10
```

### 예제 2: 여러 개의 독립적인 기능 커밋 정리
서로 다른 두 기능(A와 B)에 대한 수정 사항이 섞여 있을 때, 각 기능의 베이스 커밋 해시를 사용해 정리합니다.
```bash
git commit --fixup=<기능_A_해시>
git commit --fixup=<기능_A_해시>
git commit --fixup=<기능_B_해시>
git commit --squash=<기능_B_해시>
git rebase -i --autosquash HEAD~12
```
리베이스 To-Do 리스트에서 기능 A와 B에 대한 수정 커밋들이 각각의 원본 커밋 아래로 묶여 정리됩니다.

---

## 자주 발생하는 실수와 예방법

| 실수 | 결과 | 예방법 |
|---|---|---|
| `fixup!`/`squash!` 접두어 누락 | 자동 매칭 실패로 정리 안 됨 | `git commit --fixup=<해시>` 사용을 습관화 |
| 리베이스 범위 지나치게 넓음 | 복잡한 충돌 다발 및 관리 어려움 | 기능 단위로 분할하여 정리. `git merge-base` 활용 |
| `git push --force` 사용 | 동료의 작업을 덮어쓸 위험 | **`git push --force-with-lease`** 사용 원칙화 |
| 공유/메인 브랜치에서 리베이스 | 협업자 히스토리 혼란 초래 | **개인 작업 브랜치에서만 리베이스** |
| 서로 무관한 변경 사항을 하나로 스쿼시 | 커밋 의도 불분명, 추적困難 | 주제별 베이스 커밋을 분리하고, 관련 없는 변경은 별도 커밋 유지 |

---

## 팀 차원의 규칙 및 템플릿 제안

효율적이고 안전한 협업을 위해 팀 내에서 다음과 같은 규칙을 정하는 것을 권장합니다.

1.  **작업 범위**: 리베이스 정리는 오직 **개인 작업 브랜치에서만 수행**합니다.
2.  **PR 전 준비 스크립트**: PR 생성 전 커밋 정리를 자동화하는 스크립트를 활용할 수 있습니다.
    ```bash
    # scripts/prepare-pr.sh 예시
    # 1. 이번 PR에 포함될 변경 사항 요약 출력
    git log --oneline origin/main..HEAD
    # 2. 사용자 확인 후, 최근 N개 커밋을 대상으로 인터랙티브 리베이스 실행
    read -p "리베이스할 커밋 수(N)를 입력하세요: " COMMIT_COUNT
    git rebase -i --autosquash HEAD~$COMMIT_COUNT
    # 3. 정리 후 변경 사항 다시 확인
    git log --oneline origin/main..HEAD
    ```
3.  **강제 푸시 정책**: 서버 측 후크(hook)나 CI 정책을 통해 `--force-with-lease` 이외의 강제 푸시를 차단합니다.
4.  **커밋 메시지 컨벤션**: Conventional Commits 규칙을 채택하고, 스쿼시 전 메시지가 명확한지 검토하는 문화를 만듭니다.

---

## 핵심 명령어 요약 (치트 시트)

```bash
### Fixup/Squash 커밋 생성 ###
git commit --fixup=<대상-커밋-해시>
git commit --squash=<대상-커밋-해시>

### 인터랙티브 리베이스 실행 ###
git rebase -i --autosquash HEAD~10                       # 최근 10개 커밋 정리
git rebase -i --autosquash $(git merge-base HEAD origin/main) # 기능 브랜치 전체 정리

### 리베이스 중 충돌 처리 ###
git add <충돌-해결-파일>
git rebase --continue
git rebase --abort  # 취소

### 안전한 원격 푸시 ###
git push --force-with-lease origin <브랜치명>

### 유용한 글로벌 설정 ###
git config --global rebase.autosquash true
git config --global pull.rebase true
git config --global rebase.autoStash true

### 복구 ###
git reflog                    # 과거 HEAD 위치 확인
git reset --hard HEAD@{n}     # 특정 시점으로 완전 복구
```

---

## 결론

`fixup!`/`squash!` 커밋과 `git rebase -i --autosquash`의 조합은 Git을 사용하는 현대적인 소프트웨어 개발 팀에게 필수적인 히스토리 관리 도구입니다. 이 방법을 통해 개발 과정에서 불가피하게 생성된 세부 사항의 커밋들을, 최종적으로는 의도가 명확하고 검토하기 쉬운 단위로 정리할 수 있습니다.

성공적인 적용의 핵심은 **작은 단위로 꾸준히 정리하는 습관**, **안전한 범위 설정과 `--force-with-lease` 사용**, 그리고 **팀 전체가 공유하는 명확한 규칙과 자동화**에 있습니다. 깔끔한 커밋 히스토리는 단순히 보기 좋은 것을 넘어서, 코드 리뷰의 질을 높이고, 버그 추적을 빠르게 하며, 새로운 팀원의 프로젝트 이해를 돕는 등 실질적인 생산성 향상으로 이어집니다.

이 강력한 기능을 두려워하지 말고, 일상적인 개발 워크플로우에 자연스럽게 통합시켜 보세요. 처음에는 작은 범위에서 시작하여 점차 자신감을 쌓아가면, 복잡한 브랜치도 능숙하게 관리할 수 있는 역량을 갖추게 될 것입니다.
