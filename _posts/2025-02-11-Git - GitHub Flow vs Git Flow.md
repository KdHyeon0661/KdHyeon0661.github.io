---
layout: post
title: Git - GitHub Flow vs Git Flow
date: 2025-02-11 20:20:23 +0900
category: Git
---
# GitHub Flow vs Git Flow

## 핵심 비교 요약

| 구분 | Git Flow | GitHub Flow |
|---|---|---|
| 개발 철학 | **릴리스 주기 중심** (사전 계획, 코드 동결 기간 존재) | **지속적 배포 중심** (소규모, 자주 배포, 병합 즉시 릴리스) |
| 구조 복잡도 | 높음 (5종 이상의 브랜치 유형) | 낮음 (주로 `main`과 `feature/*` 브랜치) |
| 주요 브랜치 | `main`, `develop`, `feature/*`, `release/*`, `hotfix/*` | `main`, `feature/*` |
| 배포 방식 | 릴리스 브랜치, 태그, 변경 로그에 의존 | PR 병합 후 자동 배포 (환경별 파이프라인 활용) |
| 적합 대상 | 모바일/데스크톱 애플리케이션, 패키지 소프트웨어, 규제/검증이 엄격한 환경 | 웹 애플리케이션, 마이크로서비스, 스타트업, DevOps 문화가 정착된 환경 |
| 위험 관리 | 브랜치 격리 및 릴리스 동결을 통한 통제 | 소규모 변경, 강력한 자동화, 기능 플래그/가드레일 활용 |

---

# Git Flow — 릴리스 단위의 계획, 격리 및 검증

## 핵심 개념

Vincent Driessen이 제안한 모델로, **기능 개발(feature)** → **통합(develop)** → **릴리스 준비(release)** → **배포(main + 태그)** → **긴급 수정(hotfix)** 의 명확한 단계와 브랜치 경로를 정의합니다.

### 주요 브랜치 역할

| 브랜치 | 생성 기준 | 병합 대상 및 소멸 | 목적 |
|---|---|---|---|
| `main` | 저장소 초기화 시 | 영구 유지 (릴리스 태그와 함께) | 프로덕션 환경의 정확한 상태를 기록 |
| `develop` | `main`에서 분기 | 영구 유지 (릴리스 병합 후 다음 사이클 진행) | 다음 릴리스를 위한 통합 및 안정화 공간 |
| `feature/*` | `develop`에서 분기 | `develop`으로 병합 후 삭제 | 개별 기능 개발을 위한 격리된 공간 |
| `release/*` | `develop`에서 분기 | `main`과 `develop`에 양방향 병합 후 삭제 | 버전 고정, QA, 문서화, 번역 완료 |
| `hotfix/*` | `main`에서 분기 | `main`과 `develop`에 양방향 병합 후 삭제 | 프로덕션 환경의 긴급 버그 수정 |

### 표준 작업 흐름

```
main ← release ← develop ← feature/*
main ← hotfix/*
```

## 명령어 라인 예제

```bash
# 1. develop 브랜치 생성 및 푸시
git checkout -b develop main
git push -u origin develop

# 2. 기능 개발 (feature 브랜치)
git checkout -b feature/login develop
# ... 작업 수행 ...
git commit -m "feat(login): implement form + validation"
git push -u origin feature/login
# GitHub에서 develop을 대상으로 PR 생성 → 리뷰/CI 통과 → 병합

# 3. 기능 브랜치 정리
git branch -d feature/login
git push origin --delete feature/login

# 4. 릴리스 준비 (release 브랜치)
git checkout -b release/1.0.0 develop
# 버전 번호 동결, 번역, 문서화, 마지막 버그 수정 수행
git commit -m "chore(release): freeze 1.0.0"
# main과 develop을 대상으로 각각 PR 생성 및 병합

# 5. 릴리스 태그 생성
git tag -a v1.0.0 -m "Release 1.0.0"
git push origin v1.0.0

# 6. 핫픽스 (hotfix 브랜치)
git checkout -b hotfix/1.0.1 main
# ... 긴급 수정 작업 ...
git commit -m "fix: urgent crash on startup"
# main을 대상으로 PR 생성 및 병합 후 태그 생성, 이후 develop에도 병합
```

## 장단점 분석

- **장점**: **격리성과 가시성**이 뛰어납니다. 릴리스 준비, QA, 번역과 같은 **비기능적 작업에 충분한 시간을 확보**할 수 있으며, 버전 경계가 명확합니다.
- **단점**: 브랜치와 병합 경로가 **복잡**하며, **중복 병합**(하나의 변경사항을 `main`과 `develop`에 두 번 병합)이 발생할 수 있습니다. 긴 개발 주기에서는 브랜치 간 차이(`drift`)가 커지고 대규모 충돌 가능성이 높아집니다.

## 운영 권장사항

- `release/*` 브랜치 생성 시 즉시 **버전 번호를 동결**하고 **변경 로그 초안**을 작성하세요.
- `release/*` 브랜치에서 QA가 진행되는 동안 `develop` 브랜치에 새로운 기능을 병합하는 것을 자제하세요. 필요한 경우 `feature-freeze` 기간을 설정합니다.
- `hotfix/*` 브랜치를 `main`에 병합한 후, **반드시 동일한 수정사항을 `develop` 브랜치(또는 진행 중인 `release/*` 브랜치)에도 병합(백포트)**하세요.

---

# GitHub Flow — 소규모, 자주, 자동화를 통한 안전성 확보

## 핵심 개념

항상 **배포 가능한 상태를 유지하는 `main` 브랜치**를 중심으로 운영됩니다. 모든 작업은 **수명이 짧은 `feature/*` 브랜치**에서 진행되며, PR 리뷰와 CI를 통과한 후 `main` 브랜치에 병합함과 동시에 배포됩니다.

## 표준 작업 절차

```bash
# 1. 기능 브랜치 생성
git checkout -b feature/login main

# 2. 작업 및 테스트
# ... 코드 수정 ...
git commit -m "feat(login): basic form and api integration"

# 3. 원격에 푸시 및 PR 생성
git push -u origin feature/login
# GitHub에서 main을 대상으로 PR 생성 → 리뷰/CI → 승인

# 4. 병합 및 브랜치 정리
# 웹 인터페이스에서 Squash and merge 또는 Rebase and merge 실행
git branch -d feature/login
git push origin --delete feature/login
```

## 장단점 분석

- **장점**: **단순하고 직관적**하여 학습 곡선이 낮습니다. **개발에서 배포까지의 리드타임이 짧아지며**, CI/CD 자동화와 궁합이 좋습니다. **선형적인 히스토리**를 유지하기 쉽습니다.
- **단점**: 릴리스 동결이나 대규모 QA 스프린트가 필요한 조직에는 부적합할 수 있습니다. 대량의 기능이 동시에 개발될 경우 PR 간의 충돌과 관리 부담이 증가할 수 있습니다.

## 운영 권장사항

- **브랜치 보호 규칙**을 적극 활용하세요: `Require pull request reviews`, `Require status checks`, `Require linear history`.
- 병합 방식을 팀 내에서 표준화하세요: **Squash and merge**(커밋 압축) 또는 **Rebase and merge**(선형 히스토리 유지).
- **기능 플래그(Feature Flag)** 를 도입하여 코드 배포와 기능 노출을 분리하고, **자동화된 롤백** 전략(버전 고정, 상태 확인, 카나리 배포)을 마련하세요.

---

# 배포, 버전 관리, 태깅 — 두 워크플로우의 공통 필수 요소

## 버전 관리 (Semantic Versioning)와 태깅

- 태그는 **빌드 아티팩트, 릴리스 노트, 배포물**의 기준점이 됩니다.
- **Semantic Versioning (SemVer)** 규칙(`MAJOR.MINOR.PATCH`)을 따르는 것이 일반적입니다.

간단한 버전 증가 규칙 (개념 표현):
```
next_patch(x.y.z) = x.y.(z+1)
next_minor(x.y.z) = x.(y+1).0
next_major(x.y.z) = (x+1).0.0
```

```bash
# 태그 생성 및 푸시 예시
git tag -a v1.2.3 -m "Release 1.2.3"
git push origin v1.2.3
```

## 변경 로그 (Changelog) 자동화

Conventional Commits 규칙을 따르면 `release-please`, `semantic-release` 같은 도구로 변경 로그를 자동 생성할 수 있습니다.
```
feat: add login form
fix: handle empty email gracefully
chore: update dependencies
```

---

# CI/CD 파이프라인 예시 (GitHub Actions)

## GitHub Flow용 (환경별 자동 배포)

```yaml
# .github/workflows/ci.yml
name: CI
on:
  pull_request:
    branches: [ main ]
  push:
    branches: [ main ]
jobs:
  test:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-node@v4
        with:
          node-version: '20'
      - run: npm ci
      - run: npm test -- --ci
  deploy-prod:
    if: github.ref == 'refs/heads/main' && github.event_name == 'push'
    needs: [test]
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - run: ./scripts/deploy-prod.sh
```

## Git Flow용 (릴리스/핫픽스 태그 기반 배포)

```yaml
# .github/workflows/release.yml
name: Release
on:
  push:
    tags:
      - 'v*.*.*'
jobs:
  build-and-publish:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - run: ./scripts/build.sh
      - run: ./scripts/publish.sh
```

---

# 핫픽스, 백포트, 릴리스 격리 전략

## Git Flow에서의 핫픽스

- `hotfix/*` 브랜치에서 수정 후 `main`에 병합 및 태그를 생성합니다.
- 동일한 변경 사항을 **`develop` 브랜치나 진행 중인 `release/*` 브랜치에 백포트**하여 동기화를 유지합니다.
- 충돌이 발생하면 `cherry-pick`을 사용하거나, **백포트용 PR을 별도로 생성**하여 명시적으로 기록합니다.

## GitHub Flow (지속적 배포)에서의 긴급 대응

- 이론적으로는 `main` 브랜치에 직접 긴급 수정을 병합할 수 있지만, **PR 리뷰와 CI는 반드시 통과**해야 합니다.
- 프로덕션 문제가 발생한 경우, **단기적인 릴리스 브랜치**(예: `release/2025-11-06.1`)를 생성한 후 그 위에서 핫픽스를 개발, 배포한 뒤 즉시 `main`에 병합하는 방식을 고려할 수 있습니다.
- 장기적인 해결책으로는 **기능 플래그를 활용한 롤아웃 제어**를 습관화하는 것이 좋습니다.

---

# 브랜치 보호, 리뷰, 서명, 병합 정책

## 브랜치 보호 규칙 설정 아이디어

- `Require pull request reviews`: 1~2명 이상의 승인을 요구합니다.
- `Require status checks to pass`: 단위/통합 테스트, 정적 분석, 보안 스캔 등이 성공해야 합니다.
- `Require linear history`: **병합 커밋 생성을 차단**하여 선형 히스토리를 유지합니다.
- `Require signed commits`: GPG 또는 SSH 서명을 필수로 합니다.
- `Restrict who can push`: CI 시스템만 직접 푸시할 수 있도록 제한합니다 (태그 및 릴리스 포함).

## 병합 전략 표준화

- **Squash and merge**: GitHub Flow에서 많이 사용됩니다. 히스토리를 깔끔하게 압축하고, 기능 단위로 한 개의 커밋을 생성합니다.
- **Rebase and merge**: 개별 커밋의 세부 내역을 보존하면서 선형 히스토리를 유지합니다. 팀의 리뷰 문화에 맞게 채택할 수 있습니다.
- 일반적인 Merge Commit 방식은 `Require linear history` 규칙과 상충할 수 있습니다.

---

# 하이브리드 모델 — "간소화된 Git Flow"와 "릴리스 브랜치를 추가한 GitHub Flow"

## 간소화된 Git Flow (GitHub Flow + α)

- `develop` 브랜치를 제거하고, **`main` 브랜치를 통합 브랜치로 사용**합니다.
- 대부분의 일상 개발은 GitHub Flow처럼 진행하되, **필요한 경우에만 `release/*` 브랜치를 생성**합니다.
- 장기적인 QA나 번역이 필요한 특별한 릴리스 시기에만 `release/*` 브랜치를 활용하여 안정화 작업을 진행합니다.

## GitHub Flow + 릴리스 브랜치

- 기본적으로는 GitHub Flow를 따릅니다.
- 그러나 **대규모 배포나 주요 이벤트 전**과 같이 특별한 관리가 필요한 시점에는 단기적으로 `release/*` 브랜치를 생성하여 사용합니다.
- 배포가 완료되면 `release/*` 브랜치의 변경 사항을 `main`에 병합하고 브랜치를 삭제합니다.

---

# 모노레포 또는 다중 서비스 환경에서의 고려사항

## 모노레포에서의 워크플로우

- **GitHub Flow를 권장**합니다. **경로 기반 CI**를 설정하여 변경된 특정 패키지나 서비스만 빌드하고 배포하도록 합니다.
```yaml
# paths-filter 액션을 사용한 예시
- uses: dorny/paths-filter@v3
  id: filter
  with:
    filters: |
      serviceA:
        - 'services/a/**'
      serviceB:
        - 'services/b/**'
```

## 독립적인 버전 관리와 릴리스

- Git Flow를 모노레포에 적용하면 서비스별 `release/*` 브랜치 운영이 매우 복잡해질 수 있습니다.
- 해결책으로, 태그 이름에 **서비스 접두사를 포함**할 수 있습니다 (예: `a/v1.2.3`, `b/v3.4.5`).
- 또는, **워크스페이스나 배포 파이프라인을 서비스별로 분리**하는 방법도 있습니다.

---

# 워크플로우 전환 가이드

## Git Flow에서 GitHub Flow로 전환

1.  `develop` 브랜치에 남아 있는 PR과 커밋을 `main` 브랜치로 정리합니다 (필요한 충돌 해결 포함).
2.  `main` 브랜치에 강력한 **보호 규칙을 활성화**하고, CI/CD 자동화를 강화합니다.
3.  팀 내 **병합 방식을 표준화**합니다 (Squash and merge 또는 Rebase and merge).
4.  `release/*` 브랜치 사용을 최소화하고, 필요할 때만 임시로 생성하는 방식으로 전환합니다.
5.  문서화, 번역, QA와 같은 작업은 **기능 플래그, 스테이징 환경, 릴리스 컷오프 날짜** 등을 통해 관리합니다.

## GitHub Flow에서 Git Flow로 전환

- 릴리스 동결, 규제 준수 검증 등이 필요해지면 `develop` 브랜치를 다시 도입하고 `release/*` 브랜치 사용을 시작합니다.
- 배포 승인 프로세스, 릴리스 노트 작성, 태깅 정책을 강화합니다.

---

# 품질, 보안, 규정 준수 (공통 필수 요소)

두 워크플로우 모두 다음과 같은 가드레일을 마련하는 것이 중요합니다.
- **PR 검증 체인**: 단위/통합 테스트, 정적 분석(SAST), 의존성 취약점 스캔(SCA), 라이선스 검사, 시크릿 노출 검사.
- **환경별 승인 게이트**: 스테이징(staging) → 프로덕션(production) 승인을 GitHub Environments 기능으로 관리할 수 있습니다.
```yaml
# .github/workflows/deploy-prod.yml
name: Deploy to Production
on:
  workflow_dispatch:
    inputs:
      version:
        required: true
        description: 'Deployment version tag'
env:
  ENVIRONMENT: production
jobs:
  deploy:
    runs-on: ubuntu-latest
    environment: production # 보호된 환경: 지정된 승인자 필요
    steps:
      - uses: actions/checkout@v4
      - run: ./scripts/deploy --version "${{ github.event.inputs.version }}"
```

---

# 안티패턴과 대응 방안

| 안티패턴 | 증상 | 대응 방안 |
|---|---|---|
| 장기간 유지되는 기능 브랜치 | 대규모 충돌, 컨텍스트 상실 | **작은 규모의 PR**을 지향하고, **트렁크 기반 개발(Trunk-Based Development)** 의 주기를 따르세요. |
| `develop`과 `main` 간의 과도한 차이 | 릴리스나 핫픽스의 역병합 누락 | 병합 시 **체크리스트를 활용**하거나, **자동 백포트 GitHub Action**을 도입하세요. |
| 수동 변경 로그 작성 누락 | 변경 사항 추적 실패 | **Conventional Commits 규칙 채택** 및 **자동 변경 로그 생성 도구** 사용. |
| `main` 브랜치에 직접 푸시 | 코드 검증 절차 우회 | **브랜치 보호 규칙 강화** 및 **CI 통과 필수화**. |
| 무분별한 강제 푸시(`force push`) | 커밋 히스토리 소실 | `--force-with-lease` 옵션 사용을 권장하고, **브랜치 보호 규칙으로 강제 푸시 제한**. |
| 기능 비노출 제어 부재 | 롤백 시 전체 재배포 필요 | **기능 플래그(Feature Flag)** 시스템 도입. |

---

# 실제 적용 시나리오 제안

## 스타트업 웹 서비스
- **권장 워크플로우**: GitHub Flow
- **세부 운영**: 작은 PR(200라인 이하) 기준, **Squash and merge** 병합 방식 사용.
- **배포 전략**: `main` 브랜치 푸시 시 자동 배포. **기능 플래그**를 활용한 점진적 롤아웃.

## 모바일 애플리케이션 (앱 스토어 심사 필요)
- **권장 워크플로우**: Git Flow 또는 하이브리드 모델
- **세부 운영**: `release/*` 브랜치에서 앱 번들 버전 고정, QA, 현지화 작업 수행.
- **배포 전략**: 태그 생성 후 앱 스토어에 제출. 긴급 수정은 `hotfix/*` 브랜치 활용.

## 규제 산업 (의료, 금융 등 - 검증 및 추적 필수)
- **권장 워크플로우**: Git Flow, 또는 GitHub Flow에 릴리스 브랜치/태깅을 강화한 모델
- **세부 운영**: 서명된 커밋, 릴리스 아티팩트 보관, 변경 승인 워크플로우 활용.

---

# 브랜치/커밋 네이밍 규칙 및 PR 템플릿

## 브랜치 네이밍 규칙
```
feature/login-form
fix/payment-timeout
chore/infra-cache
release/1.4.0
hotfix/1.4.1
```

## 커밋 메시지 규칙 (Conventional Commits)
```
feat(login): add form and input validation
fix(auth): handle empty email gracefully
refactor(ui): extract banner component
chore(deps): bump axios to 1.7.0
```

## PR 템플릿 예시 (`.github/pull_request_template.md`)
```markdown
## 변경 목적
-

## 주요 변경 사항
-

## 테스트 수행 여부
- [ ] 단위 테스트 통과
- [ ] 통합 테스트 통과 (결과 링크)

## 기타 확인 사항
- [ ] 관련 문서/변경 로그 업데이트
- [ ] 기능 플래그/롤백 경로 확인 완료
```

---

# 핵심 명령어 및 설정 요약

```bash
### 공통 명령어 ###
# 기능 브랜치 생성
git switch -c feature/x main
git push -u origin feature/x

### Git Flow 특화 명령어 ###
git switch -c develop main          # develop 브랜치 생성
git switch -c release/1.0.0 develop # 릴리스 브랜치 생성
git switch -c hotfix/1.0.1 main     # 핫픽스 브랜치 생성

# 비 Fast-forward 병합 (기록 보존)
git merge --no-ff feature/x

# 태그 생성
git tag -a v1.0.0 -m "Release 1.0.0"

### GitHub Flow 운영 설정 ###
# 저장소 설정(Settings)에서 다음을 구성:
# 1. 병합 버튼 설정: "Squash merging" 또는 "Rebase merging" 허용
# 2. 브랜치 보호 규칙(main): Require linear history, Require status checks, Require pull request reviews 활성화

### 태깅 및 배포 ###
git tag -a v1.2.3 -m "Release 1.2.3"
git push origin v1.2.3
```

---

# 워크플로우 선택 결정 가이드

- 조직에 **릴리스 단위의 검증, 번역, 마케팅 캠페인**이 중요하다면 → **Git Flow** 또는 **하이브리드 모델**을 고려하세요.
- **지속적 배포와 빠른 피드백 루프, 소규모 자주 배포**가 목표라면 → **GitHub Flow**가 적합합니다.
- 두 가지 사이의 중간 지대에 있다면 → GitHub Flow를 기본으로 하되, 필요시 **단기적인 `release/*` 브랜치를 보완**적으로 사용하는 방안을 검토하세요.

의사결정을 돕는 간단한 흐름도:
```
조직에 규제, 심사, 대규모 QA가 필요한가?
    예 → Git Flow 또는 하이브리드 모델 고려
    아니오 → GitHub Flow 채택
```

---

## 결론

Git Flow와 GitHub Flow는 각각 뚜렷한 장점과 적합한 환경을 가진 협업 모델입니다. Git Flow는 **격리된 개발 환경과 체계적인 릴리스 관리**를 제공하는 반면, GitHub Flow는 **단순함과 빠른 배포 속도**를 강점으로 합니다.

어떤 모델을 선택하든, 성공적인 운영을 위해서는 **브랜치 보호 규칙, 강력한 CI/CD 파이프라인, 명확한 병합 정책, 일관된 태깅 및 변경 로그 관리**가 반드시 뒷받침되어야 합니다. 이러한 기반이 없다면 어떤 워크플로우도 제대로 된 효과를 발휘하기 어렵습니다.

가장 중요한 것은 팀의 실제 상황—배포 문화, 위험 허용도, 외부 규제 요구사항—을 진단하여 가장 조화를 이루는 방식을 선택하고, 필요하다면 두 모델의 장점을 결합한 하이브리드 형태로 유연하게 적용하는 것입니다. 워크플로우는 도구일 뿐이며, 궁극적인 목표는 팀의 생산성과 소프트웨어 품질을 지속적으로 높이는 데 있습니다.
