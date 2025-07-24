---
layout: post
title: Git - Rebase + Autosquash
date: 2025-02-26 20:20:23 +0900
category: Git
---
# 🧹 Git Rebase + Autosquash로 커밋 정리하는 법

---

## 🧠 왜 커밋 정리가 필요할까?

> 실무에서 작업을 하다 보면, 하나의 기능을 개발하면서 여러 개의 잡다한 커밋이 생기기 마련입니다.

예:
```text
feat: 회원가입 기능 추가
fix: 변수 오타 수정
fix: 유효성 검사 조건 추가
```

이런 커밋은 **PR(Pull Request)** 전에 정리하여:

- **의미 있는 커밋 단위로 병합**
- 깔끔한 Git 로그 유지
- `main` 브랜치 이력에 잡다한 수정 로그를 남기지 않음

---

## ✨ 핵심 기능: `--autosquash`

Git의 `--autosquash`는 `fixup!`, `squash!` 커밋을 자동으로 원래 커밋에 병합하는 기능입니다.  
이를 위해서는 **커밋 메시지 규칙**이 필요합니다.

---

## 🧪 예제 흐름

### 1. 작업 중 커밋

```bash
git commit -m "feat: 게시글 작성 기능"
git commit -m "fixup! feat: 게시글 작성 기능"
git commit -m "squash! feat: 게시글 작성 기능"
```

> 위처럼 커밋 메시지 앞에 `fixup!` 또는 `squash!` + 기존 메시지를 붙이면 Git이 **자동으로 병합 대상을 찾습니다.**

---

### 2. `rebase -i --autosquash` 실행

```bash
git rebase -i --autosquash HEAD~3
```

- `HEAD~3` → 최근 3개의 커밋을 대상으로 실행
- 자동으로 `fixup`/`squash` 커밋이 재배치되고, 병합됩니다

---

## 🔍 실제 인터랙티브 화면 예시

```text
pick  a1b2c3  feat: 게시글 작성 기능
fixup  d4e5f6  fixup! feat: 게시글 작성 기능
squash  e7f8g9  squash! feat: 게시글 작성 기능
```

- `fixup`은 커밋 메시지를 유지하지 않고 병합
- `squash`는 메시지를 **선택적으로 병합**할 수 있게 편집 창이 뜸

---

## ✅ 커밋 메시지 자동화 팁

### `fixup` 커밋 만들기 (기존 커밋 기준 자동 설정)

```bash
git commit --fixup=<커밋해시 or 메시지>
```

예:
```bash
git commit --fixup=feat: 게시글 작성 기능
```

→ 자동으로 `fixup! feat: 게시글 작성 기능` 메시지 생성됨

---

## 🛠 rebase -i --autosquash 전체 흐름 요약

```bash
# 1. 커밋들 생성
git commit -m "feat: A"
git commit -m "fixup! feat: A"

# 2. 커밋 메시지 확인
git log --oneline

# 3. 인터랙티브 리베이스
git rebase -i --autosquash HEAD~2

# 4. 충돌 해결 (필요 시)
# 5. force push (리베이스 후에는 강제 푸시)
git push -f origin feature/my-feature
```

---

## ⚠️ 주의사항

| 항목 | 주의점 |
|------|--------|
| `fixup!` 메시지는 **정확히 일치해야** 자동 매칭됨 |
| `--autosquash`는 `-i`(interactive) 옵션과 함께 써야 작동함 |
| 리베이스 이후에는 **강제 푸시** (`git push -f`) 필요 |
| 협업 브랜치에서는 충돌이나 이력 꼬임 유의 |

---

## 🧹 커밋 정리를 위한 실전 팁

| 작업 | 명령어 |
|------|--------|
| 기존 커밋 수정 | `git commit --amend` |
| 특정 커밋에 수정 덧붙이기 | `git commit --fixup=<hash or msg>` |
| squash + 메시지 병합 | `squash!` 커밋 사용 |
| 깔끔한 커밋 단위 유지 | 작업 단위마다 커밋, 리뷰 전 정리 |
| 자동 병합 적용 | `git rebase -i --autosquash` 사용 |

---

## 🔗 참고 자료

- [Git 공식 문서 - Rebase](https://git-scm.com/docs/git-rebase)
- [Pro Git Book - Interactive Rebase](https://git-scm.com/book/en/v2/Git-Tools-Rewriting-History)
- [Rebase Autosquash 설명](https://git-scm.com/docs/git-rebase#Documentation/git-rebase.txt---autosquash)
