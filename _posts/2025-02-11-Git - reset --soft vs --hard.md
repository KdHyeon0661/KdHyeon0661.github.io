---
layout: post
title: Git - reset --soft vs --hard
date: 2025-02-11 19:20:23 +0900
category: Git
---
# `git reset --soft` vs `git reset --hard`

---

## 🧠 개념: git reset은 무엇을 "되돌리나"?

Git에는 내부 상태가 3단계로 존재합니다:

1. ✅ **HEAD**: 현재 브랜치가 가리키는 커밋
2. ✅ **Staging Area (Index)**: `git add`된 상태
3. ✅ **Working Directory**: 파일이 실제로 존재하는 내 작업 폴더

`git reset`은 이 중 일부 또는 전부를 **과거 커밋 상태로 되돌립니다.**

---

## 🔁 `--soft`, `--mixed`, `--hard` 비교

| 옵션 | HEAD 이동 | Staging 초기화 | 작업 디렉토리 롤백 | 설명 |
|------|-----------|----------------|---------------------|------|
| `--soft` ✅ | O | X | X | 커밋만 취소, 코드/스테이징 상태는 유지 |
| `--mixed` (기본) | O | O | X | 커밋 + 스테이징만 초기화, 파일은 그대로 |
| `--hard` | O | O | O | **모든 것 제거**, 되돌릴 수 없음 |

---

## 💡 `--soft` 예제

```bash
git reset --soft HEAD~1
```

- 가장 최근 커밋 제거
- 변경 내용은 **그대로 스테이징됨**
- 커밋만 되돌리고 다시 수정 or 메시지 바꿔 재커밋 가능

```bash
git commit -m "새 메시지로 커밋"
```

🔎 유용한 상황:
- 커밋 메시지 실수
- 커밋 순서 정리

---

## ⚠️ `--hard` 예제

```bash
git reset --hard HEAD~1
```

- 최근 커밋 제거
- 스테이징도 제거
- **작업 폴더까지 완전히 되돌림**

→ 되돌린 변경사항은 **복구 불가** (reflog 외에는 불가능)

🔎 유용한 상황:
- 완전 초기화하고 싶을 때
- 빌드/테스트 중 엉킨 변경을 제거할 때

❗ 절대 협업 브랜치에서 사용하지 마세요!

---

## 🧪 비교 실험

파일 상태 예시:

```bash
# 파일 생성 및 커밋
echo "v1" > hello.txt
git add hello.txt
git commit -m "v1 커밋"

# 내용 변경
echo "v2" > hello.txt
git add hello.txt
git commit -m "v2 커밋"

# 또 변경 (v3)
echo "v3" > hello.txt
git add hello.txt
git commit -m "v3 커밋"
```

### `--soft`:

```bash
git reset --soft HEAD~1
```

- v2 커밋까지 돌아감
- hello.txt는 v3 내용이 **그대로 스테이징됨**

### `--hard`:

```bash
git reset --hard HEAD~1
```

- v2 커밋까지 돌아감
- hello.txt는 v2 내용으로 **완전히 롤백**

---

## 📊 정리 비교표

| 항목 | `--soft` | `--hard` |
|------|----------|-----------|
| HEAD(브랜치) 되돌림 | ✅ | ✅ |
| Staging 되돌림 | ❌ | ✅ |
| Working 디렉토리 되돌림 | ❌ | ✅ |
| 실수 복구 가능성 | 높음 | **낮음** |
| 사용 목적 | 커밋만 되돌리기 | 완전 초기화 |
| 위험도 | 낮음 | **높음** |
| 협업 사용 가능 | O (로컬에 한정) | ❌ ❌ ❌ |

---

## 🔗 관련 명령

- `git reset --mixed` ← 기본값
- `git reset HEAD` ← staging만 취소
- `git checkout -- 파일명` ← 파일만 복구
- `git reflog` ← reset 전 상태로 돌아가는 유일한 희망

---

## ✅ 실무 팁

| 상황 | 추천 reset 옵션 |
|------|-----------------|
| 커밋 메시지만 다시 작성하고 싶음 | `--soft` |
| 코드 유지하고 커밋만 정리하고 싶음 | `--soft` |
| 스테이징/코드도 초기화하고 싶음 | `--hard` |
| 테스트하다가 코드 망가졌을 때 | `--hard` |
| 공동 작업 중인 브랜치 | reset ❌ / revert or rebase O |

---

## ❗ 실수한 경우 복구 방법

```bash
git reflog
git reset --hard <실수 전 HEAD 해시>
```

→ 가능은 하지만 **운이 좋아야 한다**고 생각하세요.

---

## 📎 참고 링크

- [Git reset 공식 문서](https://git-scm.com/docs/git-reset)
- [Pro Git Book - Reset](https://git-scm.com/book/en/v2/Git-Tools-Reset-Demystified)
