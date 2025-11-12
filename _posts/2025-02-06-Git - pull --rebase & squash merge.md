---
layout: post
title: Git - pull --rebase & squash merge
date: 2025-02-06 19:20:23 +0900
category: Git
---
# `git pull --rebase` 완벽 가이드 + GitHub `Squash and merge` 심화

## 0. 빠른 개요

- **`git pull --rebase`**: 원격의 최신 커밋을 가져온 뒤, **내 로컬 커밋들을 ‘꺼냈다가’ 최신 위로 다시 재생성**해 **선형 이력**을 만든다.  
- **GitHub `Squash and merge`**: PR 내 **여러 커밋을 하나의 커밋**으로 압축(squash)하여 병합한다.  
- 공통 목표: **히스토리 단순화**. 단, **디버깅/리그레션 추적의 단서가 줄어드는 대가**가 있다.

---

# 1. `git pull --rebase`: 깔끔한 로컬 병합

## 1.1 개념 복습

```bash
git pull --rebase origin main
```

동작 순서:
1. 원격 `origin/main` 최신 커밋을 가져온다(fetch).
2. 내 로컬 커밋을 **임시로 분리**한다.
3. 원격 최신 커밋 위에 **내 커밋을 순서대로 재적용**한다.
4. 충돌이 없으면 **선형(linear) 이력**이 된다.

### 일반 `pull` vs `pull --rebase` 비교

| 항목 | `git pull`(merge) | `git pull --rebase` |
|---|---|---|
| 병합 방식 | merge 커밋 생성 | rebase로 재배치 |
| 히스토리 | 분기/병합 흔적 유지 | 선형으로 간결 |
| 충돌 빈도 | 한 지점 | 커밋 단위로 반복 가능 |
| 협업 난이도 | 낮음 | 주의 필요(해시 재작성) |

---

## 1.2 실전 옵션 — 모드, 자동 스태시, 반복 충돌 학습

### A) `--rebase`의 모드
```bash
git pull --rebase=merges        # merge 토폴로지를 보존하며 rebase
git pull --rebase=interactive   # 대화형(reword/squash/fixup 등)
git pull --rebase               # true와 동일(선형 이력 목적)
git pull --rebase=false         # 평소처럼 merge
```
- 팀에 **선형 이력** 규칙이 있다면 `true` 또는 `merges`를 조직 표준으로.

### B) 변경 보관: `--autostash` / 자동 설정
```bash
git pull --rebase --autostash
git config --global rebase.autoStash true
```
- 워킹 디렉터리에 미커밋 변경이 있을 때 자동으로 stash/복원.

### C) 반복 충돌 자동 해결 힌트: `rerere`
```bash
git config --global rerere.enabled true
```
- 이전에 해결한 **동일 패턴의 충돌**을 학습하여 재적용(검토 후 커밋).

---

## 1.3 충돌 처리 루틴(필수 루프)

```bash
git pull --rebase
# 충돌 발생 시:
# 1. 파일 열어 충돌 마커(<<<<<<<, =======, >>>>>>>) 해결
git add <수정파일>
git rebase --continue

# 현재 커밋 건너뛰기
git rebase --skip

# 전체 중단(시작 전으로 복귀)
git rebase --abort
```

### 파일 단위 빠른 선택(전체 채택)
```bash
git checkout --ours   path/to/file   # 현재 브랜치 쪽 선택
git checkout --theirs path/to/file   # 원격(적용 대상) 쪽 선택
git add path/to/file
```

> GUI 선호 시:
> ```bash
> git config --global merge.tool meld   # 또는 kdiff3, vscode 등
> git mergetool
> ```

---

## 1.4 재현 가능한 실습(로컬에서 그대로 따라하기)

```bash
# 실습 레포 초기화
rm -rf pull-rebase-lab && mkdir pull-rebase-lab && cd pull-rebase-lab
git init
git config user.name  "lab"
git config user.email "lab@example.com"
git branch -M main

echo "line 1" > app.txt
git add . && git commit -m "init: line 1"
git remote add origin https://example.com/your/repo.git    # 예시 URL(실사용 시 교체)

# 원격 시뮬레이션 없이도 체험:
# main에서 커밋 추가(원격 최신이라고 가정)
git checkout -b temp
echo "upstream 2" >> app.txt
git commit -am "feat: upstream 2"
git checkout main
git merge --ff-only temp

# 로컬에서 작업 커밋 만들어 충돌 유도
git checkout -b feature/line
echo "local 2" >> app.txt
git commit -am "feat: local 2"

# 이제 "원격 최신(main)" 위로 내 커밋 재배치
git checkout feature/line
git rebase main   # 충돌이 나도록 일부러 같은 줄을 수정해도 됨
# 충돌 해결 → add → rebase --continue
```

---

## 1.5 기본/전역 설정(매번 타이핑 방지)

```bash
# pull 기본을 rebase로
git config --global pull.rebase true

# pull 시 자동 스태시
git config --global rebase.autoStash true

# 브랜치 생성 시 자동 추적+rebase
git config --global branch.autosetuprebase always
```

> 팀/레포 단위 설정은 `--global` 대신 **레포 로컬**에서 동일 명령을 실행.

---

## 1.6 CI·보호 브랜치·협업 규칙과의 상호작용

- **보호 브랜치(Protected Branch)**: Required status checks(테스트/린트/빌드), Required reviews, Linear history 등이 설정되어 있으면 **rebase 후 강제 푸시가 제한**될 수 있음.  
- PR 워크플로우:
  - 선형 이력을 강제하는 팀: 로컬에서 `git pull --rebase` → 충돌 해결 → push
  - rebase 후 원격에 올릴 때는 **`git push --force-with-lease`** 사용(동료 커밋 보호).
- CI 실패 시: 로컬에서 테스트/포맷 확인 후 재푸시.

---

## 1.7 장단점·리스크·회피법

| 항목 | 장점 | 단점/리스크 | 회피/보완 |
|---|---|---|---|
| 이력 | 선형/간결 | 맥락(분기 흔적) 약화 | PR 설명·range-diff로 보완 |
| 협업 | 로컬 정리 유리 | 공유 브랜치 rebase 위험 | **개인 브랜치**에서만 rebase |
| 충돌 | 반복 해결에 유리(rerere) | 커밋마다 충돌 가능 | rerere/mergetool/ours-theirs |
| 복구 | reflog/ORIG_HEAD | 실수 시 손상 가능 | 백업 태그/브랜치, `--abort` 습관 |

복구 비상키:
```bash
git reflog
git reset --hard ORIG_HEAD
git switch -c rescue HEAD@{2}
```

---

# 2. GitHub `Squash and merge` 심화

## 2.1 개념 복습
- PR의 **모든 커밋을 단 하나의 커밋으로 압축**해 base 브랜치에 병합.
- 결과: **히스토리 단순화**(기능 단위 1커밋).  
- 대가: **세부 커밋의 추적성**은 줄어듦(디버깅 시 granularity 감소).

---

## 2.2 실전 흐름

1) PR에서 **Squash and merge** 선택  
2) GitHub가 **커밋 메시지 편집 창**을 제공  
3) 메시지를 기능 단위로 정리  
4) **Confirm squash and merge** 클릭 → base에 **단일 커밋** 생성

예시(정리 전):

```
feat: 로그인 폼 생성
fix: 버튼 스타일 수정
refactor: 네이밍 변경
```

정리 후(최종 커밋 메시지):

```
feat(auth): 로그인 화면 구현
- 로그인 폼 생성
- 버튼 스타일 조정
- 필드 네이밍 정리
```

---

## 2.3 메시지 규칙·트레일러(Co-authored-by, Signed-off-by)

- 다수 기여자를 **Co-authored-by** 트레일러로 유지:
```
Co-authored-by: Alice <alice@example.com>
Co-authored-by: Bob <bob@example.com>
```

- DCO 정책 사용 시 **Signed-off-by** 유지:
```
Signed-off-by: Your Name <you@example.com>
```

- 컨벤션(Conventional Commits) 예:
```
feat(ui): add login page

- email/password fields
- validation and tests

Co-authored-by: Alice <alice@example.com>
Signed-off-by: You <you@example.com>
```

> PR 템플릿/CI 체크로 메시지 포맷을 표준화하면 품질이 올라간다.

---

## 2.4 장단점·선택 기준

| 항목 | 장점 | 단점 |
|---|---|---|
| 히스토리 | 매우 깔끔 | 세부 커밋의 맥락 손실 |
| 코드리뷰 | 기능 단위로 병합 | bisect·세밀 회귀 추적 어려움 |
| 기여도 | Co-authored-by로 보존 가능 | 커밋 단위의 구분 사라짐 |

**권장 상황**
- 개인/작은 팀, 기능 단위 릴리즈가 중요한 저장소
- 오픈소스에서 PR을 기능 단위로 관리하고 싶은 경우
- 리뷰 과정에서 잔 커밋(fixup, nit)을 정리하고 싶은 경우

**피하는 편이 좋은 상황**
- **bisect**로 세밀 추적이 빈번한 핵심 저장소
- 규정상 커밋 단위 증적이 필요한 조직

---

## 2.5 충돌과 보호 브랜치 정책

- Squash and merge 전에 **PR 충돌이 없어야** 병합 버튼이 활성화된다.  
- 보호 브랜치의 **Required status checks**, **Required reviews**, **Linear history** 등에 따라 병합이 **보류/거부**될 수 있다.  
- 충돌이 있으면 웹 Resolve conflicts 또는 **로컬에서 해결 후 푸시**(PR 업데이트) → 체크 통과 후 squash.

---

## 2.6 CLI/자동화(옵션)

- GitHub CLI:
```bash
# Squash merge
gh pr merge <PR_NUMBER> --squash --admin
# 메시지를 자동 생성/편집하려면 --body 옵션 병행
```

- 병합 전략 제어는 저장소 설정에서 허용/차단 가능  
  (Create a merge commit / Squash and merge / Rebase and merge).

---

## 2.7 재현 가능한 PR 시뮬레이션(로컬 → GitHub)

```bash
# 1. 로컬 새 레포
rm -rf squash-lab && mkdir squash-lab && cd squash-lab
git init
git config user.name  "lab"
git config user.email "lab@example.com"
git branch -M main
echo "v1" > README.md
git add . && git commit -m "chore: init"

# 2. 기능 브랜치 커밋 여러 개
git checkout -b feature/login
echo "- login form" >> README.md
git commit -am "feat: login form"
echo "- button style" >> README.md
git commit -am "fix: button style"
echo "- rename vars" >> README.md
git commit -am "refactor: rename vars"

# 3. 원격 생성 및 푸시(실제 사용 시 본인 리포로 교체)
#   gh repo create <owner>/<name> --public --source=. --push
#   gh pr create --fill --base main --head feature/login

# 4. PR 화면에서 Squash and merge 선택 →
#    최종 메시지 정리 → Confirm.
```

---

# 3. 종합 체크리스트

### A. `git pull --rebase`
- [ ] 개인 브랜치에서만 사용(공유 브랜치 rebase 금지)
- [ ] `pull.rebase=true`, `rebase.autoStash=true` 설정
- [ ] 충돌 시 `edit → add → rebase --continue`
- [ ] 반복 충돌은 `rerere.enabled=true`
- [ ] 문제 발생 시 `reflog`, `ORIG_HEAD` 로 복구
- [ ] 푸시는 `--force-with-lease`

### B. `Squash and merge`
- [ ] 메시지 정리(Conventional Commits 권장)
- [ ] Co-authored-by, Signed-off-by 트레일러 유지
- [ ] 보호 브랜치 정책/CI 체크 통과 확인
- [ ] 충돌은 웹/로컬에서 해소 후 병합
- [ ] bisect/세밀 추적 필요 저장소라면 신중히

---

# 4. 명령어 요약

```bash
# pull --rebase 기본
git pull --rebase
git pull --rebase=merges
git pull --rebase=interactive
git pull --rebase --autostash

# 설정
git config --global pull.rebase true
git config --global rebase.autoStash true
git config --global rerere.enabled true
git config --global branch.autosetuprebase always

# 충돌 루틴
git status
git checkout --ours   <path>
git checkout --theirs <path>
git mergetool
git add <path>
git rebase --continue | --skip | --abort

# 복구 비상키
git reflog
git reset --hard ORIG_HEAD
git switch -c rescue HEAD@{N}
```

---

## 5. 결론

- **`git pull --rebase`** 는 로컬 협업의 마찰을 줄이고 **선형 이력**을 만든다.  
- **`Squash and merge`** 는 PR을 **기능 단위 1커밋**으로 정리해 읽기 쉬운 히스토리를 만든다.  
- 두 전략 모두 “깔끔한 히스토리”라는 공통 목표를 가지지만, **세부 추적성 감소**라는 대가가 있다.  
- 팀의 **보호 브랜치 정책/CI/리뷰 문화**와 맞춰, 언제 **rebase**를 쓰고 언제 **squash**를 쓰는지 **명확한 기준**을 문서화하면 가장 강력하다.

---

## 참고
- Git Docs — `git pull` (`--rebase`)  
  https://git-scm.com/docs/git-pull#Documentation/git-pull.txt---rebase  
- GitHub Docs — Merge a PR (Squash and merge)  
  https://docs.github.com/en/pull-requests/collaborating-with-pull-requests/merging-a-pull-request#squash-and-merge