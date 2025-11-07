---
layout: post
title: Git - Rebase + Autosquash
date: 2025-02-26 20:20:23 +0900
category: Git
---
# Git Rebase + Autosquash로 커밋 정리하는 법

## 0) 핵심 요약

- `fixup!`/`squash!` 접두어가 붙은 커밋은 `git rebase -i --autosquash` 실행 시 **자동으로 원 커밋 바로 아래로 이동**되고, `fixup`/`squash` 동작이 **자동 지정**된다.
- `git commit --fixup=<커밋>` / `git commit --squash=<커밋>`으로 **정확한 대상**을 지정할 수 있다(해시, `HEAD~n`, `:<path>` 등).
- 인터랙티브 리베이스 범위를 **충분히 크게** 잡되, **안전하게** 진행(충돌 시 한 단계씩, 실패 시 `reflog`로 복구).
- 리베이스 후 푸시는 **`git push --force-with-lease`** 권장(협업 안전성).
- 반복 흐름을 단축하려면 `rebase.autosquash=true`, `pull.rebase=true` 등을 **글로벌 설정**.

---

## 1) 왜 “커밋 정리”가 필요한가

하나의 기능 동안 잦은 수정으로 **자잘한 커밋**(오타 수정, 콘솔 로그 제거, 네이밍 변경…)이 쌓인다.  
PR 리뷰어, 미래의 나 모두에게 **핵심/맥락이 보이는 이력**이 중요하다.

정리 목표:
- “의도(기능/버그/리팩토링)가 보이는 커밋”만 남긴다.
- **리뷰 단위**와 **릴리스 노트 단위**를 맞춘다.
- 회귀 분석(회귀 버그) 시 **신속한 양자화**가 가능하도록 한다.

---

## 2) `--autosquash` 동작 원리 이해

### 2.1 매칭 규칙
- 커밋 메시지가 **`fixup! <원커밋제목>`** 혹은 **`squash! <원커밋제목>`** 형태면, 인터랙티브 리베이스 시 해당 원 커밋을 찾아 **자동 재배치** 및 **동작 지정**:
  - `fixup!` → `fixup`
  - `squash!` → `squash`
- 동일 제목의 커밋이 여러 개면 **가장 가까운 조상(보통 최신)**에 매칭되는 경향이 있다.  
  → 안전을 위해 **해시 기반**의 `--fixup=<해시>` 사용 권장.

### 2.2 자동 정렬 시점
- `git rebase -i --autosquash <범위>` 실행 시 **To-Do 리스트를 생성할 때** 자동 재배치가 일어난다.
- 이미 열려 있는 To-Do 편집 화면에서 수동으로 바꿔도 되지만, 원칙은 “자동화하고 최소 편집”.

---

## 3) 실전 흐름 — 가장 많이 쓰는 3가지 패턴

### 패턴 A: 단일 기능 정리(소규모)

```bash
# 기능 커밋 1개 + 자잘한 수습 커밋 2개
git commit -m "feat: 게시글 작성 기능"
git commit -m "fixup! feat: 게시글 작성 기능"
git commit -m "squash! feat: 게시글 작성 기능"

# 최근 3개만 대상으로 자동 정리
git rebase -i --autosquash HEAD~3
# 편집 화면 예시(자동 배치):
# pick   a1b2c3 feat: 게시글 작성 기능
# fixup  d4e5f6 fixup! feat: 게시글 작성 기능
# squash e7f8g9 squash! feat: 게시글 작성 기능
# 저장/종료 후 메시지 병합 편집(필요 시)
```

### 패턴 B: 커밋 대상 정확 지정(안전)

```bash
# 대상을 해시로 지정: 메시지 충돌/동명이인 방지
git commit --fixup=<해시>
git commit --squash=<해시>

# 넉넉한 범위로 리베이스(예: 최근 15개)
git rebase -i --autosquash HEAD~15
```

### 패턴 C: 작업 분기 중간 정리(중규모)

```bash
# 기능 개발 진행 중, 이전 기능 커밋들을 정리
# 작업 변경은 계속 커밋하되, fixup 커밋으로 누적
git commit --fixup=<기능 베이스 커밋 해시>
git commit --fixup=<기능 베이스 커밋 해시>
git commit --squash=<기능 베이스 커밋 해시>

# 기능 흐름이 한눈에 보이도록 대역폭 넓게 리베이스
git rebase -i --autosquash origin/main
# 또는 HEAD~N 형태로도 가능
```

---

## 4) `git commit --fixup/--squash` 정확히 알기

```bash
# 가장 안전한 형태: 커밋 해시로 타깃 지정
git commit --fixup=<커밋해시>
git commit --squash=<커밋해시>

# 메시지 매칭도 가능(권장도 낮음: 중복 위험)
git commit --fixup="feat: 게시글 작성 기능"
```

- `--fixup`은 원 커밋 **메시지 유지**(새 메시지 폐기).
- `--squash`는 메시지 병합 편집 과정을 열어 **설명 가다듬기**에 유리.

> 참고: 최신 Git에서는 `--fixup=(amend|reword):<커밋>` 같은 확장 옵션(*버전 의존*)도 제공한다. **팀/환경의 Git 버전**에 따라 사용 가능 여부를 확인하고, 문서화해 혼선을 줄인다.

---

## 5) 인터랙티브 리베이스 범위 잡기 전략

- **작을수록 안전**: `HEAD~3`, `HEAD~5` 처럼 최근만.  
- **중간 규모**: 기능 브랜치 전체를 정리할 땐 `git merge-base`를 활용.

```bash
# 기능 브랜치에서 main과의 공통 조상 찾기
BASE=$(git merge-base HEAD origin/main)
git rebase -i --autosquash $BASE
```

- **PR 기준 정리**: “이번 PR에 포함되는 범위만” 대상으로.

---

## 6) 충돌이 날 때 — 표준 수습 루틴

```bash
git rebase -i --autosquash HEAD~20
# ... 충돌!

# 1) 충돌 파일 수정
# 2) 스테이징
git add <수정한 파일들>

# 3) 계속
git rebase --continue

# 포기(전부 되돌리기)
git rebase --abort
```

- 충돌이 **여러 커밋에서 연속**으로 터질 수 있다(리베이스는 커밋 단위 재적용).  
- 커다란 충돌이 계속 발생한다면 범위를 줄이거나 **단위별로 분할**해 처리한다.

---

## 7) 리베이스 이후 푸시 — 안전 수칙

- 리베이스는 커밋 해시를 **재작성**하므로 원격 대비 이력이 달라진다.
- 푸시는 **`--force-with-lease`**를 사용:

```bash
git push --force-with-lease origin feature/my-feature
```

- 단순 `--force`는 팀원의 새로운 작업을 덮어쓸 위험이 크다.  
- CI/브랜치 보호 규칙(Require linear history)와 병행하여 **혼선 최소화**.

---

## 8) 설정으로 반복 작업 줄이기

```bash
# 8.1 리베이스 때 autosquash 기본 활성화
git config --global rebase.autosquash true

# 8.2 git pull의 기본 동작을 rebase로
git config --global pull.rebase true

# 8.3 rebase 시 변경파일을 임시 저장/복구(충돌 회피에 도움)
git config --global rebase.autoStash true

# 8.4 push할 때 upstream이 다르면 안전 확인
git config --global push.default simple
```

**TIP**: 팀 공통 개발 컨테이너/DevBox에서 이 설정을 **스캐폴딩 스크립트**로 배포.

---

## 9) 대규모 PR 정리 “레시피”

### 9.1 Conventional Commits와 함께 쓰기
- `feat:`, `fix:`, `refactor:` 단위로 **의도별 커밋 묶기**.
- 자잘한 수정은 **`--fixup`** 커밋으로 밀어 넣고 `squash`/`fixup` 처리.

### 9.2 타깃 베이스 커밋 고정
- 정리 대상이 되는 “대표 커밋(핵심 기능)”을 **해시로 고정**하고 여기에 계속 `--fixup`을 달아준다.

```bash
# 핵심 기능 커밋 해시가 FEA1234라고 가정
git commit --fixup=FEA1234
git commit --fixup=FEA1234
git commit --squash=FEA1234
git rebase -i --autosquash HEAD~20
```

### 9.3 스텝 바이 스텝
- “한 번에 50개 커밋 정리”는 리스크가 크다.  
- 기능별/폴더별로 나누어 **여러 번에 걸쳐** 정리하고 푸시.

---

## 10) 팀/CI와의 연계

- **Commitlint**로 메시지 규칙 강제(Conventional Commits).  
- **PR 체크**에 `Require linear history`를 켜서 merge commit 제한.  
- **`Squash and merge` vs `Rebase and merge`**: 조직 문화에 맞게 선택.  
- 리베이스/스쿼시 후 **자동 changelog 생성**(conventional-changelog/Release Drafter)과 잘 맞는다.

---

## 11) 보안/감사 관점에서의 주의

- 리베이스는 **역사 재작성**이므로, 이미 배포/공개된 이력에 남용 금지.  
- 릴리스 태그 이후에는 커밋 정리가 아닌 **새 커밋 추가**가 원칙.  
- 감사 추적이 필요한 저장소에서는 **개인 브랜치에서만 정리**하고, 메인 이력은 가능한 **불변**으로 유지.

---

## 12) 실패 복구(꼭 알아두기)

```bash
# 리베이스로 이력이 꼬였다면…
git reflog                 # 예: HEAD@{3}이 리베이스 전 상태
git reset --hard HEAD@{3}  # 되돌리기

# 일부만 살릴 때(새 브랜치 파생)
git checkout -b rescue HEAD@{3}
```

- `reflog`는 **브랜치/HEAD 이동의 히스토리**를 기억한다.  
- 과감한 정리 전에 **임시 브랜치**를 하나 만들어 두면 심리적 안정도↑.

---

## 13) 예제 컬렉션

### 13.1 “메시지”로 fixup 대상 지정(동명이인 위험 있음)

```bash
git commit --fixup="feat: 게시글 작성 기능"
git rebase -i --autosquash HEAD~5
```

- 같은 제목이 여럿이면 매칭이 원하는 결과가 아닐 수 있다.  
  → 안전을 위해 **해시 사용** 권장.

### 13.2 여러 베이스 커밋을 각자 정리

```bash
# 기능 A, 기능 B가 뒤섞여 있는 이력
git commit --fixup=<A의 해시>
git commit --fixup=<A의 해시>
git commit --fixup=<B의 해시>
git commit --squash=<B의 해시>

# 최근 12개 정리
git rebase -i --autosquash HEAD~12
```

- To-Do 리스트에서 A/B 블록이 각자 묶여 정리된다.

### 13.3 파일 단위 대상을 빠르게 찾기

```bash
# 이 파일에 마지막으로 영향을 준 커밋을 베이스로 잡아 fixup
BASE=$(git log -n1 --pretty=format:%h -- path/to/file)
git commit --fixup=$BASE
git rebase -i --autosquash HEAD~10
```

### 13.4 리베이스 중 충돌 다발 시 분할

```bash
# 반씩 쪼개서 두 번에 나눠 정리
git rebase -i --autosquash HEAD~10
# 끝난 뒤
git rebase -i --autosquash HEAD~10
```

---

## 14) 자주 하는 실수와 방지 요령

| 실수 | 결과 | 방지 |
|---|---|---|
| 메시지 앞 접두어 누락(`fixup!`/`squash!`) | 자동 매칭 안 됨 | `git commit --fixup/--squash=<해시>` 사용 |
| 범위 과도하게 큼 | 충돌 폭증/스트레스 | 기능 단위로 분할, `merge-base` 활용 |
| `git push --force` 남용 | 팀 작업 덮어쓰기 | `--force-with-lease` 사용 |
| 공개 브랜치 리베이스 | 이력 혼선/감사 취약 | 개인 브랜치에서만 정리, 메인은 불변 |
| 서로 다른 주제 섞어 스쿼시 | 이력 의미 상실 | 주제별로 베이스 커밋 분리, 스쿼시도 분리 |

---

## 15) 팀 규칙(권장 템플릿)

- 개인 브랜치에서만 `rebase -i --autosquash` 사용.  
- PR 전 “정리 스크립트” 실행:
  ```bash
  # scripts/prepare-pr.sh (예시)
  # 1) 변경 요약 출력
  git log --oneline origin/main..HEAD
  # 2) 최근 20개 대상으로 자동 스쿼시(사용자 편집 허용)
  git rebase -i --autosquash HEAD~20
  # 3) 마지막 확인
  git log --oneline origin/main..HEAD
  ```
- 푸시는 `--force-with-lease`만 허용(서버 hook/CI 정책으로 강제).
- 메시지 규칙: Conventional Commits + 스쿼시 전 메시지 점검.

---

## 16) 참고 명령 치트시트

```bash
# fixup/squash 생성
git commit --fixup=<hash>
git commit --squash=<hash>

# 인터랙티브 리베이스 + 자동 스쿼시
git rebase -i --autosquash HEAD~N
git rebase -i --autosquash $(git merge-base HEAD origin/main)

# 충돌 흐름
git add <files>
git rebase --continue
git rebase --abort

# 푸시(안전)
git push --force-with-lease

# 설정
git config --global rebase.autosquash true
git config --global pull.rebase true
git config --global rebase.autoStash true

# 복구
git reflog
git reset --hard HEAD@{n}
```

---

## 17) 마무리

- `fixup!`/`squash!` + `--autosquash`는 **정리의 표준 루틴**이다.  
- 작은 단위로, 안전한 범위에서, 팀 규칙과 CI를 곁들여 **항상 같은 방식으로 반복**하라.  
- 깔끔한 히스토리는 **리뷰 품질**을 높이고, **버그 분석**을 빠르게 하고, **온보딩 비용**을 줄인다.

---

## 참고 자료

- Git 공식 문서: Rebase — https://git-scm.com/docs/git-rebase  
- Pro Git Book: Interactive Rebase — https://git-scm.com/book/en/v2/Git-Tools-Rewriting-History  
- `--autosquash` 설명 — https://git-scm.com/docs/git-rebase#Documentation/git-rebase.txt---autosquash