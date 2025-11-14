---
layout: post
title: Git - Git GUI & 시각화 도구
date: 2025-03-02 19:20:23 +0900
category: Git
---
# Git GUI & 시각화 도구 총정리 — CLI 그래프부터 tig, LazyGit, GitLens, SourceTree, GitKraken, GitHub Desktop까지

## 큰그림: 언제 무엇을 쓰나

- **CLI**: 기록이 남는 스크립트·자동화·서버 환경·SSH 원격 환경에 필수.
  → `git log --graph` + 커스텀 pretty format, 필터, `reflog`/`bisect`까지 숙달.
- **터미널 UI**: “커밋·diff·스테이징·blame”을 **한 화면**에서 빠르게 탐색.
  → `tig`, `lazygit`.
- **에디터/IDE 확장**: 코드와 히스토리를 **한 눈**에 결합.
  → VSCode의 `GitLens`, JetBrains IDE 내장 Git log.
- **독립 GUI**: 시각적 브랜치 그래프/리베이스/체리픽/충돌 해결.
  → SourceTree, GitKraken, GitHub Desktop, Fork, Tower 등.

---

## Git 내장 시각화 — `git log`를 “눈”처럼 쓰는 법

### 필수 그래프 옵션

```bash
git log --graph --oneline --decorate --all
```

| 옵션 | 설명 |
|------|------|
| `--graph` | 아스키 그래프(브랜치 분기/병합) |
| `--oneline` | 간결한 한 줄 출력 |
| `--decorate` | 브랜치/태그를 커밋 옆에 표시 |
| `--all` | 모든 참조(브랜치/태그)를 출력 |

예시:

```text
* a1b2c3 (HEAD -> main, origin/main) Fix typo
| * 4d5e6f (feature/login) Add login form
|/
* 9a8b7c Initial commit
```

### 가독성 향상: pretty format·색상

```bash
git log --graph --all \
  --pretty=format:'%C(yellow)%h%Creset %Cgreen%ad%Creset %C(red)%d%Creset %s %C(cyan)<%an>%Creset' \
  --date=short
```

설명:
- `%h`: 짧은 해시, `%ad`: 날짜, `%d`: 데코(브랜치/태그), `%s`: 제목, `%an`: 작성자
- `%C(color)`로 부분 색상 지정

**권장 alias**:

```bash
git config --global alias.lg "log --graph --decorate --all --date=short --pretty=format:'%C(yellow)%h%Creset %Cgreen%ad%Creset %C(red)%d%Creset %s %C(cyan)<%an>%Creset'"
```

이후:

```bash
git lg
```

### 흐름 파악 스킬

- **병합만 보기**: 대형 기능 릴리스 트래킹
  ```bash
  git log --merges --oneline --decorate --graph
  ```
- **첫 부모만 따라가기**(릴리스 라인/릴리스 노트용):
  ```bash
  git log --first-parent --oneline --graph
  ```
- **파일·디렉터리 국소화**(원인 추적):
  ```bash
  git log --graph --oneline -- path/to/file_or_dir
  ```
- **PR 범위만 보기**:
  ```bash
  git log --graph --oneline --decorate origin/main..HEAD
  ```

### 리베이스/체리픽/리셋 전 “드라이런”으로 확인

- 리베이스 시작 전 공통 조상:
  ```bash
  git merge-base HEAD origin/main
  ```
- 적용 대상 커밋만:
  ```bash
  git log --oneline --graph --decorate $(git merge-base HEAD origin/main)..HEAD
  ```

---

## `tig` — 터미널 기반 인터랙티브 Git 탐색기

### 설치

- macOS: `brew install tig`
- Ubuntu/Debian: `sudo apt install tig`
- Windows: WSL 또는 패키지 매니저(예: scoop)로 설치

### 핵심 명령

```bash
tig                # 전체 로그 뷰
tig status         # 변경 파일/스테이징 인터랙션
tig blame <file>   # 라인별 blame
tig <branch>       # 특정 브랜치 히스토리
tig refs           # 참조(브랜치/태그) 보기
```

### 자주 쓰는 키맵(기본)

| 키 | 기능 |
|----|------|
| `j/k` | 이동 |
| `Enter` | 커밋 상세/파일 diff 진입 |
| `q` | 나가기 |
| `/` | 검색 |
| `s` | 스테이지/언스테이지(상황에 따라) |
| `u` | 상위 뷰로 |
| `h` | 도움말 |

### 워크플로 예시: “파일→커밋→diff→원인 추적”

1) `tig blame src/app.js`
2) 문제 라인 선택 후 `Enter` → 해당 커밋으로 이동
3) `Enter`로 diff 확인 → 관련 변경 주변 커밋을 상하로 스캔
4) `u`로 상위로 돌아가 다른 파일도 연쇄 탐색

### tig 설정(선택)

`~/.tigrc` 예시:

```ini
set main-view = id,date,author,refs,commit-title
set show-changes = yes
bind main J move-last-line
bind main K move-first-line
```

---

## `lazygit` — 빠른 조작을 위한 터미널 UI

### 설치

- macOS: `brew install jesseduffield/lazygit/lazygit`
- Linux: 패키지 또는 릴리스 바이너리
- Windows: scoop/choco 또는 릴리스 바이너리

### 특징

- 변경 파일, 스테이징 청크, 커밋, 브랜치, 로그, 리베이스, 체리픽을 **패널**로 즉시 조작
- 충돌 해결 시 좌/우/결과 창을 이동하며 선택 처리
- 작업 속도가 매우 빠름(마우스 불필요)

---

## VSCode GitLens — 코드-히스토리 융합 시각화

### 설치

```bash
code --install-extension eamodio.gitlens
```

### 핵심 기능

- **라인 히스토리**: 해당 줄의 작성자/시간, 커밋 툴팁
- **File History / Line History** 패널: 파일별 변경 타임라인
- **Commit Graph**(GitLens+): 브랜치 그래프 UI
- **Blame/Annotate**: 에디터에 실시간 주석
- **PR 연동**: GitHub/GL/BB와 연결해 리뷰/체크
- **Rebase/Cherry-pick** 일부 UI 지원(버전에 따라 가능 범위 상이)

### 워크플로 예시: “리팩터링 영향 범위 확인”

1) 파일을 열고, 상단/사이드바의 **File History** 열기
2) 특정 커밋 클릭 → diff/주석으로 영향 라인 확인
3) Commit Graph에서 브랜치 분기점 맥락 파악
4) 라인 히스토리로 **“이 라인을 왜, 누가, 언제 바꿨는지”** 즉시 확인

---

## 독립 GUI 툴 비교

### SourceTree (Atlassian)

| 항목 | 내용 |
|------|------|
| 플랫폼 | Windows / macOS |
| 장점 | 무료, 브랜치 그래프/체리픽/리베이스/충돌 해결 UI가 익숙 |
| 단점 | 대형 저장소에서 성능 이슈 가능 |
| 적합 | 입문~중급. GUI로 Git 개념 습득에 유리 |

#### 자주 쓰는 흐름

- 좌측 브랜치/태그·우측 그래프에서 커밋 선택 → 마우스 우클릭 **체리픽/리버트/리셋**
- 충돌 시 **Resolve Conflicts** 패널에서 파일별 병합 도구 호출
- **Interactive Rebase** 대화상자에서 순서 변경·스쿼시

### GitKraken

| 항목 | 내용 |
|------|------|
| 플랫폼 | Windows / macOS / Linux |
| 장점 | 시각화가 뛰어남, 멀티-리모트/이슈/PR 통합, 팀 플로 지원 |
| 단점 | 팀 기능 유료 |
| 적합 | 협업, 다수 리모트/이슈 트래킹, 직관적 rebase/merge |

#### 특징

- 그래프에서 드래그&드롭 리베이스/체리픽
- PR 보기·리뷰·체크 상태를 통합 패널로 관리

### GitHub Desktop

| 항목 | 내용 |
|------|------|
| 플랫폼 | Windows / macOS |
| 장점 | GitHub 액션/PR과의 통합이 쉬움, 간단한 UI |
| 단점 | 고급 Git 기능(복잡 rebase 등)은 제한적 |
| 적합 | GitHub 중심 개인/소규모 팀, 쉬운 온보딩 |

### Fork, Tower (프로급 GUI)

| 항목 | Fork | Tower |
|------|------|------|
| 플랫폼 | macOS/Windows | macOS/Windows |
| 장점 | 빠른 성능, 명쾌한 UI, rebase/충돌/스테이징 조각 단위 편리 | 기업용 품질, GPG/서브모듈/LFS/워크트리 등 고급 지원 |
| 라이선스 | 유료(합리적) | 유료(엔터프라이즈 친화) |
| 적합 | 고급 사용자/대형 리포 | 엔터프라이즈·보안/정책 요구 높은 팀 |

---

## JetBrains IDE(예: IntelliJ/Rider/WebStorm)의 Git Log

- **Version Control** 탭 → **Log**: 브랜치 그래프·검색·필터
- **Rebase/Cherry-pick/Merge**를 IDE UI로 실행
- 변경 파일과 **in-place conflict resolver**(3-way merge)가 강력

---

## 시각화로 해결하는 실무 케이스

### “PR이 왜 커졌는지”를 1분 내 파악

- CLI:
  ```bash
  git log --graph --oneline --decorate origin/main..HEAD
  ```
- GitLens Commit Graph 또는 GitKraken 그래프에서 **분기점**·**병합 지점** 확인
- 파일/폴더 필터로 변경 범위를 좁히고, 커밋 단위로 **스쿼시 후보**를 표기

### 충돌 해결 전략(시각화 중심)

1) 그래프에서 **공통 조상** 커밋과 양 브랜치 방향을 식별
2) GUI의 3-way 병합 도구로 **의도(ours/theirs)**를 코드 맥락과 함께 선택
3) 큰 충돌이면 **폴더 단위**로 나누어 여러 번 커밋 → PR 리뷰의 심리적 부하 감소

### 리베이스로 선형 이력 유지

- VSCode GitLens 또는 GitKraken의 rebase UI로 **순서 조정·스쿼시**
- CLI 대비 시각적 확인이 쉬워 **실수(대상 범위 과대)**를 줄임

---

## 대규모 저장소 성능 최적화(시각화/탐색을 빠르게)

- **부분 클론/희소 체크아웃**:
  ```bash
  git clone --filter=blob:none --no-checkout <repo>
  git sparse-checkout init --cone
  git sparse-checkout set src/ apps/service-a/
  ```
- **최신만 클론(shallow)**:
  ```bash
  git clone --depth 50 <repo>
  ```
- **로그 제한**(기간/작성자/경로):
  ```bash
  git log --since=2025-09-01 --author="kim" -- path/to/dir
  ```
- GUI에서 **폴더 범위**만 인덱싱(가능한 툴 옵션 활용)

---

## 서브모듈/서브트리/LFS를 시각화에서 다루는 팁

- 서브모듈 그래프는 **서브모듈 디렉토리 내부에서** 별도로 log/tig/GUI 열기
- LFS 포인터(diff 불가)에 주의 → GUI에서 바이너리 비교 도구 설정
- 서브트리는 메인 그래프에 섞이므로 **폴더 기준**으로 로그 필터링

---

## 팀 정책·교육 플로(현실적 가이드)

1) **CLI 기초 교육**(30분): `commit/add/reset/log/branch/merge/rebase/cherry-pick`
2) **시각화 툴 온보딩**(20분): GitLens 또는 GitKraken 중 팀 표준 하나
3) **PR 체크리스트**(10분):
   - 그래프에서 분기/병합 경로 검사
   - 파일별 diff 스크롤(대형 diff는 폴더 단위로 나눔)
   - 커밋 메시지 규칙(Conventional Commits)·스쿼시 여부
4) **브랜치 보호 규칙**: Require linear history, Require reviews, CI 필수
5) **문제 복구 교육**: `reflog`, GUI의 “reset to…”, “restore” 데모

---

## 치트시트

### CLI 그래프

```bash
# 전체 브랜치 그래프(짧은 형태)

git log --graph --oneline --decorate --all

# 병합만(릴리스 라인 추적)

git log --merges --oneline --graph

# 첫 부모만(릴리스 노트)

git log --first-parent --oneline --graph

# PR 범위만

git log --oneline --graph origin/main..HEAD

# 파일/폴더 국소화

git log --oneline --graph -- path/to/dir
```

### tig

```bash
tig                 # 로그
tig status          # 스테이징/커밋
tig blame file      # blame
```

키: `j/k` 이동, `Enter` 상세, `q` 종료, `/` 검색, `s` 스테이징

### GitLens 핵심

- File History / Line History
- Commit Graph(확장 플랜)
- Blame in-editor
- PR 패널 연동

### GUI 공통 조작

- 그래프에서 커밋 우클릭: **Cherry-pick / Revert / Reset / Create Branch**
- 드래그&드롭 리베이스(GitKraken/Fork/Tower 등 지원)
- Conflict Resolver(3-way 머지): ours/theirs 선택 + 수동 편집

---

## 실습 시나리오(끝까지 따라 해보기)

### 시나리오 A: CLI + tig로 브랜치 탐색

```bash
# 예시 리포 클론

git clone https://example.com/repo.git
cd repo

# 그래프 한눈

git lg

# 특정 폴더만

git log --graph --oneline -- path/to/module

# 변경 몰아보기

tig
# -> j/k로 이동, Enter로 diff, u로 상위

```

### 시나리오 B: GitLens로 “이 라인”의 역사 보기

1) VSCode로 프로젝트 열기 → GitLens 설치
2) 파일에서 커서 위치 → 사이드바 **Line History**
3) 변경 커밋 리스트에서 클릭 → diff 탭 확인
4) Commit Graph(사용 가능 시)로 브랜치 분기 확인

### 시나리오 C: GUI로 충돌 해결

1) 기능 브랜치에서 `git pull --rebase origin main`(CLI)
2) 충돌 발생 → GUI(예: SourceTree/GitKraken)로 열기
3) Conflict 패널에서 파일 선택 → 내장 3-way 머지
4) 완료 후 **Continue rebase** / **Commit** → 푸시

---

## 도구별 요약 비교

| 도구/명령 | 종류 | 강점 | 약점 | 추천 대상 |
|-----------|------|------|------|-----------|
| `git log --graph` | CLI | 가볍고 빠름, 자동화 친화 | 텍스트 난이도 | 모든 개발자(필수) |
| `tig` | 터미널 UI | diff/히스토리/스테이징 일체형 | 키맵 학습 | CLI 선호자 |
| `lazygit` | 터미널 UI | 초고속 조작/충돌/청크 스테이징 | GUI 편의 덜함 | 조작 속도 중시 |
| GitLens | VSCode 확장 | 코드-히스토리 결합, 라인단 추적 | VSCode 의존 | VSCode 사용자 |
| SourceTree | GUI | 입문 친화, 그래프 명료 | 대형 리포 성능 | 입문~중급 |
| GitKraken | GUI | 그래프/협업 통합, rebase 편리 | 팀 유료 | 협업/시각화 중시 |
| GitHub Desktop | GUI | GitHub 연동 간편 | 고급 기능 제한 | 개인/소규모 |
| Fork/Tower | GUI(유료) | 성능/기능성/프로급 | 비용 | 고급/엔터프라이즈 |

---

## 결론

- **CLI 시각화는 필수 근력**: `git log --graph --decorate --all`을 몸에 익혀라.
- **터미널 UI(tig/lazygit)**로 탐색 속도를 끌어올려라.
- **에디터 통합(GitLens)**로 “코드-히스토리”를 한 화면에서 판단하라.
- **독립 GUI**로 리베이스/체리픽/충돌 해결을 시각적으로 안전하게 관리하라.
- 대형 리포에선 **희소 체크아웃/부분 클론** 등 성능 전략을 병행하라.
- 팀 레벨에선 **브랜치 보호/리뷰/CI**와 결합해 **일관된 워크플로**를 구축하라.

---

## 참고 링크

- Tig: https://jonas.github.io/tig/
- SourceTree: https://www.sourcetreeapp.com/
- GitKraken: https://www.gitkraken.com/
- GitLens: https://marketplace.visualstudio.com/items?itemName=eamodio.gitlens
- Git log 문서: https://git-scm.com/docs/git-log
