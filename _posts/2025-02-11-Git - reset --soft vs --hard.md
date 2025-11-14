---
layout: post
title: Git - reset --soft vs --hard
date: 2025-02-11 19:20:23 +0900
category: Git
---
# `git reset --soft` vs `git reset --hard`

## 개념 리마스터: Git은 무엇을 “되돌리는가” — 3계층 상태 모델

Git 작업 상태는 크게 **세 층**으로 이해하면 명확하다.

1. **HEAD**: 현재 브랜치가 가리키는 **커밋 스냅샷** (참조 포인터)
2. **Index(= Staging Area)**: 다음 커밋에 포함될 **파일 스냅샷** (준비 영역)
3. **Working Directory(WD)**: 실제 디스크의 **작업 파일들**

리셋은 이 세 층 중 **어디까지 과거 스냅샷으로 동기화할지**를 결정한다.

수학적 관점으로 간단히 표현하면, 특정 커밋 \(C\)를 목표로 할 때:

$$
\mathrm{reset\_soft}(C):\quad HEAD \leftarrow C
$$

$$
\mathrm{reset\_mixed}(C):\quad HEAD \leftarrow C,\;\; INDEX \leftarrow \mathrm{Tree}(C)
$$

$$
\mathrm{reset\_hard}(C):\quad HEAD \leftarrow C,\;\; INDEX \leftarrow \mathrm{Tree}(C),\;\; WD \leftarrow \mathrm{Tree}(C)
$$

여기서 \(\mathrm{Tree}(C)\)는 커밋 \(C\)의 트리(파일 스냅샷)를 의미한다.

---

## 옵션별 비교(정확표)

| 옵션 | HEAD 이동 | Index 초기화 | WD 롤백 | 핵심 효과 |
|---|---|---|---|---|
| `--soft` | O | X | X | **커밋만 취소**, 현재 변경은 그대로 유지(대개 스테이징 유지) |
| `--mixed` (기본) | O | O | X | 커밋 취소 + **스테이징 해제**, 파일은 그대로 |
| `--hard` | O | O | O | 커밋/스테이징/파일 모두 과거 스냅샷로 **강제** 동기화 |

주의: `--hard`는 **추적(tracked) 파일**의 변경을 버린다. **미추적(untracked) 파일/디렉토리**는 기본적으로 삭제되지 않는다(그건 `git clean -fd` 영역). 단, 무시(.gitignore) 규칙과 조합 시 동작을 혼동하지 않도록 주의한다.

---

## 기본 예제(초안 보강)

### 실험 세팅

```bash
rm -rf lab-reset && mkdir lab-reset && cd lab-reset
git init
git config user.name "lab" && git config user.email "lab@example.com"

echo "v1" > hello.txt
git add hello.txt
git commit -m "v1 커밋"

echo "v2" > hello.txt
git add hello.txt
git commit -m "v2 커밋"

echo "v3" > hello.txt
git add hello.txt
git commit -m "v3 커밋"
```

현재 `hello.txt` 내용은 `v3`.

### `--soft`

```bash
git reset --soft HEAD~1
```
- HEAD만 `v2` 커밋으로 이동.
- Index와 WD에는 `v3` 스냅샷이 **그대로** 남아 있으므로, 즉시 재커밋 가능:
```bash
git commit -m "v3 메시지 수정 또는 추가 변경 포함"
```

### `--mixed`(기본)

```bash
git reset HEAD~1
# 또는

git reset --mixed HEAD~1
```
- HEAD는 `v2`로 이동, Index는 `v2`로 동기화되어 `v3` 변경이 **스테이징 해제**됨.
- WD 파일은 아직 `v3` 상태이므로, `git add` 하면 다시 커밋 가능.

### `--hard`

```bash
git reset --hard HEAD~1
```
- HEAD/Index/WD 모두 `v2` 스냅샷로 동기화.
- `v3`에서 했던 변경은 **추적 파일 기준**으로 작업 디렉토리에서도 사라짐.

---

## 실무 자주 쓰는 패턴

### 커밋 메시지/내용을 즉시 손보고 재커밋(안전)

- 방금 한 커밋을 풀어 커밋만 되돌리되, 변경물은 그대로 두고 싶을 때:
```bash
git reset --soft HEAD~1
# 필요한 수정/파일 추가

git commit -m "정돈된 메시지로 재커밋"
```
대안: 바로 이전 커밋이라면 `git commit --amend` 도 가능(커밋 해시가 바뀌므로 로컬 전용).

### “스테이징만” 풀기(파일은 그대로)

```bash
git reset HEAD -- path/to/file   # 해당 파일만 스테이징 해제
git reset                        # 전체 스테이징 해제(== --mixed HEAD)
```

### “파일만” 이전 커밋 상태로 복원(부분 리셋)

```bash
git checkout <commit> -- path/to/file
# Git 2.23+이면

git restore --source=<commit> --worktree --staged path/to/file
```
- 커밋 단위 전체 롤백이 아니라 **특정 파일만** 과거 상태로 복원하는 안전한 방법.

### 완전 초기화(테스트 파손 정리)

```bash
git reset --hard
# + 미추적 파일/디렉토리 제거까지 필요하면(주의!):

git clean -fd
```

---

## 부분 리셋(pathspec) — `git reset` 으로 특정 파일만 Index 동기화

```bash
# 특정 파일만 스테이징을 “과거 커밋(C)” 스냅샷으로 되돌림(파일 내용은 그대로)

git reset <C> -- path/to/file
```

- WD는 손대지 않고, **Index만** 바꾼다. 이후 `git checkout -- path` 또는 `git restore` 로 WD를 동기화할 수도 있다.

---

## 협업에서의 안전 수칙 — 언제 reset을 쓰고, 언제 쓰지 말까

- `--soft`/`--mixed` 는 **로컬 브랜치 정리**에 유용. 푸시 전 커밋 구조 정돈 목적.
- `--hard` 는 **로컬의 망가진 워킹 트리/빌드 산출물 정리** 등 “청소” 용도. 하지만 **협업 브랜치에서 이력 변경 + 강제 푸시**는 사고의 지름길.
- 이미 원격에 공유된 이력을 “무효화”하려면 **`git revert`** 로 취소 커밋을 쌓는 방식이 안전하다.
- 불가피하게 이력 재작성 후 푸시가 필요하다면:
```bash
git push --force-with-lease
```
- 보호 브랜치(Require linear history / Require status checks / Require PR reviews)로 **사용 자체를 통제**하라.

---

## `reset` 과 친척 명령 비교

| 명령 | 목적/특징 | 안전성 | 주 사용처 |
|---|---|---|---|
| `git reset --soft` | 커밋만 취소(변경물 유지) | 높음 | 커밋 메시지/단위 정리 |
| `git reset --mixed` | 커밋 취소 + 스테이징 해제 | 높음 | 커밋 전 단계로 되돌려 재구성 |
| `git reset --hard` | 커밋/Index/WD 강제 동기화 | 낮음 | 테스트/빌드 잔재 일괄 폐기 |
| `git revert` | “취소 커밋” 생성(이력 보존) | 매우 높음 | 공유 브랜치 취소 작업 |
| `git checkout -- file` | 파일만 과거 스냅샷 복원(구식) | 보통 | 부분 복원 |
| `git restore` | checkout 역할 분해(파일 복원) | 높음 | Git 2.23+ 권장 |
| `git clean -fd` | 미추적 파일/폴더 삭제 | 위험 | 잔재/산출물 제거 |

---

## 복구 관점: reset 실수 이후 되돌리기

- `--hard` 로 잃은 변경(추적 파일)은 대개 **`git reflog`** 로 과거 HEAD를 찾아 복귀 가능(오래되면 GC로 소멸).
- 프로시저:
```bash
git reflog
git reset --hard HEAD@{N}     # 실수 직전으로 복귀
# 또는 안전하게

git checkout -b rescue HEAD@{N}
```
- `untracked` 삭제는 reflog로 복구 불가. 미추적은 `git clean` 전에 백업 스크립트를 습관화하라.

---

## rebase/merge/cherry-pick 와 reset의 조합

- **커밋 구조 정리**: `--soft` 로 여러 커밋 풀어내고 파일/메시지 정돈 → 단일 커밋으로 재커밋(간이 squash).
- **충돌 해소 반복**: rebase 충돌이 빈번하다면 `reset --hard` 로 워킹을 깨끗이 하고, 브랜치 전략 자체를 점검(작은 PR, 더 잦은 동기화).
- **실험 브랜치 정리**: 실험 결과만 남기고 싶다면 `--mixed` 로 스테이징만 해제 → 의미 있는 파일만 선택 커밋.

---

## 고급 고려사항

### Submodule

- `reset` 은 상위 저장소의 **gitlink** 를 되돌린다. 실제 서브모듈 워킹 디렉토리 상태 동기화는 추가 작업 필요:
```bash
git submodule update --init --recursive
```

### Sparse Checkout / Worktree

- 스파스 체크아웃이나 다중 워크트리에서 `--hard` 는 **현재 WD에 보이는 경로**에만 영향. 숨은 경로/다른 워크트리는 별개로 관리됨.

### Hooks / CI

- 로컬에서 reset 후 푸시하면 서버측 Hooks/CI가 예상치 못한 이력 재작성으로 동작이 꼬일 수 있다. 보호 브랜치/승인 규칙으로 방지.

---

## 실전 시나리오 모음

### “방금 커밋 메시지 잘못 썼다”

```bash
git reset --soft HEAD~1
# 메시지 정정/파일 추가

git commit -m "feat: 로그인 폼 도입 및 기본 검증"
```

### “커밋은 유지하되 스테이징만 다 풀고 싶다”

```bash
git reset
# 또는

git reset --mixed
```

### “테스트 하다 워킹 디렉토리 난장판”

```bash
git reset --hard
# 미추적까지 제거하려면(주의!)

git clean -fd
```

### “특정 파일만 어제 커밋으로 되돌리고 싶다”

```bash
git checkout <yesterday-commit> -- src/foo.ts
# 또는

git restore --source=<yesterday-commit> src/foo.ts
```

### “로컬에서 커밋 구조 정리 후 올리고 싶다”

```bash
# 여러 커밋을 풀어 하나로 뭉치기(간단 버전)

git reset --soft HEAD~3
git commit -m "feat: 결제 취소 흐름 추가(가이드/로그 포함)"
git push --force-with-lease
```
주의: 공유 브랜치에서는 rebase/force-push 자체를 팀 규칙으로 제어하라.

---

## 작은 랩: 상태 관찰 명령으로 이해하기

```bash
# 파일/스테이징/HEAD 관계 관찰

git status
git diff                 # WD vs Index
git diff --cached        # Index vs HEAD
git show HEAD:hello.txt  # HEAD 스냅샷 확인
```

- `reset --mixed HEAD~1` 이후:
  - `git diff` 에는 WD와 Index 차이(= 방금 커밋했던 내용)가 보인다.
  - `git diff --cached` 는 비어 있을 가능성이 크다(스테이징 해제).

---

## FAQ

### Q1. `--hard` 는 미추적 파일도 지우나?

- 아니오. `--hard` 는 **추적 파일**을 HEAD 스냅샷에 맞춘다. 미추적은 `git clean` 대상.

### Q2. `reflog` 가 항상 구원해주나?

- **아니다.** 오래된 dangling 객체는 GC로 사라질 수 있다. 큰 변경 전에는 **백업 브랜치**를 만들어두는 습관이 최선:
```bash
git switch -c backup/$(date +%F-%H%M)
```

### Q3. reset 대신 안전하게 과거 상태로 “되돌린” 기록을 남기고 싶다

- `git revert` 사용. 취소 커밋이 남아 이력이 보존되므로 협업에 안전.

---

## 실무 체크리스트

- 대규모 정리 전:
  - `git branch backup/<ts>` 로 백업 브랜치 생성
  - `git log --oneline --graph --decorate --all` 로 이력 시각화
- 협업 브랜치:
  - reset/rebase 후 푸시는 원칙적으로 금지, 필요시 `--force-with-lease`
  - 보호 브랜치, 필수 리뷰/체크, 선형 이력 강제
- 정리/청소:
  - 추적 파일 정리: `reset --hard`
  - 미추적 산출물 정리: `clean -fd` (주의)
- 복구:
  - `reflog` → `reset --hard HEAD@{N}` 또는 `checkout -b rescue HEAD@{N}`

---

## 명령 요약(치트시트)

```bash
# 커밋만 되돌리기(변경물 유지)

git reset --soft HEAD~1

# 커밋 되돌리고 스테이징 해제(파일은 그대로)

git reset --mixed HEAD~1
git reset HEAD               # 전체 언스테이징
git reset HEAD -- path/file  # 부분 언스테이징

# 커밋/스테이징/파일 모두 과거로

git reset --hard HEAD~1

# 파일만 과거 커밋으로 복원

git checkout <C> -- path
# 또는 (Git 2.23+)

git restore --source=<C> --worktree path
git restore --source=<C> --staged path

# 미추적 잔재 제거(위험)

git clean -fd

# 복구(운 좋게)

git reflog
git reset --hard HEAD@{N}
```

---

## 결론

- `--soft` 는 **커밋만 되돌리는 안전한 정리 도구**, `--mixed` 는 **스테이징 상태 초기화**, `--hard` 는 **WD까지 강제 동기화하는 날카로운 칼**이다.
- 공유 이력에는 `revert` 가 우선이며, reset/rebase 후 푸시는 보호 규칙과 `--force-with-lease` 로만 제한하라.
- 리셋 전 **백업 브랜치**와 **reflog 인지**는 생존 스킬이다.

---

## 참고 링크

- Git reset: https://git-scm.com/docs/git-reset
- Pro Git — Reset: https://git-scm.com/book/en/v2/Git-Tools-Reset-Demystified
- Git restore: https://git-scm.com/docs/git-restore
