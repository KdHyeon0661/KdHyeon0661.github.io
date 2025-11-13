---
layout: post
title: Git - rebase
date: 2025-01-30 19:20:23 +0900
category: Git
---
# Git Rebase

## 0. TL;DR(핵심 요약)

- **Rebase** = “내 브랜치 커밋들을 **다른 베이스** 위로 **재배치**해 **선형 이력**으로 만든다.”
- **언제?** 개인 브랜치 정리·리뷰 전 히스토리 단순화·FF 병합을 위해.
- **주의!** 공유(원격) 브랜치 rebase 금지. 필요 시 팀 합의 + `--force-with-lease`.
- **필수 옵션**:
  - `--onto`(선택 구간만 이동), `-i`(인터랙티브), `--rebase-merges`(병합 토폴로지 보존),
  - `--autostash`/`rebase.autoStash`(작업 보관), `--autosquash`(fixup!/squash! 자동 정렬).

---

## 1. Rebase란? (개념 재정의)

**rebase**는 현재 브랜치의 커밋들을 **새로운 기준 커밋** 뒤로 옮겨 붙이면서 **새 커밋 해시**로 다시 쓰는 과정입니다.
- 결과: **병합 커밋 없이** 선형 이력이 만들어짐(그래프가 단순해짐).
- 대가: 해시 재작성 → 원격과 공유된 브랜치에서는 충돌/협업 리스크.

### 최소 예시(기본)

```bash
git checkout feature/login
git rebase main
```

시각화(개념):

```text
rebase 전:
main:     A---B
               \
feature:        C---D

rebase 후:
main:     A---B
                   \
feature:            C'--D'
# C, D는 C', D'로 "재작성"되어 해시가 달라짐
```

---

## 2. rebase vs merge (실무형 비교)

| 항목 | merge | rebase |
|---|---|---|
| 병합 방식 | 병합 커밋 생성(브랜치 흔적 보존) | 커밋을 재배치(선형 이력) |
| 충돌 처리 | 한번(병합 커밋 시점) | 커밋마다(여러 번) |
| 이력 가독성 | 분기/병합 맥락 가시 | 선형·간결(맥락은 상대적으로 약화) |
| 협업 안전성 | 높음(권장) | 공유 브랜치에선 위험(해시 재작성) |
| 사용 맥락 | 기록 보존, 다자 협업 | 개인 브랜치 정리, 리뷰 전 정돈 |
| FF 병합 | 비필수 | 쉽게 가능(정리 후 `--ff-only`) |

---

## 3. Rebase 충돌 처리(기본→실전)

충돌 메시지:

```bash
CONFLICT (content): Merge conflict in index.html
```

해결 절차:

```bash
# 1. 충돌 파일 수정(마커 <<<<<<<, =======, >>>>>>> 제거/통합)
git add index.html

# 2. 계속
git rebase --continue

# 포기(시작 전으로 되돌리기)
git rebase --abort

# 현재 커밋 스킵
git rebase --skip
```

### 충돌 학습/재사용: rerere
같은 유형 충돌을 반복 해결할 때 자동 제안:

```bash
git config --global rerere.enabled true
```

---

## 4. 인터랙티브 Rebase(-i) — 정리·스쿼시·메시지·순서

```bash
# 최근 5개 커밋 정리
git rebase -i HEAD~5
```

열리는 편집화면에서 사용할 수 있는 대표 액션:

| 액션 | 의미 |
|---|---|
| `pick` | 커밋 유지 |
| `reword` | 메시지만 수정 |
| `edit` | 커밋 내용 수정(중단 후 amend 가능) |
| `squash` | 이전 커밋과 합치고 메시지 병합 |
| `fixup` | 이전 커밋과 합치되 메시지 폐기(깔끔) |
| `exec` | 해당 시점에서 쉘 명령 실행(테스트/린트 등) |
| `drop` | 커밋 제거 |

예시:

```
pick 3a1b2c3 login 기능 추가
reword 4d5e6f7 불필요한 로그 제거
fixup  8g9h0i1 리팩토링 사소 수정
exec   npm test
```

- `--autosquash` 와 `fixup!`, `squash!` 메시지를 함께 쓰면, 관련 커밋이 **자동으로** 원 커밋 뒤로 정렬:

```bash
git rebase -i --autosquash origin/main
```

> `edit`을 선택하면 그 커밋 시점에서 rebase가 멈춥니다. 수정 후:
> ```bash
> git commit --amend
> git rebase --continue
> ```

---

## 5. 고급: `--onto`로 “특정 구간만” 새 베이스로 이동

`--onto`는 **A 이후 B까지의 구간을 떼어내서** 새 베이스로 붙이는 강력한 도구입니다.

```bash
# 의미: A..B 구간(= B에서 A를 뺀 커밋들)을 newbase 위로 옮긴다.
git rebase --onto newbase A B
```

시나리오 예:
- 기능 브랜치에서 중간 일부만 다른 브랜치로 “이식”.
- 오래된 기반에서 시작한 연속 커밋 묶음을 최신 토대 위로 옮기기.

실전 절차(검증형):
```bash
# 먼저 어떤 커밋이 옮겨질지 확인
git log --oneline A..B

# 이동
git rebase --onto newbase A B
```

---

## 6. 병합 토폴로지 유지: `--rebase-merges`

기본 rebase는 병합 커밋을 평탄화하지만, 복잡한 브랜치 구조를 **보존**하고 싶다면:

```bash
git rebase --rebase-merges origin/main
```

- 다중 토픽 브랜치가 얽힌 복잡한 그래프에서 병합의 **맥락**을 유지한 채 재배치.
- 난이도/충돌 리스크는 증가 → 사전 테스트 권장.

---

## 7. 작업 보관/자동 스태시: `--autostash` 와 설정

rebase 시작 전에 워킹 디렉터리가 더럽다면 자동으로 stash/복원:

```bash
git rebase --autostash origin/main
# 또는 전역 설정
git config --global rebase.autoStash true
```

- rebase 중간에 변경사항을 안전하게 보관/복원.
- 반드시 변경 확인(`git status`) 습관화.

---

## 8. 업스트림 동기화 패턴: `git pull --rebase`

기본 `git pull`은 merge를 수행하지만, 로컬 이력을 선형으로 유지하려면:

```bash
git config --global pull.rebase true
# 또는 개별 호출
git pull --rebase
```

변형:
```bash
git pull --rebase=merges   # 병합 구조 보존하며 pull
git pull --rebase=interactive
```

> 팀 정책에 맞추어 표준화하세요. 공유 브랜치에서 rebase는 신중히.

---

## 9. rebase 전/후 비교: `git range-diff`

rebase(또는 `--onto`) 전/후에 “패치 시리즈”가 어떻게 변했는지 비교:

```bash
# main 기준, topic 브랜치를 재배치 전후 비교
git range-diff main...topic   # 또는 main..topic (환경에 맞게)
```

- 리뷰어에게 변경 요약 제공
- 패치 순서/내용 변화 검증

---

## 10. Rebase 이후 푸시: 안전한 강제 푸시

Rebase는 해시가 바뀌므로 원격에 **강제 푸시** 필요:

```bash
git push --force-with-lease origin feature/login
```

- `--force-with-lease`는 **동료의 최신 커밋을 보호**(내 로컬이 생각하는 원격 상태와 실제가 다르면 거부).
- 무조건 `-f` 보다는 `--force-with-lease`를 습관화.

---

## 11. 복구와 보험: ORIG_HEAD / reflog / 백업

rebase, reset, cherry-pick 등 **파괴적 작업 전** 스냅샷:

```bash
git branch backup/pre-rebase-$(date +%Y%m%d-%H%M)
# 또는
git tag pre-rebase
```

사고 처리:
```bash
git reflog                  # 최근 HEAD 이동 내역
git reset --hard ORIG_HEAD  # 직전 큰 작업 이전으로
# 또는 특정 시점으로 새 브랜치 구출
git switch -c rescue HEAD@{3}
```

---

## 12. 서브모듈·LFS와 rebase

- **서브모듈**: 상위 리포가 가리키는 **포인터(커밋 해시)** 가 재작성될 수 있음 → 서브모듈 디렉터리에서 적절한 커밋으로 체크아웃 후 상위에서 해당 포인터를 **다시 커밋** 필요.
- **Git LFS**: 포인터 파일/락 정책 확인. CI/개발 환경에서 `git lfs install` 누락 시 포인터만 남는 문제 발생.

---

## 13. 팀 운영 규칙(정책화 권장)

- **공유 브랜치(예: main/release/hotfix)는 rebase 금지**.
- 개인 브랜치에서는 rebase 자유, 단 **푸시 전**에만.
- rebase 후 푸시는 **`--force-with-lease`** 필수.
- PR에서 병합 정책 정의: “rebase and merge” 허용 여부, “squash and merge” 기준, 선형 이력 요구 여부.
- 린트/테스트 자동화와 인터랙티브 rebase의 `exec` 단계 연계(실수 방지).

---

## 14. 자주 쓰는 명령어(치트시트, 확장)

```bash
# 기본
git rebase main
git rebase origin/main
git rebase --continue | --abort | --skip

# 인터랙티브/자동 스쿼시
git rebase -i HEAD~5
git rebase -i --autosquash origin/main

# 특정 구간만 새 베이스로
git rebase --onto <newbase> <A> <B>

# 병합 토폴로지 보존
git rebase --rebase-merges origin/main

# 작업 보관 자동
git rebase --autostash
git config --global rebase.autoStash true

# pull 시 rebase
git config --global pull.rebase true
git pull --rebase
git pull --rebase=merges

# 전/후 비교
git range-diff main...feature/x

# 푸시
git push --force-with-lease
```

---

## 15. 실전 시나리오

### 15.1 개인 브랜치 정리 → FF 병합
```bash
git fetch origin
git checkout feature/pay
git rebase origin/main
# 충돌 해결 → add → rebase --continue 반복
git checkout main
git merge --ff-only feature/pay
```

### 15.2 오래된 중간 구간만 최신 베이스로(`--onto`)
```bash
# 토픽 브랜치 topic에서 A..B 구간만 newbase로 이동
git checkout topic
git log --oneline A..B          # 이동 대상 확인
git rebase --onto newbase A B
```

### 15.3 인터랙티브로 스쿼시 + 메시지 정리
```bash
git rebase -i --autosquash origin/main
# fixup!/squash! 커밋은 원 커밋 뒤로 자동 정렬
```

### 15.4 rebase 도중 테스트 자동 실행(`exec`)
```bash
git rebase -i HEAD~4
# todo 파일에:
# pick ...
# exec npm test
# pick ...
```

### 15.5 사고 복구
```bash
git reflog
git reset --hard ORIG_HEAD
# 또는
git switch -c rescue HEAD@{2}
```

---

## 16. 트러블슈팅(상황→원인→해결)

| 상황/증상 | 원인 | 해결 |
|---|---|---|
| rebase 중 반복 충돌 | 다수 커밋에 같은 충돌 | `rerere` 활성화(`rerere.enabled=true`), 충돌 해결 패턴 재사용 |
| rebase가 중단 상태로 고착 | 미해결 충돌/미스테이징 | `git status`로 파일 확인 → 수정 후 `add` → `rebase --continue` |
| rebase 후 push 거부 | 해시 재작성으로 원격과 불일치 | `git push --force-with-lease` |
| 동료 커밋 덮어씌움 우려 | `-f` 사용 | `--force-with-lease` 사용, 원격 상태 확인 |
| 작업 내용이 섞임 | rebase 전 비커밋 변경 있음 | `--autostash` 또는 수동 `git stash` 선행 |
| 서브모듈 포인터 충돌 | 상하위 리포 포인터 불일치 | 서브모듈 디렉터리에서 원하는 커밋 checkout → 상위에서 그 포인터 커밋 |
| LFS 포인터만 남음 | LFS 미설치/락 문제 | `git lfs install`, LFS 정책 점검, CI 설정 확인 |

---

## 17. 안전한 워크플로우 체크리스트

- [ ] **개인 브랜치**에서만 rebase
- [ ] 시작 전 `git fetch`로 최신 업스트림 동기화
- [ ] 비커밋 변경은 `--autostash` 또는 수동 `stash`
- [ ] 대형 작업 전 **백업 브랜치/태그** 생성
- [ ] 충돌 반복 시 `rerere` 켬
- [ ] 푸시는 `--force-with-lease`
- [ ] 전/후 비교는 `git range-diff`
- [ ] 서브모듈/LFS 있는 리포는 별도 체크

---

## 18. 최소 실습 루틴(로컬 재현)

```bash
# 1. 준비
mkdir rebase-lab && cd rebase-lab
git init
echo "v1" > app.txt
git add . && git commit -m "init: v1"
git branch -M main

# 2. 토픽 생성
git checkout -b feature/ui
echo "ui1" >> app.txt
git commit -am "feat(ui): ui1"
echo "ui2" >> app.txt
git commit -am "feat(ui): ui2"

# 3. main도 변경
git checkout main
echo "core1" >> app.txt
git commit -am "feat(core): core1"

# 4. rebase
git checkout feature/ui
git rebase main
# 충돌 시: 파일 수정 → git add → git rebase --continue

# 5. 인터랙티브 정리
git rebase -i HEAD~2

# 6. 복구 체험
git reflog
git reset --hard ORIG_HEAD
```

---

## 19. 결론

- **Rebase**는 “선형 이력”과 “리뷰/분석 편의”를 주지만, **협업 안전성**을 대가로 합니다.
- 개인 브랜치에서 **정리·스쿼시·재배치**에 적극 활용하고, 공유 브랜치에서는 **merge 중심**으로 운용하세요.
- `--onto`, `-i`, `--rebase-merges`, `--autostash`, `--autosquash`, `range-diff`, `rerere`, `--force-with-lease`를 함께 익히면 **안전하고 강력한 rebase 워크플로우**를 구축할 수 있습니다.

---

## 참고

- Git Rebase 공식 문서: https://git-scm.com/docs/git-rebase
- GitHub 가이드(포크 동기화 및 rebase): https://docs.github.com/en/pull-requests/collaborating-with-pull-requests/working-with-forks/keeping-your-fork-synced-with-the-original-repository#rebase-your-fork
