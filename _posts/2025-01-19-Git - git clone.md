---
layout: post
title: Git - 개요
date: 2025-01-14 19:20:23 +0900
category: Git
---
# Git 클론(clone)에 대해 자세히 알아보기

## 📌 1. git clone이란?

`git clone`은 **원격 저장소(Remote Repository)**의 전체 내용을 복사해서 **로컬(Local) 저장소로 가져오는 명령어**입니다.

```bash
git clone [원격 저장소 주소]
```

- 원격 저장소의 커밋 히스토리, 브랜치, 파일 등이 모두 복제됨
- 주로 GitHub, GitLab, Bitbucket 등에서 사용

---

## 🧪 2. 기본 사용 예시

### ✅ GitHub 저장소를 clone 하기

```bash
git clone https://github.com/username/repo-name.git
```

또는 SSH 방식:

```bash
git clone git@github.com:username/repo-name.git
```

> 기본적으로 현재 디렉토리에 `repo-name`이라는 폴더가 생성됩니다.

---

## ✏️ 3. 디렉토리 이름 지정하기

```bash
git clone https://github.com/username/repo-name.git my-folder
```

- 위 명령어는 저장소를 `my-folder`라는 폴더로 복제합니다.

---

## 🔧 4. 특정 브랜치만 clone

```bash
git clone -b 브랜치이름 --single-branch https://github.com/username/repo-name.git
```

예시:

```bash
git clone -b develop --single-branch https://github.com/username/repo-name.git
```

- `--single-branch`: 해당 브랜치만 복제하여 불필요한 히스토리 절약

---

## ⚙️ 5. --depth 옵션 (얕은 복제)

```bash
git clone --depth 1 https://github.com/username/repo-name.git
```

- **최근 커밋만 복사**하고 전체 히스토리는 생략 → 속도 빠름
- 주로 대용량 저장소를 빠르게 받거나 테스트용으로 사용

---

## 🔄 6. clone 후 기본 리모트 설정 확인

```bash
git remote -v
```

출력 예시:

```
origin  https://github.com/username/repo-name.git (fetch)
origin  https://github.com/username/repo-name.git (push)
```

---

## 🔒 7. HTTPS vs SSH 비교

| 항목 | HTTPS | SSH |
|------|-------|-----|
| 주소 예시 | https://github.com/user/repo.git | git@github.com:user/repo.git |
| 인증 방식 | GitHub ID/PW or PAT 입력 | SSH 키 등록 필요 |
| 보안성 | 비교적 낮음 | 강력한 공개키 기반 인증 |
| 사용 편의성 | 간단하지만 매번 인증 필요 | 초기 설정은 번거롭지만 이후 편리 |
| 협업 환경 | 공용/단기 프로젝트 | 장기 프로젝트나 빈번한 커밋 시 유리 |

> **개인 프로젝트**는 SSH 권장, **간단한 탐색용 clone**은 HTTPS도 무방

---

## 🧠 8. clone 후 해야 할 일들

1. 디렉토리 진입:
   ```bash
   cd repo-name
   ```

2. 브랜치 확인:
   ```bash
   git branch -a
   ```

3. 최신 상태로 동기화:
   ```bash
   git pull
   ```

4. 작업 브랜치 생성:
   ```bash
   git checkout -b feature/my-work
   ```

---

## 🛠️ 9. 자주 묻는 질문

### Q1. "clone 후 수정했는데 GitHub에 올라가지 않아요"
→ `git remote -v` 확인 후, 권한 있는 계정인지 확인  
→ SSH 방식으로 다시 연결하거나 `push` 권한 있는 계정인지 확인

### Q2. "원격 저장소 바꾸고 싶어요"
→ 다음 명령어로 변경 가능:
```bash
git remote set-url origin [새 주소]
```

---

## 📎 참고 명령어 요약

```bash
# 기본 clone
git clone [주소]

# 디렉토리 이름 지정
git clone [주소] [폴더명]

# 특정 브랜치만 복제
git clone -b 브랜치명 --single-branch [주소]

# 얕은 복제 (최신 커밋만)
git clone --depth 1 [주소]

# 원격 저장소 변경
git remote set-url origin [새 주소]
```

---

## 🔗 참고 링크

- Git 공식 clone 문서: https://git-scm.com/docs/git-clone
- GitHub clone 가이드: https://docs.github.com/en/repositories/creating-and-managing-repositories/cloning-a-repository
