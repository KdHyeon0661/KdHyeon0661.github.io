---
layout: post
title: Git - rebase and merge & 자동 병합 정책 설정
date: 2025-02-06 19:20:23 +0900
category: Git
---
# GitHub의 `Rebase and merge`

## 0) 한눈에 보기 — 무엇이, 왜 필요한가

- **Rebase and merge**: PR의 커밋들을 **base 브랜치(main)의 최신 커밋 뒤에 순서대로 재적용**하여 병합 커밋 없이 **선형(linear) 이력**을 만든다.
- **장점**: 이력이 간결해지고 `git log --oneline --graph` 가 깔끔하다. `git bisect` 흐름도 직관적이다.
- **주의**: 커밋 해시가 바뀐다(재작성). 충돌 시 PR 화면에서 바로 해결되지 않을 수 있으며, 로컬 rebase가 필요하다. GPG/DCO 서명 정책과의 호환성도 점검해야 한다.
- **팀 정책**: 저장소 설정에서 **Allow rebase merges** 활성화 + **Require linear history** 로 병합 커밋을 차단하면 효과가 가장 크다.

---

# 1) 개념과 작동 원리

## 1.1 GitHub 병합 방식 3종
- Create a merge commit
- Squash and merge
- **Rebase and merge** ← 본 문서의 주제

## 1.2 Rebase and merge란?
- PR 브랜치의 각 커밋을 **base 브랜치 끝에 재적용**한다.
- GitHub는 서버 측에서 rebase를 수행하여 base 브랜치에 **fast-forward** 가능한 상태를 만든 뒤 병합한다.
- 결과적으로 **merge commit 없음**, 이력은 **선형**.

### 시각적 예시
```
main:    A---B---C
               \
feature:        D---E

# Rebase and merge 결과(개념적으로):
main:    A---B---C---D'---E'
```
- `D'`, `E'`는 재작성된 커밋(해시 변경). 코드 내용은 동일하되 **커밋의 부모**가 바뀌므로 해시가 달라진다.

## 1.3 내부 동작(핵심 포인트)
- GitHub는 PR을 병합하기 직전, **서버 측 rebase**를 수행한다.
- 이 과정에서 **충돌**이 없고, **보호 규칙**(CI, 리뷰, 서명, 선형 이력 등)이 충족되면 fast-forward로 base에 반영한다.
- 충돌 발생 시:
  - PR 화면에 “Rebase conflicts” 류의 메시지가 나타날 수 있다.
  - **웹 에디터로 곧바로 해결 불가**한 케이스가 잦다(서버 측 rebase 성격).  
  - 일반적으로 **로컬에서 rebase** → PR 브랜치 업데이트 → 다시 시도.

---

# 2) 실전 사용법(웹·CLI)과 재현 가능한 예제

## 2.1 웹에서 절차
1. PR 페이지 하단의 병합 드롭다운에서 **Rebase and merge** 선택
2. 상태 체크(Required status checks)·리뷰 조건 충족 확인
3. **Confirm rebase and merge** 클릭
4. 성공 시 base 브랜치(main)에 **merge commit 없이** 커밋들이 선형으로 반영

## 2.2 GitHub CLI(gh) 예
```bash
# PR 123을 rebase 방식으로 병합
gh pr merge 123 --rebase
# 관리자 권한이 있고 보호 규칙을 무시해야 한다면(권장 X)
# gh pr merge 123 --rebase --admin
```

## 2.3 재현 가능한 로컬-→PR 실습(gh 포함)
```bash
# 준비
rm -rf rebase-merge-lab && mkdir rebase-merge-lab && cd rebase-merge-lab
git init
git config user.name  "lab"
git config user.email "lab@example.com"
git branch -M main
echo "v1" > app.txt
git add . && git commit -m "chore: init v1"

# 새 기능 브랜치(F1)에서 커밋 여러 개
git checkout -b feature/f1
echo "f1-a" >> app.txt
git commit -am "feat(f1): a"
echo "f1-b" >> app.txt
git commit -am "feat(f1): b"

# main에도 독립 변경(병합 커밋 만들지 않음)
git checkout main
echo "main-x" >> app.txt
git commit -am "feat(main): x"

# GitHub 리포 생성/푸시(실제 사용 시 본인 계정/조직으로)
# gh repo create <owner>/<repo> --public --source=. --push
# PR 만들기
# gh pr create --fill --base main --head feature/f1

# 웹/gh에서: Rebase and merge 선택/수행
# gh pr merge <PR_NUMBER> --rebase
```
- 웹에서 “Rebase and merge” 후, `git pull` 로 main을 가져오면 **merge commit 없이** `feat(f1): a`,`feat(f1): b` 가 `feat(main): x` 뒤에 선형으로 붙어있는 것을 확인할 수 있다.

---

# 3) 충돌 해결 플로우(웹/로컬)

## 3.1 왜 웹에서 바로 안 되나?
- Rebase and merge는 GitHub가 서버 측 rebase를 수행한 뒤 fast-forward 병합을 시도한다.
- **충돌이 나면 서버 측 rebase를 중단**하고 사용자에게 해결을 요구.
- **웹 에디터는 merge 충돌용 UI는 제공하지만**, rebase 충돌은 구조상 제한이 있다.  
  → 보통 **로컬에서 rebase** 후 PR을 업데이트해야 한다.

## 3.2 로컬에서 충돌 해결 절차
```bash
# 1) 최신 base(main) 반영
git fetch origin
git checkout feature/my-pr
git rebase origin/main

# 2) 충돌 시 파일 열어 마커(<<<<<<<, =======, >>>>>>>) 해결
git add <파일들>

# 3) 계속 진행
git rebase --continue

# 4) 반복 충돌 시 rerere 권장
git config --global rerere.enabled true

# 5) rebase 완료 → 원격 업데이트
git push --force-with-lease
```
- PR이 업데이트되며, 상태 체크가 다시 돌고, 이제 “Rebase and merge”가 성공할 수 있다.

## 3.3 빠른 선택(파일 단위 전체 채택)
```bash
git checkout --ours   path/to/file   # 현재(내 PR 브랜치) 버전
git checkout --theirs path/to/file   # base(main) 버전
git add path/to/file
```
- 라인 단위가 아니라 **파일 전체**를 한쪽으로 택한다는 점에 주의.

---

# 4) 서명·검증 정책과의 상호작용

## 4.1 GPG 서명(Require signed commits)
- **커밋 재작성(rebase)** 시 커밋 해시가 바뀌므로, **서명이 깨질 수 있음**.
- 조직에서 “Require signed commits”를 켜둔 경우:
  - PR 브랜치의 커밋들이 전부 **서명된 상태**여야 rebase 후에도 정책을 통과하기 쉽다.
  - 미서명 커밋이 PR에 섞여 있으면 Rebase and merge가 **정책 위반**으로 거부될 수 있다.
- 실무 팁:
  - 로컬에서 커밋 시 기본 서명 활성화
    ```bash
    git config --global user.signingkey <YOUR_KEY_ID>
    git config --global commit.gpgsign true
    ```
  - CI에서 커밋/버전 태깅을 한다면 **서명 키 관리** 전략(프로텍티드 시크릿, keyless 서명 등)을 명확히.

## 4.2 DCO(Developer Certificate of Origin)
- “Signed-off-by” 트레일러를 요구하는 정책이 있다면, squash 또는 rebase로 인한 메시지 변경 시 **트레일러 유지** 필요.
- PR 최종 병합 전에 **메시지 편집 불가**한 Rebase and merge 특성상, **PR 커밋 자체가 트레일러를 포함**하도록 관리하는 것이 안전하다.

---

# 5) Branch Protection Rules(브랜치 보호 규칙)

## 5.1 설정 위치
- 저장소 → Settings → Branches → Branch protection rules → Add rule → `main` 지정

## 5.2 핵심 항목 정리

| 설정 | 효과 |
|---|---|
| Require pull request reviews | 리뷰 승인 없이는 병합 금지 |
| Require status checks to pass | CI(빌드/테스트/린트) 통과 필요 |
| **Require linear history** | 병합 커밋 금지 → **Rebase/Squash만 허용** |
| Require signed commits | GPG 서명 없는 커밋 병합 차단 |
| Restrict who can push | 푸시 권한 제한(보호 강화) |
| Dismiss stale reviews | 새 커밋 푸시 시 기존 승인 무효화 |

## 5.3 Rebase and merge만 허용하고 싶을 때
1) **Require linear history** 활성화  
2) 저장소 Settings → Merge button 설정에서  
   - Allow merge commits **비활성화**  
   - **Allow rebase merges 활성화**  
   - Allow squash merges는 팀 정책에 따라 선택

> Require linear history를 켜면 merge commit이 막히므로, **Rebase and merge** 혹은 **Squash and merge** 만 사용 가능하다.

---

# 6) CI·Auto-merge·Merge queue와의 연동

## 6.1 Status Checks(예: GitHub Actions)
- Rebase and merge는 최종적으로 base 상태가 바뀌므로, **병합 직전**에 다시 상태 체크가 돌 수 있다.
- PR이 길게 대기하는 동안 base가 계속 변하면, **리베이스/리트리거**가 자주 필요할 수 있다.

간단한 예시 워크플로우:
```yaml
# .github/workflows/ci.yml
name: CI
on:
  pull_request:
    branches: [ main ]
jobs:
  test:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
        with:
          fetch-depth: 0   # rebase 시 로그가 필요한 경우 0 권장
      - name: Setup Node
        uses: actions/setup-node@v4
        with:
          node-version: 20
      - run: npm ci
      - run: npm test -- --ci
```

## 6.2 Auto-merge(자동 병합)
- 저장소에서 **Auto-merge**를 켜면, PR이 **모든 조건(리뷰/체크/정책)** 을 충족하는 순간 지정한 방식(여기서는 Rebase and merge)으로 자동 병합.
- 전제: 저장소 설정에서 **Allow rebase merges** 가 켜져 있어야 선택 가능.

## 6.3 Merge queue(대기열)
- 많은 PR이 몰릴 때, **Merge queue** 를 이용하면 순차적으로 병합하며 base 갱신에 따른 **체크 재실행**을 자동으로 조정한다.
- Rebase and merge와 함께 쓰면, 매 PR이 병합 커밋 없이 선형으로 들어오도록 **자동화된 파이프라인** 구축이 가능.

---

# 7) Rebase and merge vs Squash and merge vs Merge commit

| 기준 | Rebase and merge | Squash and merge | Merge commit |
|---|---|---|---|
| 이력 | **선형**, 커밋 유지(해시 재작성) | **선형**, 커밋 1개로 압축 | 분기/병합 흔적 유지 |
| 커밋 수 | 유지(원래 개수) | 1개로 축약 | 원래 + merge 커밋 1개 |
| 커밋 해시 | 변경됨 | 완전히 새 1커밋 | 기존 커밋 유지 |
| 디버깅 | 중간 커밋별 추적 가능 | 기능 단위 1커밋 추적 | 토폴로지 맥락 유지 |
| 서명/트레일러 | 재작성 영향 가능 | 메시지 재작성 필요 | 원본 유지 쉬움 |
| 선호 시나리오 | 깔끔한 선형 + 커밋 단위 추적 | 기능 단위 관리 깔끔 | 토폴로지 맥락 중시 |

---

# 8) 트러블슈팅

| 증상 | 원인 | 해결 |
|---|---|---|
| “Rebase conflicts”로 버튼 비활성 | 서버 측 rebase 충돌 | 로컬에서 `git rebase origin/main` → 해결 → `push --force-with-lease` |
| Require linear history에서 병합 실패 | merge commit 필요 상황 | Rebase 또는 Squash로 재시도, 설정 점검 |
| Require signed commits 실패 | 미서명/서명 파손 | 모든 커밋 서명 후 PR 업데이트, rebase 중 서명 유지 |
| Status checks 통과 안 됨 | CI 실패 | 로컬 수정 → push → 체크 통과 후 재시도 |
| 리뷰가 사라짐/무효화 | Dismiss stale reviews 설정 | 새 커밋 푸시 후 재승인 필요 |
| 강제 푸시 충돌 | 동시 작업/원격 변경 | `--force-with-lease` 사용 습관화, 푸시 전 `fetch` |

---

# 9) 체크리스트

- [ ] 저장소 설정에서 **Allow rebase merges** 활성화
- [ ] **Require linear history** 로 merge commit 차단
- [ ] 팀/조직의 **서명(DCO/GPG) 정책**을 문서화하고 PR 커밋에 준수
- [ ] PR 장기화 시 base 업데이트 빈번 → **Auto-merge / Merge queue** 고려
- [ ] 충돌 시 **로컬 rebase → push --force-with-lease**
- [ ] CI 워크플로우에서 `fetch-depth: 0` 등 rebase 친화 옵션 고려
- [ ] 긴급 롤백 대비 **태그/릴리즈/릴리즈 브랜치** 전략 수립

---

# 10) 명령어 모음(로컬 보조)

```bash
# 최신 base 반영 및 rebase
git fetch origin
git checkout feature/my-pr
git rebase origin/main

# 충돌 해결 루틴
git status
git add <files>
git rebase --continue
git rebase --skip
git rebase --abort

# 빠른 선택(파일 단위)
git checkout --ours   <path>
git checkout --theirs <path>
git add <path>

# 도구/자동화
git config --global rerere.enabled true
git config --global merge.tool meld
git mergetool

# 안전 푸시
git push --force-with-lease

# 관찰
git log --oneline --graph --decorate --all
git range-diff origin/main...HEAD
```

---

# 11) 결론

- **Rebase and merge** 는 PR 병합 시 **병합 커밋 없이 선형 이력**을 유지하는 강력한 방식이다.  
- 다만 **커밋 재작성**으로 인한 서명·정책 상호작용과 **충돌 처리** 부담을 이해해야 한다.  
- 저장소 설정에서 **Require linear history**, **Allow rebase merges** 를 함께 쓰고, **CI/리뷰 정책**을 명확히 하면 **안정적으로 깔끔한 이력**을 유지할 수 있다.  
- 충돌 시에는 로컬 rebase → `--force-with-lease` 로 PR을 갱신하고, 필요하면 **Auto-merge / Merge queue** 로 운영 자동화를 더해 생산성을 높인다.

---

## 참고
- GitHub Docs — Rebase and merge  
  https://docs.github.com/en/pull-requests/collaborating-with-pull-requests/merging-a-pull-request#rebase-and-merge
- GitHub Docs — Branch protection rules  
  https://docs.github.com/en/repositories/configuring-branches-and-merges-in-your-repository/managing-a-branch-protection-rule