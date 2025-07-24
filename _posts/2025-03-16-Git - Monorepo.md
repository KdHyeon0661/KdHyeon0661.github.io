---
layout: post
title: Git - Monorepo
date: 2025-03-16 20:20:23 +0900
category: Git
---
# 🧩 Monorepo 전략 완벽 정리 (with Git 도구)

---

## 1️⃣ Monorepo란?

> Monorepo(모노레포)는 **여러 프로젝트(패키지, 서비스, 모듈)**를 **하나의 Git 저장소(repository)**에서 관리하는 전략입니다.

---

## 📦 Monorepo 구조 예시

```
📁 my-org-repo/
├── apps/
│   ├── web/
│   └── admin/
├── packages/
│   ├── auth/
│   ├── ui/
│   └── utils/
├── tools/
│   └── scripts/
├── package.json
└── nx.json (또는 lerna.json)
```

---

## ✅ Monorepo의 장점과 단점

| 장점 | 설명 |
|------|------|
| 코드 공유 용이 | 공통 패키지, 유틸 코드 재사용 |
| 변경 추적 통일 | 커밋 하나로 전체 변경 이력 추적 |
| 일관된 툴링 | 린트, 테스트, 빌드 설정 통일 |
| 단일 CI 파이프라인 | CI/CD 구성 단순화 |

| 단점 | 설명 |
|------|------|
| Git 저장소 커짐 | clone, pull 속도 느려짐 |
| 빌드 느림 | 전체 빌드 시간이 늘어날 수 있음 |
| 권한 분리 어려움 | 하위 프로젝트 별로 접근 제어 어려움 |

---

## 🛠️ Monorepo를 위한 주요 도구들

---

### 1. `git sparse-checkout`

> Git 2.25+부터 지원하는 **디렉토리 단위 체크아웃 기능**  
> 대형 Monorepo에서 특정 서브 디렉토리만 클론할 수 있도록 도와줍니다.

#### ✅ 예시

```bash
# 1. 저장소 클론 (depth=1 추천)
git clone --filter=blob:none --no-checkout https://github.com/your-org/monorepo.git
cd monorepo

# 2. sparse-checkout 활성화
git sparse-checkout init --cone

# 3. 특정 디렉토리만 포함
git sparse-checkout set apps/web packages/ui
```

#### 🔍 결과

→ 전체 저장소 중 `apps/web`과 `packages/ui`만 다운로드/체크아웃됨  
→ clone 속도, 디스크 사용량 대폭 절감

---

### 2. Lerna (for JS/TS Monorepo)

> 여러 Node.js 패키지를 **단일 저장소**에서 관리할 수 있도록 도와주는 도구

#### ✅ 설치

```bash
npm install --global lerna
lerna init
```

#### 📁 구조

```
lerna.json
packages/
  ├── core/
  ├── ui/
  └── api/
```

#### ✅ 주요 명령어

```bash
lerna bootstrap     # 의존성 설치 및 링크
lerna run build     # 모든 패키지에 빌드 실행
lerna changed       # 변경된 패키지 탐색
lerna publish       # 변경된 패키지만 버전 배포
```

#### 📌 특징

- npm workspace, Yarn Berry 등과 함께 사용 가능
- 각 패키지를 독립적으로 버전 관리하거나 하나로 묶어서 관리 가능

---

### 3. Nx (by Nrwl)

> `Angular`, `React`, `NestJS` 등 현대 웹 프레임워크를 위한 **스마트한 Monorepo 관리 툴**

#### ✅ 설치 및 초기화

```bash
npx create-nx-workspace@latest
```

> 설정 기반으로 React, Next.js, Node, Express, NestJS 등을 선택하여 시작 가능

#### 🧠 Nx의 강력한 기능

| 기능 | 설명 |
|------|------|
| Dependency Graph | 패키지 간 의존성 자동 분석 |
| Affected Only | 변경된 프로젝트만 테스트/빌드 |
| Caching | 빌드 결과 캐시 → 속도 향상 |
| Generators | boilerplate 자동 생성 |
| Executors | 빌드, 린트, 테스트 명령 통합 관리 |

#### ✅ 명령어 예시

```bash
nx run-many --target=build --all
nx affected:test
nx graph
```

---

## 🛠️ 기타 도구

| 도구 | 설명 |
|------|------|
| Bazel | Google이 만든 대형 코드베이스용 빌드 시스템 |
| Turborepo | Vercel이 만든 JS 기반 Monorepo 빌드 최적화 도구 |
| pnpm workspace | 모듈 간 링크, 병렬 빌드, 의존성 최적화 |
| Rush.js | 대형 기업용 Monorepo 관리 도구 (by Microsoft) |

---

## 🧩 Monorepo vs Polyrepo

| 항목 | Monorepo | Polyrepo |
|------|----------|----------|
| 저장소 수 | 1개 | 여러 개 |
| 코드 공유 | 쉬움 (내부 디렉토리) | 어렵고 외부 패키지로 관리 |
| 협업 | 전체 코드 한 눈에 | 독립 관리로 충돌 적음 |
| CI 설정 | 단일 | 프로젝트마다 별도 |
| 실무 적용 | 대규모 플랫폼, 공통 라이브러리 | 마이크로서비스, 분산 팀

---

## 📌 Monorepo 도입 고려 기준

- ✅ 팀 간 코드 공유가 많음
- ✅ 공통 린트/빌드/테스트 전략이 필요함
- ✅ 릴리스 & 배포를 통합하고 싶음
- ❌ 권한 분리가 필요한 민감한 저장소는 지양

---

## 🔗 참고 링크

- [Git 공식 문서: sparse-checkout](https://git-scm.com/docs/git-sparse-checkout)
- [Lerna 공식 사이트](https://lerna.js.org/)
- [Nx 공식 문서](https://nx.dev/)
- [Turborepo by Vercel](https://turbo.build/repo)
- [pnpm workspace](https://pnpm.io/workspaces)