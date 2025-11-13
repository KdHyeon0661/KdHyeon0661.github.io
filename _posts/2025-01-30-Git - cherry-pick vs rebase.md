---
layout: post
title: Git - cherry-pick vs rebase
date: 2025-01-30 20:20:23 +0900
category: Git
---
# Git **cherry-pick vs rebase**

## 0. 한눈에 요약

- **cherry-pick**: “저 브랜치의 **특정 커밋 몇 개만** 지금 브랜치에 **복사**(새 커밋으로 재생성)해서 붙이자.”
- **rebase**: “내 브랜치의 **전체(또는 연속 구간) 커밋**을 **다른 베이스 위로 재배치**해 선형 이력을 만들자.”

**선택 가이드**
- 특정 수정(버그 픽스 등)만 골라서 반영 → **cherry-pick**
- 기능 브랜치 이력을 깔끔히 정리/선형화 후 병합 → **rebase**

---

## 1. `git cherry-pick`이란?

**특정 커밋만 선택**해서 현재 브랜치에 **새 커밋**으로 적용합니다.
- 장점: 필요한 것만 골라서 반영(불필요한 변경 유입 방지)
- 단점: 커밋이 “복사”되므로 **해시가 달라짐**(중복 관리 필요), 많이 사용하면 “포크된 이력”이 산개

### 기본 명령

```bash
# 현재 브랜치로 <commit> 들을 순서대로 적용
git cherry-pick <commit>
git cherry-pick <commit1> <commit2> <commit3>
git cherry-pick <start>^..<end>   # 연속 범위(주의: 아래 설명)
```

#### 범위 표기 주의(A..B vs A^..B)
- `A..B` 는 “B에서 A를 뺀 차집합”의 의미(보통 **그래프 차이**에서 쓰임)
- `A^..B` 는 “A의 **바로 이전**부터 B까지 **연속 커밋**”이라는 관용적 표기
  - 실수 방지를 위해, **연속 범위**는 보통 `^` 를 붙여 명시하거나 **`git log A..B --oneline`** 으로 먼저 **검증 후** pick 하세요.

---

## 2. cherry-pick 실전 옵션(필수)

```bash
# 메시지에 원본 커밋 해시 흔적을 남김: (cherry picked from commit <sha>)
git cherry-pick -x <commit>

# 커밋 메시지 편집 열기
git cherry-pick -e <commit>

# 여러 커밋을 연속 적용하지만, 하나로 묶어 나중에 직접 커밋
git cherry-pick -n <commit1> <commit2> ...
git commit -m "squashed fixes from hotfixes"

# DCO/서명 라인 추가
git cherry-pick -s <commit>

# GPG/SSH 서명(서명 커밋 정책일 때)
git cherry-pick -S <commit>

# 병합 커밋을 pick(다중 부모 중 어느 쪽을 기준으로 할지 지정)
git cherry-pick -m 1 <merge-commit-sha>   # 첫 부모를 기준으로
```

**추천 실무 습관**
- 조직에서 “추적성” 중요 → `-x` 기본 사용
- 정책상 Signed-off-by 필요 → `-s`
- 서명 강제 → `-S`(사전 `git config`로 서명키 설정)

---

## 3. cherry-pick 충돌 처리(강화)

```bash
git cherry-pick <commit>
# 충돌 발생 시
#   CONFLICT (content): ...
# 파일 열어 수동 수정 → 충돌 마커(<<<<<<< ======= >>>>>>>) 해결
git add <fixed-files>
git cherry-pick --continue
# 포기
git cherry-pick --abort
# 이번 커밋만 건너뛰고 계속
git cherry-pick --skip
```

### 반복 충돌 최소화: rerere
과거에 했던 충돌 해결을 **학습해서 재사용**:
```bash
git config --global rerere.enabled true
```
- 같은 유형 충돌이 반복될 때 자동 적용(검토 후 커밋)

### 빈 커밋/중복 처리
- 이미 동일 변경이 적용된 경우 **“The previous cherry-pick is now empty”** 경고
  - 유지할 필요 없으면 `git cherry-pick --skip`
  - 메시지만 남기고 싶다면 `--allow-empty` 로 명시적 생성

---

## 4. cherry-pick 예제(추가 시나리오)

### 4.1 `hotfix` 의 특정 수정만 `main`에 반영
```bash
git checkout main
git cherry-pick -x abc1234  # hotfix 커밋
git push
```

### 4.2 여러 커밋을 하나의 커밋으로 묶어 반영
```bash
git checkout main
git cherry-pick -n fix1 fix2 fix3
# 필요 시 파일 더 수정
git commit -S -m "fix(core): backport critical fixes (squashed)"
```

### 4.3 병합 커밋(cherry-pick -m)
```bash
# merge 커밋을 특정 부모 기준으로 재적용
git cherry-pick -m 1 <merge-commit-sha>
```
- **주의**: 실제 병합 상황/맥락이 복잡하면 충돌이 더 잦음. 가능하면 **원인 커밋들**을 개별 pick 하는 게 수월할 때가 많습니다.

---

## 5. cherry-pick 되돌리기/복구

- pick 직후 전체 취소: `git cherry-pick --abort` (진행 중인 시퀀스)
- 이미 커밋됨 → **되돌리는 커밋** 생성:
  ```bash
  git revert <picked-commit-sha>
  ```
- 복구(큰 수술 전 보험):
  - pick 전후의 안전 지점 확인: `git reflog`
  - 위험 작업 전 `git tag temp/pre-pick` 또는 `git branch backup/pick-<ts>`로 스냅샷

---

## 6. `git rebase`란? (비교의 기준 마련)

**브랜치의 커밋들을 다른 베이스 위로 옮겨 붙이며 새 커밋으로 재작성**하여 **선형 이력**을 만듭니다.

### 기본형

```bash
# 현재 브랜치를 origin/main 최신 위로 재배치
git fetch origin
git rebase origin/main
```

- 이력 “재작성”이므로 **해시가 모두 바뀜**
- 충돌 발생 시 각 커밋 지점마다 해결 → `git rebase --continue`
- 포기: `git rebase --abort`
- 특정 커밋 건너뛰기: `git rebase --skip`
- 되돌리기 비상키: 작업 직전 **ORIG_HEAD**, 그리고 `git reflog`

### `--onto` (선택적 커밋 구간만 옮기기) — 강력 필수
```bash
# topic 브랜치에서 A..B 구간을 "newbase" 위로 옮긴다
git rebase --onto newbase A B
```
- “A 이후부터 B까지”를 통째로 새 베이스로 이동
- 브랜치 재배치/분기 재정렬/히스토리 다이어트에 매우 유용

### Interactive rebase(-i) — 정리/스쿼시/메시지 편집
```bash
git rebase -i origin/main
# 편집창에서 pick/squash/fixup/reword/edit 등 조작
```
- `--autosquash` 와 `fixup!`/`squash!` 커밋 메시지를 활용하면 자동 정렬:
  ```bash
  git rebase -i --autosquash origin/main
  ```

### 병합 유지: `--rebase-merges`
- 병합 구조를 보존하며 rebase(복잡한 토폴로지 유지)
```bash
git rebase --rebase-merges origin/main
```

---

## 7. cherry-pick vs rebase — 확장 비교

| 항목 | cherry-pick | rebase |
|---|---|---|
| 목적 | **특정 커밋만 선택적으로** 반영 | 브랜치 전체(또는 구간)를 **다른 베이스로 재배치** |
| 결과 | 선택한 커밋이 **현재 브랜치 끝에 복사**됨 | 대상 커밋들이 **새 해시**로 재작성되어 **선형화** |
| 충돌 | pick한 커밋 지점에서만 | 각 커밋마다(여러 번 발생 가능) |
| 히스토리 | 필요한 변경만 최소 반영(분산/중복 가능) | 깔끔·선형(맥락 보존은 약해질 수 있음) |
| 협업 안정성 | 비교적 안전(선택적 반영, 공유 브랜치에도 OK) | **공유 브랜치 rebase 금지**(이력 재작성) |
| 기록 추적성 | `-x` 로 원본 해시 흔적 남기기 권장 | ORIG_HEAD/reflog 기반 복구 가능 |
| 사용 예 | 버그 픽스 백포트, 외부 리포의 좋은 커밋만 반영 | 리뷰 전 개인 브랜치 정리, 노이즈 커밋 스쿼시 |

---

## 8. 선택 기준(확장)

- **이 커밋 몇 개만 가져오면 끝** → **cherry-pick**
- **이 브랜치를 최신 main 위에서 깔끔히 만들고 합치자** → **rebase**
- 공용 원격에 올린 브랜치 → **rebase 지양**(필요 시 팀 합의 및 `--force-with-lease`)
- 백포트/긴급 패치/릴리스 브랜치 유지보수 → **cherry-pick** 선호

---

## 9. 실무 시나리오 모음

### 9.1 릴리스 브랜치에 버그 픽스 백포트
```bash
# fix 커밋은 main에서 이미 통과됨
git checkout release/1.2
git cherry-pick -x <fix-sha>
git push
```

### 9.2 기능 브랜치 선형화 후 FF 병합
```bash
git fetch origin
git checkout feature/pay
git rebase origin/main
# 충돌 해결 → --continue
git checkout main
git merge --ff-only feature/pay
```

### 9.3 대규모 브랜치 일부만 새 위치로 옮기기(--onto)
```bash
# 토픽에서 A..B만 떼어 newbase 위로
git rebase --onto newbase A B
```

### 9.4 이미 pick한 커밋을 또 pick(중복 감지)
```bash
git cherry-pick <sha>
# empty/duplicate 메시지 → --skip 또는 --allow-empty 판단
```

### 9.5 인터랙티브 정리 + autosquash
```bash
# 'fixup! <message>' 형태 커밋이 있다면 자동 정렬
git rebase -i --autosquash origin/main
```

---

## 10. 복구·되돌리기 안전 가이드

### cherry-pick 시퀀스 중
```bash
git cherry-pick --abort
git cherry-pick --continue
git cherry-pick --skip
```

### rebase 중
```bash
git rebase --abort
git rebase --continue
git rebase --skip
```

### 큰 작업 전 스냅샷
```bash
git branch backup/pre-op-$(date +%Y%m%d-%H%M)
# 또는
git tag pre-op
```

### 사고 이후 복구
```bash
git reflog
git reset --hard ORIG_HEAD          # 직전 위험 작업 이전으로
# 또는
git switch -c rescue <reflog-entry>
```

---

## 11. 협업 안전 수칙

- **공유 브랜치(release/main/hotfix)** 에는 rebase 금지(정책화)
- rebase 후 푸시는 `git push --force-with-lease`(동료 작업 보호)
- cherry-pick은 추적성 확보를 위해 **`-x` 강력 권장**
- PR 기준으로 squash/rebase/merge 정책을 팀 문서화(리뷰/CI 체크와 연계)

---

## 12. 명령어 치트시트(확장)

```bash
# cherry-pick
git cherry-pick <sha>
git cherry-pick <sha1> <sha2> ...
git cherry-pick <start>^..<end>   # 연속 범위
git cherry-pick -x -e -S <sha>    # 원본 해시 흔적+메시지 편집+서명
git cherry-pick -n <...> && git commit   # 여러 개 모아 한 번에 커밋
git cherry-pick --continue | --abort | --skip
git cherry-pick -m 1 <merge-sha>  # 병합 커밋 pick

# rebase
git rebase <upstream>             # 내 브랜치 커밋을 upstream 위로
git rebase --onto <newbase> <A> <B>
git rebase -i [--autosquash] <upstream>
git rebase --rebase-merges <upstream>
git rebase --continue | --abort | --skip

# 복구
git reflog
git reset --hard ORIG_HEAD
git switch -c rescue <sha_or_HEAD@{n}>
```

---

## 13. 최소 실습 세트(로컬에서 재현)

```bash
# 준비
mkdir cp-vs-rb && cd cp-vs-rb
git init
echo v1 > app.txt
git add . && git commit -m "init: v1"
git branch -M main

# feature 브랜치
git checkout -b feature/ui
echo "ui=1" >> app.txt
git commit -am "feat(ui): v1"
echo "ui=2" >> app.txt
git commit -am "feat(ui): v2"

# main에서 ui의 v2만 cherry-pick
git checkout main
git cherry-pick -x HEAD@{1}   # (예: 위에서 v2 sha 지정)

# 다시 feature/ui를 main 위로 rebase
git checkout feature/ui
git rebase main
# 충돌 시 해결 -> add -> rebase --continue

# reflog 확인/복구 체험
git reflog
```

---

## 14. 결론

- **cherry-pick**은 **정밀 선택**이 장점: 필요한 커밋만 빠르게 백포트/반영. `-x`로 추적성 보강.
- **rebase**는 **이력 선형화**가 장점: 리뷰와 bisect가 쉬워짐. 다만 공유 브랜치에는 신중.
- 두 도구를 **목적에 맞게** 쓰되, 항상 **복구 전략(reflog/ORIG_HEAD/백업 브랜치)** 를 염두에 두면 실전 안정성이 크게 올라갑니다.

---

## 참고

- Git cherry-pick: https://git-scm.com/docs/git-cherry-pick
- Git rebase: https://git-scm.com/docs/git-rebase
