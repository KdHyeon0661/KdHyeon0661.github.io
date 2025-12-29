---
layout: post
title: Git - cherry-pick 충돌 해결
date: 2025-01-31 21:20:23 +0900
category: Git
---
# Rebase & Cherry-pick **충돌 해결 가이드**

## 충돌(Conflict)이란 무엇인가?

Git에서 **충돌**은 두 개의 서로 다른 변경 사항이 같은 파일의 동일한 영역(또는 겹치는 영역)을 수정했을 때 발생합니다. 이로 인해 Git이 어떤 변경 사항을 적용해야 할지 자동으로 결정할 수 없게 됩니다. 충돌은 변경 사항을 통합하는 모든 작업—`rebase`, `merge`, `cherry-pick`—에서 발생할 수 있습니다.

---

## 빠른 개요: 기본 충돌 시나리오

### Rebase 충돌 예시
- `main` 브랜치의 `header.html`에는 `<h1>홈페이지</h1>`이 있습니다.
- `feature/banner` 브랜치는 동일한 줄을 `<h1>배너 페이지</h1>`로 수정합니다.
- `feature/banner`에서 `git rebase main`을 실행하면 충돌이 발생합니다.
- **해결**: 파일을 수동으로 편집한 후 `git add`와 `git rebase --continue`를 실행합니다.

### Cherry-pick 충돌 예시
- 현재 `main` 브랜치에 있습니다.
- `dev` 브랜치의 `bcd5678` 커밋을 `git cherry-pick bcd5678` 명령으로 적용하려 합니다.
- `footer.html` 파일의 같은 줄에 서로 다른 내용이 있어 충돌이 발생합니다.
- **해결**: 파일을 수동으로 편집한 후 `git add`와 `git cherry-pick --continue`를 실행합니다.

이 문서는 위의 기본 흐름을 확장하여, **로컬 환경에서 직접 따라 하며 재현할 수 있는 실습 스크립트**와 함께 다양한 충돌 유형과 해결 전략을 상세히 설명합니다.

---

## 재현 가능한 실습 환경 구축 스크립트

아래 스크립트를 새로운 디렉터리에서 실행하면, Rebase와 Cherry-pick 충돌을 순차적으로 실습할 수 있는 완전한 Git 저장소가 생성됩니다.

```bash
# 실습 0: 작업 환경 초기화
rm -rf rbc-lab && mkdir rbc-lab && cd rbc-lab
git init
git config user.name  "lab"
git config user.email "lab@example.com"
git branch -M main

# 공통 초기 파일 생성
echo '<h1>홈페이지</h1>' > header.html
echo '<p>© 2025 MyCompany</p>' > footer.html
git add . && git commit -m "init: header/homepage, footer/company"

# feature/banner 브랜치: header 충돌 생성
git checkout -b feature/banner
# 동일한 줄을 다른 값으로 수정하여 충돌 유발
echo '<h1>배너 페이지</h1>' > header.html
git commit -am "feat(banner): header as banner page"

# main 브랜치: header 동일 줄 수정으로 충돌 확정
git checkout main
echo '<h1>홈페이지 메인</h1>' > header.html
git commit -am "feat(main): header homepage main"

# ---------- Rebase 충돌 테스트 시작 ----------
git checkout feature/banner
git rebase main || true    # 충돌 발생을 위해 실패 허용
# 충돌 발생 확인
git status
# 이 상태에서 아래 §3의 해결 절차를 따라갑니다.

# ---------- Cherry-pick 충돌 준비 ----------
# dev 브랜치: footer 내용을 변경
git checkout -b dev main
echo '<p>© 2025 DevTeam</p>' > footer.html
git commit -am "feat(dev): footer devteam"

# main 브랜치: footer 내용을 다시 수정하여 cherry-pick 시 충돌 유발
git checkout main
echo '<p>© 2025 MyCompany</p>' > footer.html
git commit -am "feat(main): footer mycompany confirm"

# ---------- Cherry-pick 충돌 테스트 ----------
git cherry-pick dev~0 || true
# 충돌 발생 확인
git status
```

> 위 스크립트 실행 후, 이어지는 **Rebase 충돌 해결** (§3) 및 **Cherry-pick 충돌 해결** (§4) 섹션의 단계를 따라 실습을 진행하세요.

---

## Rebase 충돌 해결: 단계별 상세 가이드

### 1. 충돌 감지 및 파일 확인
`git rebase main` 명령을 실행하면 다음과 유사한 메시지가 나타납니다.
```
CONFLICT (content): Merge conflict in header.html
Auto-merging header.html
CONFLICT (content): Merge conflict in header.html
error: could not apply abc1234... feat(banner): header as banner page
Resolve all conflicts manually, mark them as resolved with
"git add/rm <conflicted_files>", then run "git rebase --continue".
You can instead skip this commit with "git rebase --skip".
To abort and get back to the state before "git rebase", run "git rebase --abort".
Could not apply abc1234... feat(banner): header as banner page
```

`git status` 명령으로 충돌이 발생한 파일을 확인합니다.

`header.html` 파일을 열어보면 Git이 삽입한 충돌 마커를 볼 수 있습니다.
```html
<<<<<<< HEAD
<h1>홈페이지 메인</h1>
=======
<h1>배너 페이지</h1>
>>>>>>> abc1234 (feat(banner): header as banner page)
```
- `<<<<<<< HEAD`와 `=======` 사이의 내용은 **새로운 베이스** (`main` 브랜치)의 상태입니다.
- `=======`와 `>>>>>>>` 사이의 내용은 **적용하려는 커밋** (`feature/banner` 브랜치의 변경 사항)입니다.

### 2. 수동으로 통합 및 해결
충돌을 해결하려면, 충돌 마커를 제거하고 최종적으로 원하는 내용으로 파일을 편집합니다.

**예시 1: `feature/banner`의 변경 사항을 유지**
```html
<h1>배너 페이지</h1>
```

**예시 2: 두 변경 사항을 합성**
```html
<h1>홈페이지 메인 | 배너 페이지</h1>
```

### 3. 해결 완료 및 Rebase 계속
파일 편집을 완료한 후, Git에 충돌이 해결되었음을 알리고 Rebase 작업을 계속 진행합니다.
```bash
git add header.html
git rebase --continue
```
Rebase는 여러 커밋을 순차적으로 적용하므로, 다른 커밋에서 또다른 충돌이 발생할 수 있습니다. 동일한 절차(파일 편집 → `git add` → `git rebase --continue`)를 반복합니다.

### 4. 작업 중단 또는 취소 (필요 시)
```bash
git rebase --abort    # Rebase 작업을 완전히 취소하고 시작 전 상태로 복구
git rebase --skip     # 현재 충돌 중인 커밋을 건너뛰고 다음 커밋으로 진행 (주의 사용)
```

---

## Cherry-pick 충돌 해결: 단계별 상세 가이드

### 1. 충돌 감지 및 파일 확인
`git cherry-pick <커밋해시>` 명령 실행 시 충돌이 발생하면, Rebase와 유사한 메시지가 출력됩니다.

`git status`로 충돌 파일을 확인한 후, `footer.html`을 열어 충돌 마커를 확인합니다.
```html
<<<<<<< HEAD
<p>© 2025 MyCompany</p>
=======
<p>© 2025 DevTeam</p>
>>>>>>> bcd5678 (feat(dev): footer devteam)
```

### 2. 수동으로 통합 및 해결
충돌 마커를 제거하고 최종 내용으로 파일을 편집합니다.

**예시 1: 두 내용 병합**
```html
<p>© 2025 MyCompany | DevTeam</p>
```

**예시 2: 한쪽 내용 선택 (예: 회사 표기 유지)**
```html
<p>© 2025 MyCompany</p>
```

### 3. 해결 완료 및 Cherry-pick 계속
```bash
git add footer.html
git cherry-pick --continue
```

### 4. 작업 중단 또는 취소 (필요 시)
```bash
git cherry-pick --abort  # Cherry-pick 작업 완전 취소
git cherry-pick --skip   # 현재 커밋 건너뛰기
```

---

## 다양한 충돌 유형별 해결 전략

### 1. 내용 충돌 (가장 일반적)
**상황**: 두 변경 사항이 동일 파일의 동일한 줄(또는 겹치는 영역)을 수정했습니다.
**해결**: 파일을 열어 충돌 마커를 제거하고, 수동으로 최종 코드를 작성한 후 `git add` 명령으로 스테이징합니다.

### 2. 파일 이름 변경 충돌
**상황**: 한 브랜치에서는 파일 이름을 변경했고(`file.txt` → `main.txt`), 다른 브랜치에서는 원본 파일(`file.txt`)의 내용을 수정했습니다.
**해결**:
  1.  최종적으로 사용할 파일명을 결정합니다(팀 규칙 또는 컨벤션에 따름).
  2.  내용 변경 사항을 새 파일명(또는 유지할 파일명)에 통합합니다.
  3.  `git add`와 `git rm`을 사용하여 최종 상태를 반영합니다.
  ```bash
  # 변경 사항을 확인하는 데 도움이 되는 명령어
  git status
  git diff --name-status --diff-filter=R  # 이름이 변경된(Renamed) 파일 찾기
  git mergetool                          # 시각적 도구로 해결
  ```

### 3. 파일 삭제 vs. 수정 충돌
**상황**: 한 브랜치에서는 파일을 삭제했고, 다른 브랜치에서는 동일한 파일을 수정했습니다.
**해결**:
  - **파일을 유지**하려면: 파일 삭제를 취소하고, 수정 사항을 반영합니다.
    ```bash
    git checkout --theirs path/to/file  # '수정' 사항을 가져옴
    git add path/to/file
    ```
  - **파일 삭제를 유지**하려면: 수정 사항을 버리고 삭제 상태를 유지합니다.
    ```bash
    git checkout --ours path/to/file    # '삭제' 상태를 가져옴 (파일이 워킹 디렉터리에서 사라짐)
    git add path/to/file                # 삭제가 스테이징됨
    ```

### 4. 파일 vs. 디렉터리 충돌
**상황**: 한 쪽에서는 `path/`를 파일로 유지하면서 수정하고, 다른 쪽에서는 `path/`를 디렉터리로 만들고 그 안에 파일을 넣었습니다(또는 그 반대).
**해결**: 프로젝트 구조에 맞는 최종 형태를 결정한 후, 수동으로 파일과 디렉터리를 재구성하고 `git add`/`git rm` 명령으로 상태를 반영합니다.

### 5. 바이너리 파일 충돌 (이미지, 문서 등)
**상황**: `.psd`, `.png` 등 Git이 텍스트로 비교할 수 없는 파일에서 충돌이 발생했습니다.
**해결**: 한쪽 버전을 선택합니다. 협업이 필요할 경우 팀과 논의합니다.
```bash
# '우리'쪽(현재 브랜치) 버전 선택
git checkout --ours assets/logo.png
# 또는 '그들'쪽(가져오는 브랜치) 버전 선택
git checkout --theirs assets/logo.png

git add assets/logo.png
```
**사전 예방**: `.gitattributes` 파일에 특정 파일 형식을 바이너리 또는 병합 불가로 지정할 수 있습니다.
```
*.psd -merge -diff
*.png binary
```

### 6. 서브모듈 포인터 충돌
**상황**: 두 브랜치가 동일한 서브모듈을 서로 다른 커밋 해시로 참조하고 있습니다.
**해결**: 서브모듈 디렉터리로 이동하여 원하는 커밋을 체크아웃한 후, 상위 저장소에서 그 참조를 커밋합니다.
```bash
cd submodules/libA
git checkout <원하는-커밋-해시>
cd ../..
git add submodules/libA
git commit -m "fix(submodule): update libA to <해시>"
```

---

## 고급 도구와 기법

### 시각적 병합 도구 (`mergetool`)
충돌을 GUI 도구를 통해 보다 쉽게 해결할 수 있습니다. 먼저 원하는 도구(예: `meld`, `kdiff3`)를 설정합니다.
```bash
git config --global merge.tool meld
```
충돌 발생 시 다음 명령어로 도구를 실행합니다.
```bash
git mergetool
```

### 자동 해결 기록 (`rerere`)
`rerere`(Reuse Recorded Resolution) 기능은 이전에 해결한 충돌 패턴을 기억했다가, 동일한 충돌이 다시 발생했을 때 자동으로 해결안을 제안합니다. 특히 여러 커밋을 Rebase할 때 시간을 크게 절약할 수 있습니다.
```bash
git config --global rerere.enabled true
```
기능을 활성화한 후, 충돌을 해결하고 `git add`, `git rebase --continue`를 수행하면 해당 해결 방법이 기록됩니다.

### 전체 파일 단위 빠른 선택
특정 파일의 충돌을 한쪽 버전 전체로 해결하려는 경우 사용합니다. **라인 단위 통합보다 덜 정확할 수 있으니 주의가 필요합니다.**
```bash
# 현재 브랜치(ours)의 버전으로 충돌 해결
git checkout --ours path/to/file

# 가져오는 브랜치(theirs)의 버전으로 충돌 해결
git checkout --theirs path/to/file

# 변경 사항을 스테이징
git add path/to/file
```

---

## 예외 상황 및 복구 방법

### 빈 커밋 또는 중복 적용
가져오려는 변경 사항이 이미 현재 브랜치에 적용되어 있어, 결과적으로 **빈(empty)** 커밋이 생성될 수 있습니다.
```bash
# 해당 커밋을 건너뜀
git cherry-pick --skip
# 또는
git rebase --skip

# 만약 빈 커밋으로라도 기록을 남기고 싶다면
git commit --allow-empty -m "docs: note - changes already applied"
```

### 작업 완전 취소
실행 중인 Rebase 또는 Cherry-pick 작업을 완전히 중단하고 작업 시작 전 상태로 돌아갑니다.
```bash
git cherry-pick --abort
git rebase --abort
```

### 비상 복구: Reflog 활용
실수로 중요한 내용을 잃어버린 것 같다면, Git의 `reflog`는 과거의 모든 HEAD 이동 기록을 보관합니다.
```bash
# 과거 상태 기록 조회
git reflog

# 특정 시점(예: HEAD@{3})으로 브랜치를 복구
git switch -c rescued-branch HEAD@{3}
```
**안전한 작업 습관**: 주요 변경 전에는 임시 브랜치나 태그를 생성하여 백업하는 것이 좋습니다.

---

## Rebase 충돌 vs. Cherry-pick 충돌 비교

| 항목 | Rebase 충돌 | Cherry-pick 충돌 |
|---|---|---|
| 발생 원인 | **여러 커밋**을 순차적으로 새로운 베이스에 재적용하는 과정에서 발생 | **선택한 단일(또는 여러) 커밋**의 변경 사항을 현재 위치에 적용할 때 발생 |
| 발생 빈도 | 여러 커밋을 처리하므로 동일한 파일에서 **충돌이 반복 발생**할 수 있음 | 일반적으로 **단발성**인 경우가 많음 |
| 해결 명령어 | `git rebase --continue` / `--abort` / `--skip` | `git cherry-pick --continue` / `--abort` / `--skip` |
| `rerere` 활용도 | 충돌 패턴이 반복될 가능성이 높아 **효과가 매우 큼** | 상대적으로 활용도가 낮을 수 있음 |
| 주요 용도 | 브랜치 히스토리 **정리 및 선형화** | 특정 버그 수정이나 기능을 **선택적으로 다른 브랜치에 반영** |

---

## 확장 실습: 복잡한 충돌 유형 경험하기

### 파일 이름 변경 충돌 재현 스크립트
```bash
# 환경 준비
rm -rf rename-lab && mkdir rename-lab && cd rename-lab
git init
git config user.name  "lab" && git config user.email "lab@example.com"
git branch -M main

echo 'v1' > file.txt
git add . && git commit -m "init"

# branch A: 파일 이름 변경 + 내용 추가
git checkout -b A
git mv file.txt main.txt
echo 'A-change' >> main.txt
git add . && git commit -m "A: rename to main.txt + change"

# main 브랜치: 원본 파일 내용 수정
git checkout main
echo 'main-change' >> file.txt
git commit -am "main: change file.txt"

# A 브랜치를 main 위로 Rebase → 이름 변경과 내용 수정 충돌 발생
git checkout A
git rebase main || true
git status
# 이제 mergetool을 사용하거나 수동으로 파일명과 내용을 통합해 보세요.
```

### 삭제 vs. 수정 충돌 재현 스크립트
```bash
# 환경 준비
rm -rf delmod-lab && mkdir delmod-lab && cd delmod-lab
git init && git branch -M main
echo 'keep-me' > note.md
git add . && git commit -m "init"

# X 브랜치: 파일 삭제
git checkout -b X
git rm note.md
git commit -m "X: delete note.md"

# main 브랜치: 동일 파일 수정
git checkout main
echo 'main edits' >> note.md
git commit -am "main: edit note.md"

# X 브랜치를 main 위로 Rebase → 삭제 vs. 수정 충돌 발생
git checkout X
git rebase main || true
git status
# 전략 선택:
#   파일 유지: git checkout --theirs note.md; git add note.md
#   삭제 유지: git checkout --ours  note.md; git add note.md
# 이후 git rebase --continue 실행
```

---

## 핵심 명령어 요약 (치트 시트)

```bash
### 공통: 상태 확인 ###
git status                  # 충돌 파일 목록 확인
git diff                   # 충돌 세부 사항 보기

### Rebase 작업 ###
git rebase <base-branch>            # Rebase 시작
# 충돌 발생 → 파일 수동 편집
git add <해결된-파일>                # 해결 표시
git rebase --continue               # 계속 진행
git rebase --skip                   # 현재 커밋 건너뛰기
git rebase --abort                  # Rebase 완전 취소

### Cherry-pick 작업 ###
git cherry-pick <커밋-해시>          # 커밋 적용
# 충돌 발생 → 파일 수동 편집
git add <해결된-파일>                # 해결 표시
git cherry-pick --continue          # 계속 진행
git cherry-pick --skip              # 현재 커밋 건너뛰기
git cherry-pick --abort             # Cherry-pick 완전 취소

### 빠른 해결 (파일 전체) ###
git checkout --ours <파일-경로>      # 현재 브랜치 버전 선택
git checkout --theirs <파일-경로>    # 가져오는 브랜치 버전 선택

### 도구 활용 ###
git config --global merge.tool meld  # 시각적 도구 설정
git mergetool                        # 도구 실행
git config --global rerere.enabled true # 자동 해결 기록 활성화

### 복구 ###
git reflog                           # 작업 기록 조회
git reset --hard ORIG_HEAD           # 최종 작업 전 상태로 복구 (주의)
git switch -c rescue-branch HEAD@{N} # 특정 시점에서 새 브랜치 생성
```

---

## 결론

Git에서 Rebase와 Cherry-pick 작업 중 발생하는 충돌은 협업 개발에서 피할 수 없는 부분입니다. 충돌을 두려워하기보다는, 이를 체계적으로 해결하는 방법을 익히는 것이 중요합니다.

**핵심 요약**:
- **Rebase 충돌**은 여러 커밋을 순차적으로 처리하므로 동일한 충돌이 반복될 수 있습니다. `rerere` 기능을 활성화하면 이전 해결 방법을 재사용하여 시간을 절약할 수 있습니다.
- **Cherry-pick 충돌**은 일반적으로 특정 커밋의 변경 사항을 적용할 때 발생하는 단발성 사건입니다.
- 충돌 유형(내용, 파일명 변경, 삭제-수정 등)에 따라 적절한 해결 전략을 선택해야 합니다. 시각적 병합 도구(`mergetool`)는 복잡한 충돌 해결에 큰 도움이 됩니다.
- `--abort` 옵션을 사용한 작업 취소와 `reflog`를 통한 복구 방법을 알고 있으면, 실수에 대한 두려움 없이 더 자신 있게 Git을 사용할 수 있습니다.
- 가장 중요한 것은 **충돌 마커를 꼼꼼히 검토하고, 팀의 코드 컨벤션과 논리를 고려하여 최종 코드를 결정**하는 것입니다. 충돌 해결은 단순한 기술적 문제를 넘어, 팀 간의 커뮤니케이션과 협의가 필요한 과정임을 기억하세요.
