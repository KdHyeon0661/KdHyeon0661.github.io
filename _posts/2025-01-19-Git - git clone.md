---
layout: post
title: Git - git clone
date: 2025-01-19 19:20:23 +0900
category: Git
---
# Git **clone**(클론)

## 1. `git clone`이란?

`git clone`은 **원격 저장소(Remote Repository)**의 이력(커밋·브랜치·태그)과 파일을 **로컬(Local) 저장소**로 복제하는 명령입니다.

```bash
git clone <원격 저장소 주소>
```

- 결과물: `.git`(이력) + 워킹 트리(체크아웃된 파일)
- 보통 `origin` 이라는 이름으로 원격이 등록됩니다(`git remote -v` 확인 가능).
- 대상은 GitHub/GitLab/Bitbucket/사내 Git 서버 등 어떤 Git 호스트든 동일합니다.

---

## 2. 가장 기본적인 클론

### 2.1 HTTPS 방식
```bash
git clone https://github.com/username/repo-name.git
```

### 2.2 SSH 방식
```bash
git clone git@github.com:username/repo-name.git
```

- 현재 디렉터리에 `repo-name/` 폴더가 생성됩니다.
- 다른 디렉터리명으로 받고 싶다면:

```bash
git clone https://github.com/username/repo-name.git my-folder
```

---

## 3. 브랜치·이력 범위를 조절하는 클론

### 3.1 특정 브랜치만
```bash
git clone -b develop --single-branch https://github.com/username/repo-name.git
```
- `-b` 또는 `--branch`: 체크아웃할 브랜치를 지정
- `--single-branch`: 해당 브랜치 이력만 복제(공간·시간 절약)

### 3.2 얕은 클론(Shallow Clone)
```bash
git clone --depth 1 https://github.com/username/repo-name.git
```
- 최근 커밋 N개만 가져옵니다(전체 이력 제외).
- 장점: 빠르고 용량 작음
- 단점: 과거 이력·탐색·bisect 등 제한

얕은 클론의 파생 옵션:
```bash
git clone --depth 50 --branch main --single-branch <url>   # main 최근 50개만
git clone --shallow-since="2024-01-01" <url>               # 특정 날짜 이후만
git clone --shallow-exclude="v1.0.0" <url>                 # 특정 태그 이전 이력 제외
```

> 얕은 클론을 나중에 “깊게” 확장할 수 있습니다:
```bash
git fetch --unshallow          # 전체 이력으로 확장
git fetch --depth=500          # 깊이를 늘림
```

---

## 4. 대형 저장소 최적화: 부분/필터/스파스 클론

### 4.1 Partial Clone (필터 기반, Git 2.19+)
대형 저장소에서 **블롭(파일 내용)** 을 필요할 때만 가져오도록 하는 방식입니다.

```bash
git clone --filter=blob:none --no-checkout <url> my-repo
cd my-repo
git sparse-checkout init --cone         # 선택 영역 체크아웃 준비(선택 사항)
git sparse-checkout set src/ include/   # 필요한 경로만 워킹트리에 풀기
git checkout main
```

- `--filter=blob:none` : 커밋·트리 정보는 받되 파일 내용은 lazy fetch
- `--no-checkout` : 초기 체크아웃 생략(스파스 설정 후 체크아웃)

다른 필터 예시:
```bash
git clone --filter=tree:0 <url>         # 트리도 최소화
git clone --filter=blob:limit=1m <url>  # 1MB 초과 블롭 제외(요청 시 개별 다운로드)
```

### 4.2 Sparse Checkout (원하는 디렉터리만 워킹트리로)
```bash
git clone <url>
cd repo
git sparse-checkout init --cone
git sparse-checkout set app/ docs/
```
- 저장소 전체 이력은 있지만 워킹트리에는 특정 경로만 배치
- 모노레포에서 특정 패키지만 개발할 때 유용

---

## 5. 서브모듈/서브트리와 함께 클론

### 5.1 서브모듈이 있는 저장소
```bash
git clone --recurse-submodules <url>
# 또는
git clone <url>
cd repo
git submodule update --init --recursive
```
- 얕은 클론과 함께:
```bash
git clone --recurse-submodules --depth 1 <url>
git submodule update --init --recursive --depth 1
```

> 자주 발생하는 이슈: 서브모듈 커밋이 원격에 없거나 권한이 없을 때 실패 → 서브모듈 원격 접근 권한 확인 필요(SSH 키, 토큰 등)

### 5.2 서브트리(참고)
서브모듈과 달리 서브트리는 외부 저장소 히스토리를 **현재 저장소에 합쳐 넣는 방식**이라 `clone` 시 별도 초기화가 필요 없습니다. 운영 방식이 달라서 목적에 맞게 선택합니다.

---

## 6. 인증 모델: HTTPS vs SSH vs PAT(토큰)

| 항목 | HTTPS | SSH |
|---|---|---|
| 주소 | `https://.../repo.git` | `git@host:user/repo.git` |
| 인증 | 사용자/비밀번호 또는 **PAT**(권장) | SSH 공개키 |
| 장점 | 방화벽/프록시 친화, 간단 | 초기 설정 후 비대화형 푸시가 편함 |
| 단점 | 자격입력/토큰 관리 필요 | 키 배포/등록 필요, 기업 SSO·보안정책 영향 |

- **HTTPS+PAT**: GitHub는 ID/비밀번호 대신 **Personal Access Token** 사용 권장
- **SSH 키**: `ssh-keygen -t ed25519 -C "you@example.com"` 후 공개키를 Git 호스트에 등록
- **크리덴셜 매니저**(Windows: Git Credential Manager, macOS: osxkeychain, Linux: libsecret 등)로 자격 캐시 권장

---

## 7. 클론 직후 권장 루틴(팀 표준)

```bash
cd repo-name

# 1. 원격 확인
git remote -v

# 2. 유저 정보(이 저장소 범위로만 설정)
git config user.name  "홍길동"
git config user.email "hong@example.com"

# 3. 브랜치 확인
git branch -a

# 4. 최신 동기화(필요 시)
git pull --rebase

# 5. 작업 브랜치 생성
git checkout -b feature/my-work

# 6. 의존성/도구 설치 (언어·프로젝트별)
# 예) Node
npm ci
npm test
```

> 팀 리드가 제공하는 `CONTRIBUTING.md`/`README.md`/`Makefile`/`scripts/` 를 우선 확인하세요.

---

## 8. 고급 옵션 모음

### 8.1 `--origin <name>`
기본 원격 이름을 `origin` 대신 다른 이름으로 지정:
```bash
git clone --origin upstream <url>   # 원격명을 upstream으로
```

### 8.2 `--config key=value`
클론 시점에 로컬 설정 주입:
```bash
git clone --config core.autocrlf=false <url>
```

### 8.3 `--template <dir>`
특정 템플릿 디렉터리를 사용:
```bash
git clone --template ~/.git-template <url>
```

### 8.4 `--separate-git-dir <dir>`
`.git` 디렉터리를 다른 경로에 분리:
```bash
git clone --separate-git-dir=/var/git/meta <url> worktree
```

### 8.5 `--jobs <N>`
여러 팩(패킷)을 병렬로 받아 속도 향상:
```bash
git clone --jobs 8 <url>
```

### 8.6 `--no-checkout`
초기 체크아웃 생략(스파스 설정 후 체크아웃할 때 유용):
```bash
git clone --no-checkout <url> myrepo
```

### 8.7 `--reference`, `--dissociate`
이미 로컬에 유사 저장소가 있을 때 오브젝트 재사용(대용량 최적화):
```bash
git clone --reference ../big-repo-cache <url> my-repo
# 이후 독립 운용 원하면
git repack -a -d
git repack -A
```
또는
```bash
git clone --reference ../big-repo-cache --dissociate <url> my-repo
```

### 8.8 `--mirror` vs `--bare` (관리·이전용)
- `--bare`: 워킹트리 없이 **관리 전용 저장소**(서버 측)
- `--mirror`: 모든 참조(브랜치·태그·원격 참조)까지 포함 **완전 미러**

```bash
git clone --bare <url> my-repo.git
git clone --mirror <url> my-repo.git
```

---

## 9. LFS(대용량 파일)과 클론

Git LFS를 사용하는 저장소는 다음을 참고:
```bash
git lfs install
git clone <url>
# 또는 필터와 함께
git clone --filter=blob:none <url>   # LFS 포인터/프리훅에 따라 자동 패치
```
- LFS 포인터만 받고 실제 바이너리는 체크아웃 시 받습니다.
- CI/빌드 환경에 LFS 설정 누락 시 파일이 “포인터 텍스트”로 남는 문제가 발생 → `git lfs install` 필수.

---

## 10. 실전 시나리오별 정답 레시피

### 10.1 모노레포에서 일부 페키지만 개발
```bash
git clone --filter=blob:none --no-checkout <url> monorepo
cd monorepo
git sparse-checkout init --cone
git sparse-checkout set packages/app-web packages/app-api
git checkout main
```

### 10.2 오래된 대형 저장소를 빠르게 보고 싶다(히스토리 최소)
```bash
git clone --depth 1 --branch main --single-branch <url>
```

### 10.3 포크를 최신 원본에 맞추고 싶다(Fork + Upstream)
```bash
# 내 포크를 클론했다고 가정
git remote add upstream https://github.com/original/repo.git
git fetch upstream
git checkout main
git rebase upstream/main
git push -f origin main
```

### 10.4 서브모듈이 많은 저장소
```bash
git clone --recurse-submodules --depth 1 <url>
cd repo
git submodule update --init --recursive --depth 1
```

### 10.5 프록시/사내망 환경
```bash
git config --global http.proxy http://proxy.company.com:8080
git config --global https.proxy https://proxy.company.com:8443
git clone https://git.company.com/team/repo.git
```
- 사내 CA 인증서가 필요할 수 있어 `http.sslCAInfo`/`http.sslBackend` 조정

---

## 11. 트러블슈팅(원인 → 해결)

| 증상 | 원인 | 해결 |
|---|---|---|
| `Repository not found` | URL 오타, 권한 없음 | URL 확인, 소속/권한/SSO 인증 확인 |
| `Permission denied (publickey)` | SSH 키 미등록/에이전트 미동작 | `ssh-add -l` 확인, 공개키 재등록 |
| 느리거나 중단됨 | 대용량 저장소/네트워크 문제 | `--depth 1`, `--filter=blob:none`, `--single-branch`, `--jobs N` |
| 서브모듈 초기화 실패 | 서브모듈 원격 권한 문제 | 서브모듈 접근 토큰/SSH 키 확인, URL 수정 |
| LFS 파일이 포인터로 남음 | LFS 미설치/훅 미등록 | `git lfs install` 후 다시 체크아웃 |
| 줄바꿈 경고(LF/CRLF) | OS 혼재 | `.gitattributes`로 EOL 정책 통제 |
| 인증 프롬프트 반복 | 크리덴셜 캐시 미사용 | Git Credential Manager/키체인 설정 |

---

## 12. 보안과 정책(실무 팁)

- **2FA 활성화**: 토큰 탈취 리스크 감소
- **최소 권한 원칙**: 읽기만 필요하면 읽기로, 쓰기 권한은 필요시 부여
- **PAT 스코프 최소화/만료 설정**
- **사내 규정 준수**: 포크/프라이빗 클론 정책, IP 허용 목록, SSO/SAML
- **배포 시크릿 분리**: 클론하는 저장소와 배포 자격 증명 분리 관리

---

## 13. 클론 후 표준 브랜치 전략(팀 가이드)

```bash
# 최신 main 동기화 후 작업 브랜치 생성
git checkout main
git pull --rebase
git checkout -b feature/login
# 작업...
git push -u origin feature/login
# PR 생성(웹/CLI)
```

- 머지 정책은 팀 표준(Merge commit / Squash / Rebase merge)에 따릅니다.
- 얕은/필터 클론을 사용했다면, PR 전 테스트에서 이력·파일 요구사항을 충족하는지 확인하세요.

---

## 14. 명령어 치트시트(요약)

```bash
# 기본
git clone <url> [dir]

# 브랜치/얕은
git clone -b <branch> --single-branch <url>
git clone --depth 1 <url>

# 부분/필터/스파스
git clone --filter=blob:none --no-checkout <url> <dir>
cd <dir>
git sparse-checkout init --cone
git sparse-checkout set <paths...>
git checkout <branch>

# 서브모듈
git clone --recurse-submodules <url>
git submodule update --init --recursive

# 원격 확인/변경
git remote -v
git remote set-url origin <new-url>

# 얕은 → 확장
git fetch --unshallow
git fetch --depth=1000
```

---

## 15. 실제 예제 시나리오 로그

### 15.1 대형 모노레포에서 FE만 개발
```bash
git clone --filter=blob:none --no-checkout https://github.com/acme/monorepo.git
cd monorepo
git sparse-checkout init --cone
git sparse-checkout set frontend/ packages/shared-ui/
git checkout main

# 설치·개발
cd frontend
npm ci
npm run dev
```

### 15.2 오픈소스 기여(포크 → 클론 → 업스트림 동기화)
```bash
# GitHub에서 포크 후
git clone git@github.com:myuser/awesome-lib.git
cd awesome-lib
git remote add upstream https://github.com/original/awesome-lib.git
git fetch upstream
git checkout main
git rebase upstream/main
git push -f origin main

git checkout -b fix/spelling
sed -i 's/recieve/receive/g' README.md
git add README.md
git commit -m "docs: fix spelling"
git push -u origin fix/spelling
# 웹에서 PR 생성
```

---

## 16. 결론

- `git clone`은 **로컬 개발의 출발점**이자, 저장소 규모/구조에 맞게 **옵션 선택**으로 속도·용량·안정성을 크게 개선할 수 있습니다.
- **얕은/부분/스파스/필터** 클론은 대형 저장소의 실무 필수 스킬입니다.
- **서브모듈/LFS/인증/프록시/브랜치 정책**을 미리 이해하면 클론 이후의 운영이 훨씬 수월해집니다.

---

## 참고 링크

- Git 공식: git-clone
  https://git-scm.com/docs/git-clone
- GitHub Docs: Cloning a repository
  https://docs.github.com/en/repositories/creating-and-managing-repositories/cloning-a-repository
- Git LFS
  https://git-lfs.com/
