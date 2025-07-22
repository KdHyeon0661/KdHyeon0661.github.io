---
layout: post
title: Git - rebase
date: 2025-01-30 19:20:23 +0900
category: Git
---
# Git Rebase 완벽 정리

---

## 🔁 1. Rebase란?

**Git Rebase**는 한 브랜치의 커밋들을 **다른 브랜치의 끝으로 "붙여넣는" 작업**입니다.

> 즉, 두 브랜치를 병합하되, **커밋 히스토리를 병합 커밋 없이 깔끔하게 재구성**할 수 있게 해줍니다.

---

## 🎯 2. Rebase 기본 예시

```bash
# main 브랜치 기준으로 feature/login 브랜치를 재배치
git checkout feature/login
git rebase main
```

- `feature/login` 브랜치가 `main`의 최신 커밋 뒤로 옮겨짐
- `merge`와 달리 병합 커밋이 생성되지 않음

### 🔍 시각적 이해

```text
기존 (rebase 전):
main:     A---B
               \
feature:        C---D

rebase 후:
main:     A---B
                   \
feature:            C'--D'
```

> `C`, `D`는 새 커밋으로 다시 생성된 것처럼 보이며, 기존 커밋과 해시가 다름

---

## ⚙️ 3. rebase vs merge 차이

| 항목 | merge | rebase |
|------|-------|--------|
| 병합 방식 | 병합 커밋(Merge Commit) 생성 | 커밋을 재배열하여 이어붙임 |
| 커밋 히스토리 | 병합 흔적이 남음 | 선형으로 깔끔하게 유지 |
| 충돌 발생 시 | 병합 커밋 내에서 해결 | 각각의 커밋마다 해결 |
| 협업 시 사용 | 안전 (권장) | ⚠️ 주의 필요 (공유 브랜치 금지) |
| 목적 | 이력 보존 | 커밋 정리, 히스토리 간결화 |

---

## ⚠️ 4. 주의: 공유된 브랜치에서 rebase ❌

> 이미 푸시한 브랜치에서 `rebase` 후 강제 푸시는 다른 사람의 작업을 망칠 수 있습니다!

공식 원칙:
> ✅ 개인 브랜치에서만 rebase  
> ❌ 협업 중인 브랜치에서는 merge 사용

---

## ✍️ 5. Rebase 충돌 해결

rebase 중 충돌 발생 시:

```bash
# 충돌 메시지 출력
CONFLICT (content): Merge conflict in index.html
```

### 해결 순서:

1. 파일 수동 수정
2. `git add [수정한 파일]`
3. `git rebase --continue`

### 중단/취소 명령어:

```bash
git rebase --abort      # 중단
git rebase --skip       # 충돌난 커밋 건너뛰기
```

---

## 🧪 6. 인터랙티브 Rebase (커밋 수정, 병합, 순서 변경)

```bash
git rebase -i HEAD~3
```

- 최근 3개의 커밋을 편집할 수 있는 화면이 열림

### 기본 키워드:

| 키워드 | 설명 |
|--------|------|
| `pick` | 커밋 유지 |
| `reword` | 커밋 메시지만 수정 |
| `edit` | 커밋 내용 수정 |
| `squash` | 이전 커밋과 합치기 |
| `drop` | 커밋 제거 |

예시:

```
pick 3a1b2c3 login 기능 추가
reword 4d5e6f7 불필요한 로그 제거
squash 8g9h0i1 리팩토링

# 결과: 1개의 커밋으로 합쳐지고 메시지 수정 가능
```

---

## 🚀 7. rebase 후 푸시

```bash
# 로컬 브랜치 커밋이 변경되므로 강제 푸시 필요
git push -f origin feature/login
```

> ⚠️ `-f`는 다른 사람 작업을 덮어쓸 수 있으므로 반드시 단독 브랜치에서만 사용하세요!

---

## 🧠 8. 언제 rebase를 사용하나요?

- 기능 브랜치의 커밋을 정리하고 main에 깔끔히 병합하고 싶을 때
- 복잡한 히스토리를 단순하게 만들고 싶을 때
- 커밋 메시지를 일괄 수정하거나 정리하고 싶을 때

---

## 📎 명령어 요약

```bash
git checkout feature/login
git rebase main                  # 기본 rebase
git rebase -i HEAD~3             # 인터랙티브 rebase
git rebase --continue            # 충돌 해결 후 재개
git rebase --abort               # 중단
git push -f origin feature/login # rebase 후 강제 푸시
```

---

## 🔗 참고

- Git 공식 Rebase 문서: https://git-scm.com/docs/git-rebase
- GitHub rebase 가이드: https://docs.github.com/en/pull-requests/collaborating-with-pull-requests/working-with-forks/keeping-your-fork-synced-with-the-original-repository#rebase-your-fork