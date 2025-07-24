---
layout: post
title: Git - Actions 기반 CI/CD 자동화
date: 2025-03-16 19:20:23 +0900
category: Git
---
# 🚀 GitHub Actions 기반 CI/CD 자동화 완벽 정리

---

## 📌 CI/CD란?

| 용어 | 설명 |
|------|------|
| CI (Continuous Integration) | 코드 변경 시 자동으로 **빌드/테스트** 실행 |
| CD (Continuous Delivery/Deployment) | CI 이후 **자동 배포**까지 진행 |

> GitHub와 CI/CD 연동 시, PR을 만들거나 merge할 때 자동으로 테스트를 돌리고, 조건이 충족되면 배포까지 가능합니다.

---

## 🔧 GitHub Actions란?

> GitHub가 제공하는 **CI/CD 플랫폼**  
> `.github/workflows/` 디렉토리에 **YAML 파일**을 작성하여 파이프라인 정의

| 장점 | 설명 |
|------|------|
| GitHub에 내장됨 | 별도 툴 없이 바로 사용 가능 |
| 커뮤니티 액션 풍부 | 테스트, 배포, Slack 알림 등 |
| PR 트리거 가능 | PR, push, release 등 다양한 이벤트 대응 |

---

## 📁 기본 구조

디렉토리 구조:
```
📁 .github/
  └── 📁 workflows/
        └── ci.yml  ← 여기에서 테스트 & 배포 자동화 작성
```

---

## 🧪 기본 예제: Node.js 테스트 자동화

```yaml
# .github/workflows/ci.yml

name: CI

on:
  pull_request:
    branches: [ main ]
  push:
    branches: [ main ]

jobs:
  build-and-test:
    runs-on: ubuntu-latest

    steps:
      - name: Checkout code
        uses: actions/checkout@v4

      - name: Use Node.js
        uses: actions/setup-node@v3
        with:
          node-version: 18

      - name: Install dependencies
        run: npm install

      - name: Run tests
        run: npm test
```

---

## ✅ 주요 구성 요소 설명

| 키워드 | 설명 |
|--------|------|
| `on` | 트리거 조건 (push, pull_request, release 등) |
| `jobs` | 수행할 작업 단위 (빌드, 테스트, 배포 등) |
| `steps` | 각 작업 내에서 실행할 명령들 |
| `uses:` | GitHub Actions Marketplace의 외부 액션 사용 |
| `run:` | 셸 명령어 직접 실행 |

---

## 🔐 merge 조건 지정하기 (보호된 브랜치 설정)

1. GitHub 저장소 → **Settings → Branches → Add rule**
2. 브랜치 이름 패턴: `main`
3. 다음 옵션 활성화:

| 항목 | 설명 |
|------|------|
| ✅ Require pull request reviews before merging | 리뷰 승인 필요 |
| ✅ Require status checks to pass | 워크플로우 테스트 통과 필요 |
| ✅ Include administrators | 관리자도 적용 |

4. Status check 이름: `CI / build-and-test` (YAML의 `name:` 기준)

> 테스트에 실패하면 merge 불가

---

## 🧑‍💻 실무에서 사용하는 트리거 예시

| 이벤트 | 사용 예시 |
|--------|-----------|
| `push` | 커밋 발생 시 |
| `pull_request` | PR 생성/수정 시 |
| `workflow_dispatch` | 수동 실행 (버튼 누르기) |
| `release` | 릴리스 태그 생성 시 자동 배포 |
| `schedule` | 매일 새벽 배치 실행 (cron) |

---

## 🚀 자동 배포 예시 (예: Netlify, Firebase, AWS)

### Netlify 배포

```yaml
- name: Deploy to Netlify
  uses: nwtgck/actions-netlify@v2
  with:
    publish-dir: './dist'
    production-branch: 'main'
    netlify-auth-token: ${{ secrets.NETLIFY_AUTH_TOKEN }}
    site-id: ${{ secrets.NETLIFY_SITE_ID }}
```

> 배포용 토큰과 사이트 ID는 GitHub `Secrets`에 등록

---

## 📊 GitHub Actions 결과 보기

- PR 페이지나 커밋 아래에서 Actions 탭 확인 가능
- 워크플로우 실행 결과, 로그 확인 가능
- 실패 시 빨간 ❌, 성공 시 초록 ✅ 표시

---

## 🎯 실전 전략 요약

| 전략 | 설명 |
|------|------|
| PR 트리거 + 자동 테스트 | PR의 신뢰성 확보 |
| 리뷰 & 테스트 조건 병합 | 코드 품질 보장 |
| 환경변수는 Secrets 사용 | 보안 강화 |
| `matrix` 전략 | 여러 Node.js, Python 버전에서 병렬 테스트 |
| 실패 시 Slack/Discord 알림 | 알림 연동으로 빠른 대응 가능 |

---

## 🔗 참고 링크

- [GitHub Actions 공식 문서](https://docs.github.com/en/actions)
- [actions/checkout](https://github.com/actions/checkout)
- [actions/setup-node](https://github.com/actions/setup-node)
- [Protected Branches 설정](https://docs.github.com/en/repositories/configuring-branches-and-merges-in-your-repository/managing-protected-branches)