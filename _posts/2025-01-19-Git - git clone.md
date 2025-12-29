---
layout: post
title: Git - git clone
date: 2025-01-19 19:20:23 +0900
category: Git
---
# Git **clone**(클론)

## `git clone`이란?

`git clone`은 **원격 저장소(Remote Repository)**의 모든 이력(커밋, 브랜치, 태그)과 파일을 **로컬(Local) 저장소**로 복제하는 Git의 기본 명령입니다. 원격 저장소의 전체 복사본을 로컬에 생성하는 것이 핵심입니다.

```bash
git clone <원격 저장소 주소>
```

명령을 실행하면 `.git` 디렉터리(버전 관리 데이터)와 함께 워킹 트리(실제 파일)가 생성됩니다. 기본적으로 원격 저장소는 `origin`이라는 이름으로 등록됩니다(`git remote -v`로 확인 가능).

---

## 가장 기본적인 클론 방법

### HTTPS 방식
```bash
git clone https://github.com/username/repo-name.git
```

### SSH 방식
```bash
git clone git@github.com:username/repo-name.git
```

위 명령어들은 현재 디렉터리에 `repo-name/` 폴더를 생성합니다. 다른 이름의 디렉터리로 복제하려면 마지막에 원하는 폴더 이름을 추가하면 됩니다.
```bash
git clone https://github.com/username/repo-name.git my-folder
```

---

## 브랜치와 이력 범위를 조절하여 클론하기

### 특정 브랜치만 클론
```bash
git clone -b develop --single-branch https://github.com/username/repo-name.git
```
- `-b` 또는 `--branch`: 체크아웃할 브랜치를 지정합니다.
- `--single-branch`: 지정한 브랜치의 이력만 가져옵니다. 저장소 크기와 시간을 절약할 수 있습니다.

### 얕은 클론(Shallow Clone)
```bash
git clone --depth 1 https://github.com/username/repo-name.git
```
- 최근 N개의 커밋만 가져옵니다(전체 이력은 제외).
- **장점**: 매우 빠르고 디스크 용량을 적게 사용합니다.
- **단점**: 과거 이력을 조회하거나 `git bisect` 같은 명령을 사용할 수 없습니다.

얕은 클론의 다양한 옵션:
```bash
git clone --depth 50 --branch main --single-branch <url>   # main 브랜치의 최근 50개 커밋만
git clone --shallow-since="2024-01-01" <url>               # 특정 날짜 이후 커밋만
git clone --shallow-exclude="v1.0.0" <url>                 # 특정 태그 이전 이력은 제외
```

> 얕은 클론으로 복제한 저장소는 나중에 전체 이력을 가져올 수 있습니다.
```bash
git fetch --unshallow          # 전체 이력으로 확장
git fetch --depth=500          # 깊이를 500개 커밋으로 늘림
```

---

## 대형 저장소를 위한 최적화 클론: 부분/필터/스파스 클론

### Partial Clone (필터 기반, Git 2.19+)
대용량 저장소에서 파일 내용(블롭)을 필요할 때만 지연 로딩하는 방식입니다.

```bash
git clone --filter=blob:none --no-checkout <url> my-repo
cd my-repo
git sparse-checkout init --cone         # 선택적 체크아웃 준비
git sparse-checkout set src/ include/   # 필요한 경로만 워킹 트리에 가져오기
git checkout main
```
- `--filter=blob:none`: 커밋과 트리 구조 정보는 받되, 파일 내용은 필요 시점에 가져옵니다.
- `--no-checkout`: 초기 체크아웃을 생략합니다. 스파스 체크아웃 설정 후 수동으로 체크아웃합니다.

다른 유용한 필터 옵션:
```bash
git clone --filter=tree:0 <url>         # 트리 정보도 최소화
git clone --filter=blob:limit=1m <url>  # 1MB를 초과하는 파일은 제외하고 필요 시 다운로드
```

### Sparse Checkout (원하는 디렉터리만 워킹 트리로 가져오기)
```bash
git clone <url>
cd repo
git sparse-checkout init --cone
git sparse-checkout set app/ docs/
```
- 저장소의 전체 이력은 로컬에 있지만, 워킹 디렉터리에는 지정한 특정 경로의 파일만 보입니다.
- 모노레포 환경에서 특정 패키지만 개발할 때 매우 유용합니다.

---

## 서브모듈을 포함한 저장소 클론

### 서브모듈이 있는 저장소 클론
```bash
git clone --recurse-submodules <url>
# 또는 두 단계로 나누어 실행

git clone <url>
cd repo
git submodule update --init --recursive
```
- 얕은 클론과 함께 사용하려면:
```bash
git clone --recurse-submodules --depth 1 <url>
git submodule update --init --recursive --depth 1
```

> 주의: 서브모듈의 커밋이 원격에 없거나 접근 권한이 부족할 경우 실패할 수 있습니다. 서브모듈 저장소에 대한 접근 권한(SSH 키, 토큰 등)을 확인해야 합니다.

---

## 인증 방식: HTTPS vs SSH vs PAT(토큰)

| 항목 | HTTPS | SSH |
|---|---|---|
| 주소 형식 | `https://.../repo.git` | `git@host:user/repo.git` |
| 인증 방법 | 사용자명/비밀번호 또는 **Personal Access Token**(권장) | SSH 공개키 |
| 장점 | 방화벽이나 프록시 환경에서 통과가 쉽고 사용이 간단함 | 초기 설정 후 비밀번호 없이 푸시 가능 |
| 단점 | 자격 증명 입력 또는 토큰 관리 필요 | SSH 키 생성 및 호스트에 등록 필요 |

- **HTTPS + PAT**: GitHub 등에서는 보안을 위해 비밀번호 대신 **Personal Access Token** 사용을 권장합니다.
- **SSH 키**: `ssh-keygen -t ed25519 -C "you@example.com"` 명령어로 생성한 공개키를 Git 호스트에 등록합니다.
- **자격 증명 관리자**(Windows: Git Credential Manager, macOS: osxkeychain)를 사용하면 인증 정보를 캐시하여 편리하게 사용할 수 있습니다.

---

## 클론 직후 권장 작업 흐름

```bash
# 저장소로 이동

cd repo-name

# 원격 저장소 정보 확인

git remote -v

# 해당 저장소에만 적용할 사용자 정보 설정 (전역 설정과 다를 경우)

git config user.name  "홍길동"
git config user.email "hong@example.com"

# 모든 브랜치 목록 확인

git branch -a

# 기본 브랜치를 최신 상태로 동기화 (필요한 경우)

git pull --rebase

# 새로운 작업 브랜치 생성

git checkout -b feature/my-work

# 프로젝트 의존성 설치 및 테스트 실행 (프로젝트별)
# 예: Node.js 프로젝트의 경우

npm ci
npm test
```

> 팀에서 제공하는 `CONTRIBUTING.md`, `README.md` 파일이나 `Makefile`, `scripts/` 디렉터리의 안내를 우선적으로 따르는 것이 좋습니다.

---

## 고급 클론 옵션 모음

### `--origin <name>`
기본 원격 저장소 이름을 `origin` 대신 다른 이름으로 지정합니다.
```bash
git clone --origin upstream <url>   # 원격 저장소 이름을 'upstream'으로 설정
```

### `--config key=value`
클론하는 시점에 로컬 Git 설정을 직접 주입합니다.
```bash
git clone --config core.autocrlf=false <url>
```

### `--separate-git-dir <dir>`
`.git` 디렉터리를 작업 디렉터리와 분리하여 다른 경로에 저장합니다.
```bash
git clone --separate-git-dir=/var/git/meta <url> worktree
```

### `--jobs <N>`
데이터 전송을 여러 스트림으로 병렬 처리하여 클론 속도를 향상시킵니다.
```bash
git clone --jobs 8 <url>
```

### `--no-checkout`
저장소 이력은 복제하지만, 파일 체크아웃은 생략합니다. 부분 체크아웃 설정 후 사용할 때 유용합니다.
```bash
git clone --no-checkout <url> myrepo
```

### `--reference`, `--dissociate`
로컬에 이미 존재하는 유사한 저장소의 객체를 재사용하여 대용량 저장소의 클론 속도를 높입니다.
```bash
git clone --reference ../big-repo-cache <url> my-repo
# 이후 독립적으로 운영하려면 리팩터링을 수행합니다.

git repack -a -d
git repack -A
```
또는 재사용 후 자동으로 분리하려면:
```bash
git clone --reference ../big-repo-cache --dissociate <url> my-repo
```

### `--mirror` vs `--bare` (서버 또는 백업용)
- `--bare`: 워킹 트리 없이 순수 Git 데이터만 복제합니다. 주로 중앙 서버에 사용합니다.
- `--mirror`: 모든 브랜치, 태그, 원격 추적 브랜치를 포함한 완전한 미러를 생성합니다. 저장소 전체를 백업하거나 이동할 때 사용합니다.

```bash
git clone --bare <url> my-repo.git
git clone --mirror <url> my-repo.git
```

---

## Git LFS(대용량 파일)와 함께 클론하기

Git LFS를 사용하는 저장소를 클론할 때는 LFS 클라이언트가 설치되어 있어야 합니다.
```bash
git lfs install
git clone <url>
# 또는 필터 클론과 함께 사용

git clone --filter=blob:none <url>   # LFS 포인터만 먼저 가져옴
```
- 실제 대용량 파일은 체크아웃 시점에 자동으로 다운로드됩니다.
- CI/CD 환경에서 LFS 설정이 누락되면 파일이 포인터 텍스트로만 남을 수 있으므로 주의가 필요합니다.

---

## 실전 시나리오별 클론 가이드

### 모노레포에서 특정 패키지만 개발하는 경우
```bash
git clone --filter=blob:none --no-checkout <url> monorepo
cd monorepo
git sparse-checkout init --cone
git sparse-checkout set packages/app-web packages/app-api
git checkout main
```

### 오래된 대형 저장소를 빠르게 살펴보고 싶은 경우
```bash
git clone --depth 1 --branch main --single-branch <url>
```

### 포크(Fork)한 저장소를 원본(Upstream)과 동기화하는 경우
```bash
# 자신의 포크를 클론한 후

git remote add upstream https://github.com/original/repo.git
git fetch upstream
git checkout main
git rebase upstream/main
git push -f origin main
```

### 서브모듈이 많은 저장소를 효율적으로 클론하는 경우
```bash
git clone --recurse-submodules --depth 1 <url>
cd repo
git submodule update --init --recursive --depth 1
```

### 프록시 또는 사내망 환경에서 클론하는 경우
```bash
git config --global http.proxy http://proxy.company.com:8080
git config --global https.proxy https://proxy.company.com:8443
git clone https://git.company.com/team/repo.git
```
- 사내 인증서(CA)가 필요한 경우 `http.sslCAInfo` 설정이 추가로 필요할 수 있습니다.

---

## 트러블슈팅 가이드

| 증상 | 가능한 원인 | 해결 방법 |
|---|---|---|
| `Repository not found` | URL 오타, 저장소가 없음, 접근 권한 없음 | URL 정확성 확인, 조직 가입 또는 접근 권한 요청 |
| `Permission denied (publickey)` | SSH 키가 등록되지 않았거나 SSH 에이전트가 실행되지 않음 | `ssh-add -l`로 키 확인, 공개키를 Git 호스트에 재등록 |
| 클론 속도가 매우 느리거나 중단됨 | 저장소 용량이 큼, 네트워크 문제 | `--depth`, `--filter`, `--single-branch`, `--jobs` 옵션 활용 |
| 서브모듈 초기화 실패 | 서브모듈 저장소에 대한 권한 없음 | 서브모듈 저장소의 접근 토큰 또는 SSH 키 확인 및 설정 |
| LFS 파일이 텍스트 포인터로만 표시됨 | Git LFS가 설치되지 않았거나 훅이 활성화되지 않음 | `git lfs install` 실행 후 재체크아웃 |
| 인증 프롬프트가 반복적으로 나타남 | 자격 증명 관리자가 설정되지 않음 | Git Credential Manager 또는 키체인 도구 설치 및 설정 |

---

## 보안 및 모범 사례

- **2단계 인증(2FA) 활성화**: 계정 보안을 강화합니다.
- **최소 권한 원칙**: 읽기만 필요한 작업에는 읽기 전용 토큰을 사용합니다.
- **Personal Access Token(PAT) 관리**: 필요한 최소 범위(Scope)로 발급하고, 만료일을 설정합니다.
- **회사 정책 준수**: 사내 Git 사용 정책(포크 금지, IP 제한 등)을 확인하고 따릅니다.
- **배포 비밀 정보 분리**: 저장소와 배포에 필요한 비밀 정보(자격 증명, API 키)는 별도로 관리합니다.

---

## 클론 후 브랜치 작업 표준 흐름

```bash
# 기본 브랜치를 최신 상태로 동기화

git checkout main
git pull --rebase

# 새로운 기능 브랜치 생성

git checkout -b feature/login

# 작업 수행 후 커밋...

# 브랜치를 원격에 푸시하고 PR 생성 준비

git push -u origin feature/login
```

팀의 머지 정책(Merge Commit, Squash and Merge, Rebase and Merge)에 따라 Pull Request를 생성하고 병합합니다. 얕은 클론이나 필터 클론을 사용한 경우, PR 테스트 시 필요한 전체 파일과 이력이 정상적으로 참조되는지 확인해야 합니다.

---

## 명령어 요약 치트시트

```bash
# 기본 클론

git clone <url> [디렉터리명]

# 특정 브랜치만 얕게 클론

git clone -b <branch> --single-branch <url>
git clone --depth 1 <url>

# 부분 클론 및 스파스 체크아웃

git clone --filter=blob:none --no-checkout <url> <dir>
cd <dir>
git sparse-checkout init --cone
git sparse-checkout set <paths...>
git checkout <branch>

# 서브모듈 포함 클론

git clone --recurse-submodules <url>
git submodule update --init --recursive

# 원격 저장소 정보 확인 및 변경

git remote -v
git remote set-url origin <new-url>

# 얕은 클론을 전체 이력으로 확장

git fetch --unshallow
git fetch --depth=1000
```

---

## 결론

`git clone`은 Git을 이용한 협업과 개발의 출발점입니다. 단순히 저장소를 복사하는 것에서 나아가, 프로젝트의 규모와 필요에 따라 **얕은 클론**, **필터 클론**, **스파스 체크아웃** 같은 고급 기법을 활용하면 시간과 디스크 공간을 크게 절약할 수 있습니다. 서브모듈, LFS, 다양한 인증 방식에 대한 이해는 원활한 작업을 위한 기반이 됩니다. 특히 대형 프로젝트에 참여할 때는 이 문서의 최적화 옵션들을 적극적으로 검토하고 적용해 보세요.
