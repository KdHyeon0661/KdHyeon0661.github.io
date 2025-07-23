---
layout: post
title: Git - Git Submodule
date: 2025-02-19 19:20:23 +0900
category: Git
---
# 🧩 Git Submodule 완전 정리

---

## 🧠 서브모듈이란?

Git 서브모듈(Submodule)은 **하나의 Git 저장소 안에 다른 Git 저장소를 포함**시켜 관리할 수 있는 기능입니다.

> 즉, 메인 저장소의 **하위 디렉토리처럼 동작하지만** 내부적으로는 별개의 독립된 저장소입니다.

---

## 📌 언제 사용할까?

- **공통 라이브러리** 또는 UI 컴포넌트를 여러 프로젝트에서 공유하고 싶을 때
- 팀/조직 내에서 **공유 모듈을 독립적으로 개발 및 버전 관리**하고 싶을 때
- 오픈소스 라이브러리를 **원본 그대로 가져와 특정 커밋에 고정**하고 싶을 때

---

## 🔧 서브모듈 추가 방법

### 1. 서브모듈 추가

```bash
git submodule add https://github.com/other/repo.git path/to/subdir
```

- `repo.git` → 가져올 서브모듈 저장소 주소
- `path/to/subdir` → 현재 프로젝트에서 서브모듈이 위치할 디렉토리

> 예시:

```bash
git submodule add https://github.com/my-org/common-ui.git libs/common-ui
```

---

### 2. 커밋 및 푸시

```bash
git add .gitmodules libs/common-ui
git commit -m "Add common-ui as submodule"
git push
```

`.gitmodules` 파일이 생성되고, **서브모듈의 커밋 해시**도 트래킹됩니다.

---

## 🔄 서브모듈 클론 및 업데이트

서브모듈이 포함된 저장소를 **처음 클론하는 경우**, 일반 `git clone`만 해서는 내부 서브모듈은 비어 있습니다.

### 1. 서브모듈까지 함께 클론하려면?

```bash
git clone --recurse-submodules https://github.com/my-org/my-repo.git
```

### 2. 이미 클론한 경우 서브모듈 초기화/업데이트

```bash
git submodule init
git submodule update
```

또는 한번에:

```bash
git submodule update --init --recursive
```

---

## 🔁 서브모듈 업데이트 방법

서브모듈은 **고정된 커밋 해시**를 참조합니다.  
업데이트하려면 해당 디렉토리로 들어가서 직접 pull 해야 합니다.

```bash
cd libs/common-ui
git pull origin main   # 혹은 원하는 브랜치
cd ../
git add libs/common-ui
git commit -m "Update submodule to latest"
```

---

## 🔍 서브모듈 상태 확인

```bash
git submodule status
```

→ 현재 서브모듈이 가리키는 커밋 해시를 확인할 수 있습니다.

---

## 🗑️ 서브모듈 삭제

1. `.gitmodules`에서 항목 제거
2. `.git/config`에서 관련 정보 제거
3. `git rm --cached path/to/submodule`
4. 실제 디렉토리 삭제
5. 커밋

```bash
git rm --cached libs/common-ui
rm -rf libs/common-ui
git commit -m "Remove submodule"
```

---

## ⚠️ 서브모듈의 단점 및 주의사항

| 항목 | 설명 |
|------|------|
| ✅ 독립 버전 관리 | 외부 저장소 커밋 기준으로 관리되어 안정적 |
| ❌ 경로 혼란 | 서브모듈 내에서 작업 시 상위 저장소와 충돌 가능성 |
| ❌ 자동 업데이트 아님 | 수동으로 `pull`, `update` 해줘야 최신화됨 |
| ❌ 클론 시 초기화 필요 | `--recurse-submodules` 또는 별도 `init/update` 필수 |
| ❌ 커밋 해시 고정 | 브랜치가 아닌 커밋을 참조하므로 최신 변경 자동 반영 X |
| ❌ push 누락 잦음 | 서브모듈 디렉토리 안에서 따로 `git push` 해야 함 |

---

## 🧪 예시 구조

```text
my-project/
├── .gitmodules
├── libs/
│   └── common-ui/      ← 서브모듈 (별도의 Git 저장소)
└── src/
```

`.gitmodules` 내용 예시:

```ini
[submodule "libs/common-ui"]
  path = libs/common-ui
  url = https://github.com/my-org/common-ui.git
```

---

## 🧭 서브모듈 vs 서브트리 비교

| 항목 | 서브모듈 | 서브트리 |
|------|----------|-----------|
| 구조 | 별도 Git 저장소 링크 | 코드가 포함됨 (병합) |
| 장점 | 독립 버전 관리 | 관리 간편, 추가 툴 없음 |
| 단점 | 경로 헷갈림, 수동 업데이트 | 기록 복잡, 추적 어려움 |
| 협업 적합성 | 낮음 (학습 곡선 높음) | 높음 (Git만 알아도 충분) |

> ✨ 서브트리(Subtree)는 추후 다른 주제로 정리 가능

---

## 🔗 참고 링크

- [Git 공식 문서 - Submodule](https://git-scm.com/book/en/v2/Git-Tools-Submodules)
- [Git Submodule vs Subtree](https://www.atlassian.com/git/tutorials/git-submodule)
