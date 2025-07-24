---
layout: post
title: Git - git bisect
date: 2025-03-16 21:20:23 +0900
category: Git
---
# 🐞 git bisect: 버그 추적 자동화 도구

---

## ✅ 개요

> `git bisect`는 Git의 내장 도구로, **버그가 발생한 커밋을 자동으로 이진 탐색**하여 찾아줍니다.

### 💡 예시 시나리오:

- 커밋 `A`: 정상 작동 (good)
- 커밋 `Z`: 버그 발생 (bad)
- 중간 수십 개의 커밋 중 어디서 버그가 생겼는지 모름

→ `git bisect`가 자동으로 중간 지점을 탐색해가며 **문제가 생긴 커밋을 pinpoint** 합니다.

---

## 🧪 사용 흐름

### 1. bisect 시작

```bash
git bisect start
```

### 2. bad 커밋 지정 (버그 발생 커밋)

```bash
git bisect bad
```

### 3. good 커밋 지정 (정상 작동하는 마지막 커밋)

```bash
git bisect good <커밋 해시 or 브랜치>
```

> 이러면 Git이 두 커밋 사이를 기준으로 **이진 탐색 시작**합니다.

---

## 🔁 이후 Git이 중간 커밋으로 자동 이동

이제 여러분은 **테스트를 수행한 후**, 다음처럼 응답합니다:

```bash
# 문제가 없으면:
git bisect good

# 버그가 있으면:
git bisect bad
```

→ Git은 다음 커밋으로 이동하며 이 과정을 반복  
→ 보통 **O(log N)** 수준으로 몇 번만에 원인 커밋을 찾아냄

---

## 🧠 전체 예제

```bash
git bisect start
git bisect bad                # HEAD: 현재 버그 있음
git bisect good abc123        # abc123 커밋: 정상 작동

# Git이 중간 커밋으로 이동
# → 수동으로 앱 실행/테스트
git bisect bad                # 버그 존재

# 다음 중간 커밋으로 이동
git bisect good               # 정상 작동

...

# 최종 결과
# "abcd1234 is the first bad commit"
```

→ 이 커밋에서부터 버그가 시작되었음을 확인할 수 있음

---

## 🧪 자동화 테스트 스크립트로 bisect 사용

### bash 예제:

```bash
git bisect start
git bisect bad
git bisect good abc123

git bisect run ./test-script.sh
```

### 예: test-script.sh

```bash
#!/bin/bash
npm install
npm test

# 테스트 실패 시 1, 성공 시 0 반환
```

> 이 방법은 사람이 개입하지 않고 **전체 자동화**가 가능합니다.

---

## 🧹 bisect 종료

```bash
git bisect reset
```

> 원래 작업하던 브랜치/커밋으로 돌아옵니다.

---

## 🧠 요약

| 명령어 | 설명 |
|--------|------|
| `git bisect start` | 이진 탐색 시작 |
| `git bisect bad` | 버그 발생 커밋 |
| `git bisect good <커밋>` | 정상 작동 커밋 지정 |
| `git bisect run <script>` | 자동화 테스트 실행 |
| `git bisect reset` | 탐색 종료 및 복귀 |

---

## 📌 유용한 팁

| 상황 | 팁 |
|------|----|
| 커밋 수 많을 때 | 시간 절약 극대화 (O(log N)) |
| 스크립트 테스트 가능할 때 | `git bisect run` 적극 활용 |
| 충돌 발생 시 | 수동으로 `git bisect skip` 가능 |
| CI/CD와 함께 사용 | 실패 시점을 추적하는 백트래킹 도구로 사용 가능 |

---

## 🔗 참고 링크

- [Git 공식 문서: git-bisect](https://git-scm.com/docs/git-bisect)
- [Pro Git Book - Bisect](https://git-scm.com/book/en/v2/Git-Tools-Debugging-with-Git)