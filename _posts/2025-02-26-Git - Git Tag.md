---
layout: post
title: Git - Git Tag
date: 2025-02-26 19:20:23 +0900
category: Git
---
# 🏷️ Git Tag와 버전 릴리스 완전 정리

---

## 📌 Git Tag란?

> `Tag`는 특정 커밋에 붙이는 **이름표(label)** 로,  
> 보통 **릴리스 버전(v1.0.0)** 이나 **중요한 마일스톤**에 사용합니다.

### ✅ 왜 사용할까?

- 커밋 해시는 외우기 어렵고 읽기 어려움 → `v1.2.0` 같은 **의미 있는 이름**을 붙이기 위해
- 빌드 배포 시 특정 시점의 **커밋을 명확하게 참조**하기 위해
- GitHub 릴리스 페이지와 **자동 연동**되므로, 사용자에게 changelog 제공

---

## 🧪 Tag 종류

| 유형 | 설명 |
|------|------|
| **Lightweight tag** | 이름만 있는 단순한 태그 (커밋 포인터) |
| **Annotated tag** | 작성자, 날짜, 메시지, GPG 서명 포함 (버전 릴리스에 적합) |

---

## 🔧 Git Tag 사용법

### ✅ 1. 태그 생성

#### 💡 Lightweight tag

```bash
git tag v1.0.0
```

#### 💡 Annotated tag (권장)

```bash
git tag -a v1.0.0 -m "Release version 1.0.0"
```

### ✅ 2. 태그 목록 확인

```bash
git tag
```

### ✅ 3. 특정 커밋에 태그 붙이기

```bash
git tag -a v1.1.0 9fceb02 -m "Bugfix release"
```

> 커밋 해시는 `git log`로 확인

---

## 🚀 태그 푸시

### ✅ 1. 개별 태그 푸시

```bash
git push origin v1.0.0
```

### ✅ 2. 모든 태그 푸시

```bash
git push origin --tags
```

---

## 🗑️ 태그 삭제

### ✅ 로컬 삭제

```bash
git tag -d v1.0.0
```

### ✅ 원격 삭제

```bash
git push origin :refs/tags/v1.0.0
```

---

## 🧭 버전 배포 전략 예시 (SemVer)

| 버전 | 의미 |
|------|------|
| `v1.2.0` | 새로운 기능 (Backward-compatible) |
| `v1.2.1` | 버그 수정 (Patch) |
| `v2.0.0` | API 변경 (Breaking change) |

> Semantic Versioning (SemVer) 을 따르는 것이 일반적

---

## 🔗 GitHub와 연동된 릴리스 페이지

### ✅ GitHub에서 릴리스 페이지 만들기

1. GitHub 저장소 → "Releases" 탭 클릭
2. "Draft a new release" 버튼 클릭
3. Tag version 입력 (예: `v1.2.3`)
4. 제목 및 Release Notes 입력
5. Optional: 빌드 파일(zip, exe 등) 첨부 가능

> **태그를 GitHub에서 직접 만들 수도 있고**,  
> 이미 푸시된 태그를 선택해서 릴리스 생성도 가능

---

## 🔄 커맨드 기반 자동 릴리스 예시 (CI/CD에서 사용)

### ✅ 예: v1.3.0 태그를 커밋에 생성하고 푸시

```bash
git tag -a v1.3.0 -m "Release v1.3.0"
git push origin v1.3.0
```

### ✅ GitHub Action에서 릴리스 자동화 예시

```yaml
name: Create GitHub Release

on:
  push:
    tags:
      - 'v*.*.*'

jobs:
  release:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v3
      - name: Create Release
        uses: softprops/action-gh-release@v1
        with:
          generate_release_notes: true
```

---

## 🎯 실무 활용 팁

- 기능이 완성된 시점에 태그 → 빌드 & 배포 트리거
- 태그 이름은 `vX.Y.Z` 형태로 표준화 (SemVer)
- 태그는 브랜치처럼 **mutable 하지 않음** (되도록 덮어쓰지 않기)
- Annotated tag를 기본으로 사용하자 (`-a` 옵션)

---

## 🧰 Git Tag 관련 명령어 정리

| 명령어 | 설명 |
|--------|------|
| `git tag` | 태그 목록 확인 |
| `git tag -a v1.0.0 -m "..."` | 주석 포함 태그 생성 |
| `git show v1.0.0` | 태그 상세 보기 |
| `git push origin v1.0.0` | 태그 푸시 |
| `git push origin --tags` | 모든 태그 푸시 |
| `git tag -d v1.0.0` | 로컬 삭제 |
| `git push origin :refs/tags/v1.0.0` | 원격 삭제 |

---

## 🔗 참고 링크

- [Git 공식 문서: git tag](https://git-scm.com/docs/git-tag)
- [Semantic Versioning 공식 사이트](https://semver.org/)
- [GitHub Releases Docs](https://docs.github.com/en/repositories/releasing-projects-on-github/about-releases)