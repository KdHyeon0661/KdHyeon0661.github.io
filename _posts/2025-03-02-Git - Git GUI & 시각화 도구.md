---
layout: post
title: Git - Git GUI & 시각화 도구
date: 2025-02-26 19:20:23 +0900
category: Git
---
# 🧭 Git GUI & 시각화 도구 총정리

---

## 1️⃣ Git 내장 시각화 명령어

### 🔍 `git log`로 브랜치 흐름 시각화

```bash
git log --graph --oneline --all
```

| 옵션 | 설명 |
|------|------|
| `--graph` | 브랜치 구조를 아스키 그래프로 표시 |
| `--oneline` | 커밋을 한 줄로 요약 |
| `--all` | 모든 브랜치의 커밋 출력 |

### 🧪 예시 출력:

```text
* a1b2c3 (HEAD -> main) Fix typo
| * 4d5e6f (feature/login) Add login form
|/
* 9a8b7c Initial commit
```

> 복잡한 브랜치 흐름을 시각적으로 확인 가능  
> **git log**는 필수 스킬이며, 숙련된 개발자도 많이 사용함

---

## 2️⃣ `tig`: 터미널 기반 Git 로그 탐색기

### 🧰 tig이란?

> `tig`는 Git 커밋, 브랜치, diff, 파일 변경 내역 등을 **터미널에서 인터랙티브하게 탐색**할 수 있는 도구입니다.

### 📦 설치 방법

- macOS: `brew install tig`
- Ubuntu: `sudo apt install tig`
- Windows: WSL 또는 Git Bash에서 설치 가능

### ✅ 사용 방법

```bash
tig               # 전체 로그 보기
tig status        # 현재 변경된 파일 보기
tig blame <file>  # 해당 파일의 blame 보기
tig <branch>      # 특정 브랜치 로그 보기
```

### 🧪 장점

- 터미널 환경에서 **직관적인 인터페이스**
- Vim 스타일 키맵 (j/k, q, Enter 등)
- 커밋 내역을 `diff`와 함께 확인 가능

---

## 3️⃣ GUI Git 클라이언트 툴

### 🖼️ 1. SourceTree (by Atlassian)

| 항목 | 설명 |
|------|------|
| 플랫폼 | Windows / macOS |
| 특징 | 무료, 깔끔한 UI, 브랜치 시각화 탁월 |
| 주요 기능 | Rebase, Stash, Cherry-pick, Conflict 해결, Submodule 지원 |
| 단점 | 대형 저장소에서 느릴 수 있음 |

> Git을 시각적으로 익히고 싶은 입문자에게 추천

---

### 🌐 2. GitKraken

| 항목 | 설명 |
|------|------|
| 플랫폼 | Windows / macOS / Linux |
| 특징 | 멋진 UI, 통합 GitHub/GitLab/Bitbucket 연동 |
| 주요 기능 | 시각적 머지, 팀 협업 기능, WSL 지원, rebase 편리 |
| 라이선스 | 개인은 무료, 팀 기능은 유료 |

> 협업 + GUI 기반 브랜치 관리를 원한다면 GitKraken

---

### 🧩 3. VSCode GitLens 확장

#### 💡 GitLens란?

> VSCode에서 Git 정보를 시각화하는 최고의 Git 확장 플러그인  
> 작성자 정보, 변경 기록, blame, diff, 커밋 로그까지 한눈에 확인

### 🔧 설치 방법

- VSCode → Extensions → `GitLens` 설치
- 또는 CLI:
  ```
  code --install-extension eamodio.gitlens
  ```

### 🧪 주요 기능

- 각 코드 라인의 **작성자/시간** 표시
- 커밋 로그 타임라인
- 시각적 blame 보기
- 파일 변경 비교 (inline diff)
- GitHub PR 연동

### 📸 예시 UI

```text
▸ src/components/Header.tsx
  - You (5 days ago) Add login check
  - Jane (2 weeks ago) Update header style
```

---

## 🔁 요약 비교표

| 툴/명령어 | 종류 | 장점 | 단점 |
|-----------|------|------|------|
| `git log --graph` | CLI | 빠르고 기본 제공 | 텍스트 기반 |
| `tig` | CLI UI | 터미널 인터페이스 | CLI 비숙련자에겐 어렵게 느껴질 수 있음 |
| SourceTree | GUI | 초보자 친화적 | 성능 이슈 |
| GitKraken | GUI | 예쁜 UI + 협업 기능 | 유료 기능 많음 |
| GitLens | VSCode 확장 | 코드 수준 blame & 시각화 | VSCode에 종속 |

---

## 🧠 마무리 팁

- **CLI를 먼저 익히고**, 필요할 때 GUI를 보조로 사용하세요.
- `tig`는 CLI 숙련자에게 최고의 도구
- 협업/리뷰가 많다면 **GitLens + GitHub 연동**으로 워크플로우 자동화 가능

---

## 🔗 참고 링크

- [Tig 공식 문서](https://jonas.github.io/tig/)
- [SourceTree 공식 사이트](https://www.sourcetreeapp.com/)
- [GitKraken 공식 사이트](https://www.gitkraken.com/)
- [GitLens Marketplace](https://marketplace.visualstudio.com/items?itemName=eamodio.gitlens)