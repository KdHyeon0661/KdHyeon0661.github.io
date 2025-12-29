---
layout: post
title: Git - rebase and merge & 자동 병합 정책 설정
date: 2025-02-06 19:20:23 +0900
category: Git
---
# GitHub의 `Rebase and merge`

## 개요: 무엇이며, 왜 사용하는가?

- **Rebase and merge**: Pull Request(PR)의 커밋들을 **베이스 브랜치(예: `main`)의 최신 커밋 뒤에 순서대로 재적용**하는 방식으로, 별도의 병합 커밋 없이 **선형적이고 깔끔한 히스토리**를 만듭니다.
- **주요 장점**: 히스토리가 단순해져 `git log --oneline --graph` 명령의 결과가 명확하며, 버그 탐색 도구인 `git bisect` 사용 시 흐름이 직관적입니다.
- **주의 사항**: 커밋이 재작성되므로 **커밋 해시가 변경**됩니다. 충돌 발생 시 웹 인터페이스에서 직접 해결하기 어려울 수 있으며, 로컬에서 Rebase 작업이 필요할 수 있습니다. 또한 GPG 서명이나 DCO(Developer Certificate of Origin) 정책과의 호환성을 확인해야 합니다.
- **팀 정책 설정**: 저장소 설정에서 **Allow rebase merges**를 활성화하고, **Require linear history** 규칙을 설정하면 병합 커밋을 차단하여 선형 히스토리를 강제할 수 있습니다.

---

# 개념과 작동 원리

## GitHub의 세 가지 병합 방식

1.  **Create a merge commit** (병합 커밋 생성)
2.  **Squash and merge** (스쿼시 병합)
3.  **Rebase and merge** (리베이스 병합) - 본 문서의 주제

## Rebase and merge란 정확히 무엇인가?

이 방식은 PR 브랜치의 각 커밋을 베이스 브랜치의 최신 지점에 다시 적용(Reapply)합니다. GitHub는 서버 측에서 이 리베이스 작업을 수행하여, 베이스 브랜치가 PR 브랜치의 변경 사항을 **Fast-forward**로 받아들일 수 있는 상태로 만든 후 병합을 완료합니다. 결과적으로 **별도의 병합 커밋이 생성되지 않고**, 히스토리가 **직선형**으로 유지됩니다.

### 시각적 예시

```
main 브랜치:    A---B---C
                       \
feature 브랜치:          D---E

# Rebase and merge 실행 후 (개념적 결과):
main 브랜치:    A---B---C---D'---E'
```
- `D'`와 `E'`는 원본 커밋 `D`, `E`의 내용은 그대로이지만, 부모 커밋이 변경되어 **재작성된 새로운 커밋**입니다. 따라서 커밋 해시가 달라집니다.

## 내부 동작 방식 (핵심 이해)

- GitHub는 PR을 병합하기 직전에 **서버 측에서 자동으로 리베이스**를 수행합니다.
- 이 과정에서 **충돌이 발생하지 않고**, 모든 **브랜치 보호 규칙**(필수 CI 통과, 리뷰 승인, 커밋 서명, 선형 히스토리 요구 등)이 충족되면, 베이스 브랜치를 Fast-forward 방식으로 업데이트합니다.
- **충돌이 발생하는 경우**:
  - PR 페이지에 "Rebase conflicts"와 같은 메시지가 표시됩니다.
  - GitHub 웹 에디터는 일반적인 Merge 충돌 해결에는 유용하지만, Rebase 과정에서의 복잡한 충돌을 해결하기에는 제한적일 수 있습니다.
  - 대부분의 경우, **로컬 환경에서 직접 리베이스를 수행하고 충돌을 해결**한 후, PR 브랜치를 강제 푸시(`--force-with-lease`)하여 업데이트해야 합니다.

---

# 실전 사용법 (웹 및 CLI)과 재현 예제

## 웹 인터페이스에서의 절차

1.  PR 페이지 하단의 병합 버튼 옆 드롭다운 메뉴에서 **Rebase and merge**를 선택합니다.
2.  필수 상태 확인(Required status checks)과 리뷰어 승인 조건이 모두 충족되었는지 확인합니다.
3.  **Confirm rebase and merge** 버튼을 클릭합니다.
4.  성공하면, 베이스 브랜치(예: `main`)에 병합 커밋 없이 PR의 커밋들이 선형으로 추가됩니다.

## GitHub CLI(`gh`)를 사용한 예시

```bash
# PR 번호 123을 Rebase 방식으로 병합
gh pr merge 123 --rebase
```

## 로컬에서 PR 생성부터 병합까지 재현 실습

```bash
# 실습 환경 준비
rm -rf rebase-merge-lab && mkdir rebase-merge-lab && cd rebase-merge-lab
git init
git config user.name  "lab"
git config user.email "lab@example.com"
git branch -M main
echo "v1" > app.txt
git add . && git commit -m "chore: init v1"

# 기능 브랜치 생성 및 여러 커밋 추가
git checkout -b feature/f1
echo "f1-a" >> app.txt
git commit -am "feat(f1): a"
echo "f1-b" >> app.txt
git commit -am "feat(f1): b"

# main 브랜치에서도 독립적인 변경 사항 생성
git checkout main
echo "main-x" >> app.txt
git commit -am "feat(main): x"

# (실제 운영 시) GitHub에 저장소 생성 및 푸시
# gh repo create <owner>/<repo> --public --source=. --push

# (실제 운영 시) PR 생성
# gh pr create --fill --base main --head feature/f1

# (실제 운영 시) 웹 또는 CLI에서 'Rebase and merge' 실행
# gh pr merge <PR_NUMBER> --rebase
```
- 웹에서 "Rebase and merge"를 수행한 후, 로컬에서 `git pull` 명령으로 `main` 브랜치를 업데이트하면, `feat(main): x` 커밋 뒤에 `feat(f1): a`와 `feat(f1): b` 커밋이 **병합 커밋 없이 선형으로 연결**된 것을 확인할 수 있습니다.

---

# 충돌 해결 플로우 (웹 vs. 로컬)

## 웹에서 즉시 해결이 어려운 이유

Rebase and merge는 GitHub 서버가 리베이스를 시도한 후 Fast-forward 병합을 수행하는 방식입니다. **충돌이 발생하면 서버 측 리베이스가 중단**되고, 사용자에게 해결을 요청합니다. GitHub 웹 에디터는 Merge 시의 충돌 해결에 최적화되어 있지만, 여러 커밋을 순차적으로 재적용하는 Rebase의 특성상 발생하는 복잡한 충돌을 처리하기에는 한계가 있습니다. 따라서 대부분 **로컬에서 리베이스를 수행하고 충돌을 해결한 후 PR을 업데이트**해야 합니다.

## 로컬에서 충돌 해결하는 표준 절차

```bash
# 1. 최신 원격 정보 동기화 및 PR 브랜치로 이동
git fetch origin
git checkout feature/my-pr

# 2. 베이스 브랜치(main)를 기준으로 리베이스 실행 (충돌 발생)
git rebase origin/main

# 3. 충돌이 발생한 파일을 편집기로 열어 수동 해결
#    파일 내의 <<<<<<<, =======, >>>>>>> 마커를 제거하고 최종 코드를 작성합니다.

# 4. 해결 완료 표시 및 리베이스 계속
git add <해결된-파일-경로>
git rebase --continue

# 5. (선택) 반복되는 충돌 패턴을 자동으로 해결하기 위해 rerere 기능 활성화
git config --global rerere.enabled true

# 6. 리베이스 완료 후, 변경된 히스토리를 원격 브랜치에 안전하게 푸시
git push --force-with-lease
```
- PR 브랜치가 업데이트되면 GitHub의 상태 확인(CI)이 다시 실행됩니다. 모든 조건이 충족되면 이제 "Rebase and merge" 버튼을 성공적으로 사용할 수 있습니다.

## 파일 전체를 한쪽 버전으로 빠르게 선택

특정 파일의 충돌을 라인 단위가 아닌 **파일 전체를 한쪽 버전으로 해결**하려는 경우 사용합니다. 주의하여 사용하세요.
```bash
# 현재 체크아웃된 브랜치(PR 브랜치)의 버전으로 선택
git checkout --ours   path/to/file

# 베이스 브랜치(예: main)의 버전으로 선택
git checkout --theirs path/to/file

# 선택 사항을 스테이징
git add path/to/file
```

---

# 서명 및 검증 정책과의 상호작용

## GPG 서명 요구 사항 (Require signed commits)

- **커밋 재작성의 영향**: 리베이스는 커밋 해시를 변경합니다. 이로 인해 원본 커밋에 부착된 **GPG 서명이 유효하지 않게 될 수 있습니다**.
- **조직 정책 하에서**: "Require signed commits"가 활성화된 저장소에서는 PR의 모든 커밋이 **서명된 상태**여야 리베이스 후에도 정책을 통과할 가능성이 높습니다. 서명되지 않은 커밋이 섞여 있으면 병합이 거부될 수 있습니다.
- **실무 팁**:
  - 개발자별 로컬 Git 설정에서 기본 서명을 활성화합니다.
    ```bash
    git config --global user.signingkey <YOUR_KEY_ID>
    git config --global commit.gpgsign true
    ```
  - CI/CD 파이프라인에서 커밋이나 태그에 서명하는 경우, 서명 키의 안전한 관리(예: GitHub Protected Secrets, Keyless 서명) 전략을 수립해야 합니다.

## DCO (Developer Certificate of Origin)

- DCO 정책은 커밋 메시지에 "Signed-off-by" 줄을 포함하도록 요구합니다.
- 리베이스나 스쿼시로 인해 커밋 메시지가 수정될 때 이 **트레일러가 유지되도록 주의**해야 합니다.
- Rebase and merge는 병합 시 메시지를 편집할 수 없는 방식이므로, **PR을 제출하기 전에 각 커밋이 DCO 서명을 포함하고 있는지 확인**하는 것이 안전합니다.

---

# 브랜치 보호 규칙 (Branch Protection Rules)

## 설정 위치

저장소의 `Settings` → `Branches` → `Branch protection rules` → `Add rule`에서 대상 브랜치(예: `main`)를 지정하여 규칙을 설정합니다.

## Rebase and merge와 관련된 핵심 규칙

| 설정 | 효과 |
|---|---|
| Require pull request reviews | 리뷰어 승인 없이는 병합 불가 |
| Require status checks to pass | CI(빌드, 테스트, 린트)가 성공해야 함 |
| **Require linear history** | **병합 커밋 생성을 차단** → **Rebase 또는 Squash 방식만 허용** |
| Require signed commits | GPG 서명되지 않은 커밋의 병합 차단 |
| Restrict who can push | 지정된 사용자/팀만 해당 브랜치에 푸시 가능 |
| Dismiss stale reviews | 새 커밋이 푸시되면 기존 승인을 자동 무효화 |

## Rebase and merge만 허용하도록 설정하려면

1.  **Require linear history** 규칙을 활성화합니다. 이는 병합 커밋을 근본적으로 방지합니다.
2.  저장소의 `Settings` → `General` → `Merge button` 섹션에서 다음을 설정합니다.
    - **Allow merge commits**: **비활성화**
    - **Allow rebase merges**: **활성화**
    - **Allow squash merges**: 팀 정책에 따라 선택적 활성화

> `Require linear history`를 켜면 `Create a merge commit` 방식이 차단되므로, 자연스럽게 **Rebase and merge** 또는 **Squash and merge** 방식만 사용하게 됩니다.

---

# CI/CD, 자동 병합(Auto-merge), 병합 대기열(Merge Queue)과의 연동

## 상태 확인 (Status Checks)과의 동작

- Rebase and merge는 최종적으로 베이스 브랜치의 상태를 변경하므로, 병합 직전에 **필요한 상태 확인(CI 작업)이 다시 실행**될 수 있습니다.
- PR이 오랜 시간 대기하는 동안 베이스 브랜치가 많이 변경되었다면, **리베이스로 인해 CI가 자주 재실행**되어야 할 수 있습니다.

간단한 CI 워크플로우 예시:
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
          fetch-depth: 0   # 리베이스 시 전체 히스토리가 필요할 수 있으므로 0으로 설정
      - name: Setup Node
        uses: actions/setup-node@v4
        with:
          node-version: 20
      - run: npm ci
      - run: npm test -- --ci
```

## 자동 병합 (Auto-merge)

- 저장소에서 Auto-merge 기능을 활성화하면, PR이 **모든 필수 조건(리뷰 승인, CI 통과, 보호 규칙 준수)**을 만족하는 순간 **지정된 병합 방식(여기서는 Rebase and merge)**으로 자동으로 병합됩니다.
- **전제 조건**: 저장소 설정에서 **Allow rebase merges**가 반드시 활성화되어 있어야 합니다.

## 병합 대기열 (Merge Queue)

- 많은 PR이 동시에 제출되는 프로젝트에서 **Merge Queue**를 사용하면 PR들을 순차적으로 병합하면서, 각 병합 후 변경된 베이스 브랜치에 대한 **CI 검증을 자동으로 재실행**합니다.
- Rebase and merge 방식과 결합하면, 모든 PR이 병합 커밋 없이 선형 히스토리로 통합되는 **완전 자동화된 병합 파이프라인**을 구축할 수 있습니다.

---

# 병합 방식 비교: Rebase and merge vs. Squash and merge vs. Merge commit

| 기준 | Rebase and merge | Squash and merge | Merge commit |
|---|---|---|---|
| 히스토리 형태 | **선형**, 개별 커밋 유지(해시 재작성) | **선형**, 모든 변경을 하나의 새 커밋으로 압축 | **비선형**, 분기 및 병합 흔적 유지 |
| 커밋 수 | 원본 PR의 커밋 개수 유지 | 항상 1개의 새 커밋 | 원본 커밋 + 1개의 병합 커밋 |
| 커밋 해시 | 변경됨 (재작성) | 완전히 새로운 해시 | 원본 커밋 해시 유지 |
| 디버깅 용이성 | 각 중간 커밋별로 변경 사항 추적 가능 | 기능 단위로 하나의 커밋만 추적 | 전체 분기 토폴로지와 맥락 유지 |
| 서명/트레일러 | 재작성으로 인해 서명이 깨질 수 있음 | 새 커밋 메시지에 서명 필요 | 원본 커밋의 서명 유지 쉬움 |
| 선호 시나리오 | 깔끔한 선형 히스토리 유지 + 커밋 단위 추적 필요 시 | 기능 단위로 히스토리를 압축해 관리하고 싶을 때 | 브랜치 병합의 전체 맥락과 역사를 보존해야 할 때 |

---

# 문제 해결 (트러블슈팅)

| 증상 | 가능한 원인 | 해결 방법 |
|---|---|---|
| "Rebase conflicts" 메시지와 함께 병합 버튼 비활성화 | 서버 측 리베이스 중 충돌 발생 | 로컬에서 `git rebase origin/main` 실행 → 충돌 해결 → `git push --force-with-lease`로 PR 업데이트 |
| `Require linear history` 설정 시 병합 실패 | 병합 커밋을 생성하려고 시도함 | Rebase and merge 또는 Squash and merge 방식으로 다시 시도. 저장소 병합 버튼 설정 확인 |
| `Require signed commits` 정책으로 인한 실패 | 커밋 서명이 없거나 리베이스로 서명이 무효화됨 | PR의 모든 커밋에 서명이 되어 있는지 확인. 로컬에서 리베이스 시 서명 유지 설정 점검 |
| 상태 확인(Status checks)이 통과하지 않음 | CI 테스트 실패 또는 조건 미충족 | 로컬에서 문제 수정 → 커밋 및 푸시 → CI 통과 후 병합 재시도 |
| 리뷰어 승인이 사라짐 | `Dismiss stale reviews` 설정이 활성화됨 | PR 브랜치에 새 커밋을 푸시한 후, 리뷰어에게 재승인 요청 |
| 강제 푸시 시 충돌 | 다른 협업자가 동시에 PR 브랜치를 수정함 | `git fetch origin`으로 최신 상태 확인 후, `--force-with-lease` 옵션을 사용해 안전하게 푸시 |

---

# 주요 명령어 모음 (로컬 작업 보조)

```bash
### 베이스 브랜치 최신화 및 리베이스 ###
git fetch origin
git checkout feature/my-pr
git rebase origin/main

### 충돌 해결 루틴 ###
git status                               # 충돌 파일 확인
git add <해결된-파일>                     # 해결 표시
git rebase --continue                    # 계속 진행
git rebase --skip                        # 현재 커밋 건너뛰기 (주의)
git rebase --abort                       # 리베이스 완전 취소

### 파일 단위 빠른 선택 ###
git checkout --ours   <파일-경로>         # 현재 브랜치 버전 선택
git checkout --theirs <파일-경로>         # 베이스 브랜치 버전 선택
git add <파일-경로>

### 도구 및 자동화 ###
git config --global rerere.enabled true  # 충돌 해결 패턴 재사용 활성화
git config --global merge.tool meld      # 시각적 병합 도구 설정
git mergetool                            # 설정된 도구 실행

### 안전한 원격 업데이트 ###
git push --force-with-lease

### 히스토리 관찰 ###
git log --oneline --graph --decorate --all  # 전체 히스토리 시각화
git range-diff origin/main...HEAD          # 리베이스 전후 변경 비교
```

---

## 결론

GitHub의 **Rebase and merge**는 병합 커밋 없이 깔끔한 선형 히스토리를 유지하고 싶은 팀에게 매우 유용한 기능입니다. 이 방식을 통해 `git log` 출력이 단순해지고, 버그 추적이나 코드 리뷰가 보다 직관적으로 이루어질 수 있습니다.

그러나 이점과 함께 고려해야 할 점들이 있습니다. 커밋 재작성은 **GPG 서명 무효화**를 초래할 수 있으며, 충돌 발생 시 **로컬에서의 수동 해결**이 필요할 수 있습니다. 또한, 모든 커밋이 서명되었는지, DCO 요구사항을 충족하는지 주의 깊게 확인해야 합니다.

이를 성공적으로 도입하기 위해서는 저장소 설정에서 **Allow rebase merges**를 활성화하고, **Require linear history** 규칙을 함께 적용하여 병합 커밋 생성을 근본적으로 방지하는 것이 효과적입니다. CI/CD 파이프라인과의 통합, Auto-merge 및 Merge Queue와 같은 고급 기능을 활용하면 병합 프로세스의 자동화와 효율성을 극대화할 수 있습니다.

궁극적으로, Rebase and merge는 팀의 워크플로우와 코드 품질 목표에 부합하는 도구입니다. 팀 내에서 이 방식의 장단점을 충분히 논의하고, 필요한 가이드라인과 정책을 마련한 후 도입한다면, 보다 체계적이고 생산적인 협업 환경을 조성하는 데 기여할 것입니다.
