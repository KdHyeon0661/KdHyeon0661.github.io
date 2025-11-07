---
layout: post
title: Git - reset vs revert vs rebase
date: 2025-02-06 19:20:23 +0900
category: Git
---
# Git: `reset` vs `revert` vs `rebase`

## 0) 기초 복습 — Git의 3가지 “상태”

- **HEAD**: 현재 체크아웃한 커밋(정확히는 브랜치 참조 또는 detached HEAD의 커밋)을 가리키는 포인터  
- **Index(= Staging area)**: 다음 커밋을 만들기 위해 모아둔 스냅샷  
- **Working tree**: 실제 파일이 있는 작업 디렉터리

명령어들은 주로 이 세 영역을 어떻게 “옮기고/맞추는지”로 구분할 수 있습니다.

---

# 1) `git reset` — 과거로 포인터를 되돌리는 강력하고 위험한 도구

## 1.1 개념(복습 강화)
`reset`은 **브랜치 포인터(HEAD)** 를 과거의 커밋로 이동시키고, 옵션에 따라 **Index / Working tree**를 그 상태로 **맞춥니다**.

- **--soft**: HEAD(브랜치)만 이동. Index와 Working tree 는 **그대로** 유지  
- **--mixed**(기본): HEAD + Index 초기화(해당 범위 변경이 unstaged 로). Working tree 는 유지  
- **--hard**: HEAD + Index + Working tree 를 **모두** 지정 커밋 상태로 **초기화**(변경 삭제)

## 1.2 시각적 이해
```
A---B---C---D   (main, HEAD)

# reset --soft B
B   (main, HEAD)
 \__ [C,D의 변경은 Index에 남아있음: 재커밋 가능]

# reset --mixed B
B   (main, HEAD)
 \__ [C,D의 변경은 Working tree에만 남음: untracked 변경처럼 보임]

# reset --hard B
B   (main, HEAD)            # C,D에서 변경된 파일은 Working tree에서도 사라짐
```

## 1.3 사용 예제(요청 내용 확장)
최근 커밋을 되돌리고 다시 커밋하고 싶을 때:
```bash
# 마지막 커밋을 취소하고, 변경은 그대로 유지(Index에 남김)
git reset --soft HEAD~1
git commit -m "메시지 수정 후 다시 커밋"

# 마지막 커밋 취소 + 스테이징 해제(Working tree에는 변경 유지)
git reset --mixed HEAD~1
git add -p
git commit -m "부분만 다시 선택해 커밋"

# 모든 변경 완전 폐기(주의)
git reset --hard HEAD~1
```

## 1.4 위험도와 협업상의 금기
- 공개/공유 브랜치(동료가 pull 해 가는 브랜치)에서 **reset으로 히스토리를 바꾸면** 동료의 이력이 꼬입니다.  
- 특히 `--hard`는 Working tree 변경까지 삭제하므로 **돌이키기 매우 어려운 실수**가 될 수 있습니다.

## 1.5 실수 복구(필수)
```bash
# 최근 이동/변경의 기록(로컬 참조 기록)
git reflog

# reset 직전 상태로 복구(ORIG_HEAD는 일부 작업에서 이전 HEAD를 가리킴)
git reset --hard ORIG_HEAD

# 또는 reflog에서 원하는 시점으로 복귀
git reset --hard HEAD@{2}

# 안전하게 별도 브랜치 만들어 백업
git switch -c rescue HEAD@{2}
```

---

# 2) `git revert` — 기존 커밋은 보존하고 “반대 동작 커밋”을 쌓는 방법

## 2.1 개념(복습 강화)
`revert`는 지정 커밋의 변경을 **반대로 적용하는 새 커밋**을 생성합니다.  
즉, **히스토리는 보존**하면서도 “취소했다”는 기록이 남습니다. 협업·감사 추적에 **가장 안전**합니다.

## 2.2 기본 예제
```bash
# 1개 커밋 취소
git revert 9fceb02

# 범위 되돌리기(HEAD 포함 최근 3개)
git revert HEAD~3..HEAD
```

## 2.3 Merge 커밋을 되돌릴 때(중요)
Merge 커밋을 revert 하려면 **어느 부모를 기준으로 반전할지** 지정해야 합니다.
```bash
# -m 1: 첫 번째 부모(main 쪽)를 기준으로 revert
git revert -m 1 <merge-commit-id>
```
- 잘못 지정하면 원치 않는 대량 변경이 반영될 수 있으니, **실험 브랜치에서 먼저 검증**하세요.

## 2.4 충돌 처리
`revert`도 내부적으로 patch를 적용하므로 충돌이 날 수 있습니다.
```bash
# 충돌 시:
# 파일 수정 → 마커 제거 → add
git add <files>
git revert --continue

# 중단
git revert --abort
```

## 2.5 되돌린 커밋을 다시 되돌리기(“revert of revert”)
```bash
git revert <revert-commit-id>
```
- 원래 변경이 복원됩니다(상황에 따라 충돌/검토 필요).

---

# 3) `git rebase` — 커밋을 다른 기반 위로 “재생성”하여 히스토리를 정렬

## 3.1 개념(복습 강화)
`rebase`는 **현재 브랜치의 커밋들을 다른 브랜치의 끝으로 재배치**합니다.  
장점: 병합 커밋 없이 **선형 이력**을 만들 수 있음  
단점: 커밋이 **재작성**되므로 해시가 바뀌고, 공유 브랜치에서는 혼란의 원인이 됨

## 3.2 기본 예제
```bash
git checkout feature
git rebase main
```
- `feature` 위 커밋들이 `main` 끝으로 순서대로 재적용됩니다.

## 3.3 인터랙티브 rebase로 “정리”
```bash
git rebase -i HEAD~5
# pick, reword, edit, squash, fixup, drop 등을 통해 커밋 정리
```

### squash / fixup 자동화(추천)
```bash
# 커밋 메시지에 fixup!/squash! 프리픽스 사용 후
git rebase -i --autosquash HEAD~10
```

## 3.4 병합 토폴로지를 살리며 rebase(고급)
```bash
git rebase --rebase-merges main
```
- 단순 선형화가 아니라 서브토픽 브랜치의 구조적 관계를 보존하며 재배치

## 3.5 pull.rebase 설정
```bash
git config --global pull.rebase true
git pull --rebase
```
- 팀에서 **선형 히스토리**를 선호한다면 기본으로 설정해두면 편리합니다.

## 3.6 충돌 처리 루틴
```bash
git rebase main
# 충돌 → 파일 수정 → add
git rebase --continue

# 현재 커밋 건너뛰기
git rebase --skip

# 전체 중단
git rebase --abort
```

---

# 4) reset vs revert vs rebase — 본질적 차이와 안전 지침

## 4.1 목적/영향/안전성(강화 표)
| 항목 | `reset` | `revert` | `rebase` |
|---|---|---|---|
| 목표 | 과거로 포인터 이동(히스토리 되돌림) | 지정 커밋을 “반대로” 적용하는 새 커밋 | 커밋을 다른 기반 위로 재배치(히스토리 정리) |
| 히스토리 변경 | O(참조 재작성) | X(보존) | O(재작성: 해시 변경) |
| 안전성(협업) | 낮음(로컬/개인 브랜치 권장) | 높음(공유 브랜치 권장) | 중간(공유 브랜치 지양) |
| Working tree 영향 | 옵션에 따라 변경 삭제 가능 | 변경 없음(새 커밋만 추가) | 파일 재적용 과정에서 충돌 가능 |
| 사용 예 | 최근 커밋 취소/분해/재커밋 | 운영에서 문제된 커밋 빠르게 되돌리기 | 기능 브랜치 정리, 선형 히스토리 유지 |
| 복구 난이도 | 높음(잘못하면 손실 위험) | 낮음(기록 남음) | 중간(충돌/강제푸시 동반) |

## 4.2 추천 시나리오(요청 표 확장)
| 상황 | 추천 |
|---|---|
| 최근 커밋 메시지만 바꾸고 싶다 | `git reset --soft HEAD~1` → 다시 커밋 |
| 최근 커밋을 나눠서 부분만 재커밋 | `git reset --mixed HEAD~1` → `git add -p` |
| 협업 중 실수 커밋을 취소 | `git revert <커밋>` |
| 기능 브랜치 정리 후 깔끔히 머지 | `git rebase -i` 또는 GitHub `Rebase and merge` |
| 팀 정책상 병합 커밋 금지(선형 이력) | `git pull --rebase`, 보호 브랜치 `Require linear history` |

---

# 5) 충돌/실수 복구 — 현장에서 가장 중요한 부분

## 5.1 충돌 마커와 해결 절차
충돌 시 파일에는 다음 마커가 들어갑니다.
```text
<<<<<<< ours
# 내 버전
=======
# 상대(기준) 버전
>>>>>>> theirs
```
해결 후 마커는 반드시 삭제 → `git add` → `continue` (rebase/revert/merge에 따라 명령 달라짐)

## 5.2 빠른 전체 선택(파일 단위)
```bash
git checkout --ours   path/to/file
git checkout --theirs path/to/file
git add path/to/file
```

## 5.3 rerere(반복 충돌 자동 학습)
```bash
git config --global rerere.enabled true
```
과거에 동일 패턴으로 해결한 충돌을 재적용합니다(그래도 눈으로 재확인 후 커밋하세요).

## 5.4 reflog/ORIG_HEAD로 구조적 복구
```bash
git reflog                   # 과거 HEAD 이동 이력 확인
git reset --hard ORIG_HEAD   # 직전 위험 작업 전으로 복귀
git switch -c rescue HEAD@{2}  # 안전 브랜치 확보
```

---

# 6) 협업 정책/보호 브랜치/서명과의 상호작용

## 6.1 보호 브랜치(Branch protection rules)
- Require pull request reviews  
- Require status checks to pass  
- Require linear history(merge commit 금지 → rebase/squash만 허용)  
- Require signed commits(GPG 서명 필수)  
- Force-push 제한 등

이 규칙은 `reset`/`rebase` 기반 강제푸시와 **충돌**할 수 있습니다. 공개 브랜치에서는 **revert** 기반으로 처리하는 것이 안전합니다.

## 6.2 서명/GPG/DCO
- rebase(재작성) 시 커밋 해시가 바뀌며, **서명이 파손**될 수 있습니다.  
- 조직이 **Require signed commits** 나 **DCO**를 쓰는 경우, PR 커밋들이 정책을 준수하도록 미리 세팅하세요.
```bash
git config --global user.signingkey <KEYID>
git config --global commit.gpgsign true
```

---

# 7) 대규모 히스토리 정리 실전 팁

- **rebase -i + autosquash** 로 잔 커밋을 `fixup!/squash!` 패턴으로 자동 압축  
- **rebase --rebase-merges** 로 서브토픽 구조를 보존하며 정리  
- 정리 전에 **백업 태그/브랜치** 생성 권장:
```bash
git tag before-rewrite
# 또는
git switch -c backup/2025-11-06
```
- 팀 합의 없이 공개 브랜치 전체를 rebase 하지 말 것

---

# 8) 재현 가능한 랩(전체 스크립트) — 로컬에서 바로 실습

## 8.1 reset 실습
```bash
rm -rf git-reset-lab && mkdir git-reset-lab && cd git-reset-lab
git init
git config user.name  "lab"
git config user.email "lab@example.com"

echo "v1" > f.txt
git add . && git commit -m "v1"
echo "v2" >> f.txt
git commit -am "v2"
echo "v3" >> f.txt
git commit -am "v3"

git log --oneline

# soft: 브랜치만 이동, Index/Working 유지
git reset --soft HEAD~1
git status        # staged 상태 확인
git commit -m "v3(rewrite by soft)"

# mixed: 브랜치 이동 + Index 초기화
git reset --mixed HEAD~1
git status        # 변경은 워킹에 남음
git add -p
git commit -m "v3(partial re-commit)"

# hard: 모두 롤백(주의: 워킹도 되돌림)
git reset --hard HEAD~1
git log --oneline

# 복구(실수 시)
git reflog
git reset --hard ORIG_HEAD
```

## 8.2 revert 실습
```bash
rm -rf git-revert-lab && mkdir git-revert-lab && cd git-revert-lab
git init
git config user.name  "lab"
git config user.email "lab@example.com"

echo "A" > a.txt
git add . && git commit -m "A"
echo "B" >> a.txt
git commit -am "B"
git log --oneline

# B 커밋 취소
B=$(git rev-parse --short HEAD)
git revert $B
git log --oneline
cat a.txt      # B가 반전되어 A상태로 돌아감(새 커밋 추가됨)

# 범위 revert
echo "C" >> a.txt && git commit -am "C"
echo "D" >> a.txt && git commit -am "D"
git log --oneline

git revert HEAD~1..HEAD   # C,D 되돌림
git log --oneline
```

## 8.3 rebase 실습(충돌 포함)
```bash
rm -rf git-rebase-lab && mkdir git-rebase-lab && cd git-rebase-lab
git init
git config user.name  "lab"
git config user.email "lab@example.com"
git branch -M main

echo "base" > app.txt
git add . && git commit -m "base"

git switch -c feature
echo "f1" >> app.txt
git commit -am "feat: f1"
echo "f2" >> app.txt
git commit -am "feat: f2"

git switch main
echo "m1" >> app.txt
git commit -am "feat: m1"

git switch feature
git rebase main || true      # 일부러 충돌나면 수동 해결
# 파일 열어 해결 → add
git rebase --continue

# 정리 완료 후 확인
git log --oneline --graph --decorate --all
```

---

# 9) 고급: reset과 rebase의 혼합 시나리오

## 9.1 최근 커밋을 분해해서 메시지/내용 재구성
```bash
# 최근 커밋을 index/working으로 풀어헤침
git reset --mixed HEAD~1

# 변경 중 일부만 선택 커밋
git add -p
git commit -m "부분1"
git add -p
git commit -m "부분2"

# 마지막에 인터랙티브 rebase 로 메시지 정리
git rebase -i HEAD~2
```

## 9.2 PR 직전 “깨끗한” 이력 만들기
```bash
git fetch origin
git rebase origin/main        # 최신 기준 위로 재배치
git rebase -i HEAD~10 --autosquash
git push --force-with-lease
```
- 팀이 선형 이력을 강제한다면 필수적. 단, 공개 브랜치의 재작성은 팀 컨벤션을 따를 것.

---

# 10) FAQ — 실무에서 자주 받는 질문

**Q1. 이미 push한 커밋을 reset으로 지워도 되나?**  
A. 공개 브랜치에서는 지양. 동료 이력에 충돌을 유발. revert가 안전.

**Q2. revert로 되돌렸는데, 나중에 다시 적용하고 싶다?**  
A. “revert 커밋”을 다시 revert 하거나, 해당 변경을 포함한 새 커밋을 만들면 된다.

**Q3. rebase 중 꼬였을 때 가장 먼저 뭘 보나?**  
A. `git status`, `git rebase --abort`, `git reflog` 순서로 상황 파악/복구.

**Q4. rebase와 merge 중 무엇이 더 좋은가?**  
A. 절대적 답은 없음. 팀 정책/리뷰 문화/CI 파이프라인에 맞춰 선택.  
   선형 이력을 원하면 rebase, 토폴로지 보존/기록을 원하면 merge.

**Q5. revert와 squash의 관계?**  
A. Squash는 여러 커밋을 1개로 압축하는 병합 방식이고, revert는 특정 커밋의 반대 동작을 “새 커밋으로” 추가한다. 목적이 다르다.

---

# 11) 명령어/옵션 치트시트

```bash
# reset
git reset --soft  <rev>   # HEAD만
git reset --mixed <rev>   # HEAD+Index(기본)
git reset --hard  <rev>   # HEAD+Index+Working (주의)
git reflog                # 이전 HEAD 이력 확인
git reset --hard ORIG_HEAD

# revert
git revert <rev>          # 1개 취소(새 커밋)
git revert <A>..<B>       # 범위 취소(B 포함)
git revert -m 1 <merge>   # Merge 커밋 취소(부모 선택)
git revert --continue | --abort

# rebase
git rebase <upstream>
git rebase -i HEAD~N
git rebase --rebase-merges <upstream>
git rebase --continue | --skip | --abort
git config --global pull.rebase true
git pull --rebase
git push --force-with-lease

# 충돌 해결
git status
git checkout --ours   <path>
git checkout --theirs <path>
git add <path>
git mergetool
git config --global rerere.enabled true
```

---

# 12) 결론

- **reset**: 강력하지만 위험. 로컬 정리/최근 실수 바로잡기에만 사용. 공개 브랜치에는 부적절.  
- **revert**: 협업 친화. 히스토리를 보존하며 “취소” 사실을 기록. 운영 중 “빠른 복구”에 최적.  
- **rebase**: 선형 이력과 깔끔한 커밋을 위한 도구. 공유 브랜치에서의 무분별한 사용은 금물.  
- 팀 정책(보호 브랜치/서명/CI)과 맞물려 올바른 도구를 고르면, **추적 가능한 기록 + 생산적인 협업**을 동시에 달성할 수 있다.

---

## 참고
- git reset: https://git-scm.com/docs/git-reset  
- git revert: https://git-scm.com/docs/git-revert  
- git rebase: https://git-scm.com/docs/git-rebase