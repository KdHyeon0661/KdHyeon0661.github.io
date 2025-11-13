---
layout: post
title: Git - GitHub Flow vs Git Flow
date: 2025-02-11 20:20:23 +0900
category: Git
---
# GitHub Flow vs Git Flow

## 0. 한눈 요약(리마스터)

| 구분 | Git Flow | GitHub Flow |
|---|---|---|
| 개발 철학 | **릴리스 주기 기반**(사전 계획·동결 기간 존재) | **지속 배포 기반**(작게/자주, 릴리스=머지) |
| 복잡도 | 높음(브랜치 5종 이상) | 낮음(보통 main + feature) |
| 대표 브랜치 | `main`, `develop`, `feature/*`, `release/*`, `hotfix/*` | `main`, `feature/*` |
| 배포 방식 | 릴리스 분기·태그·체인지로그 중심 | PR 머지→자동배포(환경별 파이프라인) |
| 적합 대상 | 모바일/데스크톱 앱, 패키지 SW, 규제/검증강 환경 | 웹/마이크로서비스/스타트업/DevOps 환경 |
| 위험 통제 | 브랜치 격리/릴리스 동결 | 작은 변경·강한 자동화·플래그/가드레일 |

---

# 1. Git Flow — 릴리스 단위로 계획·격리·검증

## 1.1 핵심 개념(요약 복원)
Vincent Driessen 모델. **기능(feature)** → **통합(develop)** → **릴리스 준비(release)** → **배포(main + 태그)** → 필요 시 **핫픽스(hotfix)** 의 명확한 경로.

### 주요 브랜치 역할 정교화

| 브랜치 | 생성 기준 | 소멸/병합 | 목적 |
|---|---|---|---|
| `main` | 배포 기준 | 릴리스 태그와 함께 유지 | 실제 프로덕션 상태의 진실 |
| `develop` | `main`에서 분기 | 릴리스 병합 후 다음 사이클 유지 | 다음 릴리스 통합·안정화 |
| `feature/*` | `develop`에서 분기 | `develop`으로 병합 후 삭제 | 기능 단위 개발 격리 |
| `release/*` | `develop`에서 분기 | `main`/`develop` 양방향 병합 후 삭제 | 버전 고정, QA/문서/번역 마감 |
| `hotfix/*` | `main`에서 분기 | `main`/`develop` 양방향 병합 후 삭제 | 프로덕션 긴급 수정 |

### 표준 흐름(요약 복구)
```
main ← release ← develop ← feature/*
main ← hotfix/*
```

## 1.2 명령 라인 예제(탄탄히)
```bash
# 1. develop 준비
git checkout -b develop main
git push -u origin develop

# 2. 기능 개발
git checkout -b feature/login develop
# ... 작업 ...
git commit -m "feat(login): implement form + validation"
git push -u origin feature/login
# PR → target: develop → 리뷰/CI 통과 → 병합
git branch -d feature/login
git push origin --delete feature/login

# 3. 릴리스 준비
git checkout -b release/1.0.0 develop
# 버전 동결, 번역/문서, 마지막 버그픽스
git commit -m "chore(release): freeze 1.0.0"
# PR → target: main (배포), 그리고 release → develop 역병합(PR)
git tag -a v1.0.0 -m "Release 1.0.0"
git push origin v1.0.0

# 4. 핫픽스
git checkout -b hotfix/1.0.1 main
# ... 수정 ...
git commit -m "fix: urgent crash on startup"
# PR → target: main(배포)
# 병합 후 main→tag v1.0.1, 그리고 hotfix→develop 역병합(PR)
```

## 1.3 장·단점 실무 감각
- 장점: **격리/가시성**. 릴리스 준비·QA·번역 등 **비기능 작업의 기간 확보**. 버전 경계 명확.
- 단점: 브랜치/병합 경로가 복잡, **중복 병합(PR 2개)**, 긴 사이클에서 **드리프트**와 대형 충돌 가능.

## 1.4 운영 규칙(권장)
- `release/*` 생성 즉시 **버전 동결**과 **체인지로그 초안** 생성.
- `release/*` QA 동안 `develop` 에 새 기능 병합 자제(필요시 `feature-freeze` 기간).
- `hotfix/*` 병합 후 **반드시 `develop`(또는 진행 중 `release/*`)로 역병합**.

---

# 2. GitHub Flow — 작게·자주, 자동화로 안전 확보

## 2.1 핵심 개념(요약 복원)
항상 배포 가능한 `main`. 작업은 **짧은 수명**의 `feature/*` 에서 진행 → PR 리뷰/CI 통과 → `main` 병합과 동시에 배포.

## 2.2 표준 절차(명령 + PR)
```bash
git checkout -b feature/login main
# ... 작업/테스트 ...
git commit -m "feat(login): basic form and api integration"
git push -u origin feature/login
# GitHub → PR(target=main) → 리뷰/CI → Approve → Squash or Rebase and merge
# 병합 후 브랜치 삭제
```

## 2.3 장·단점 실무 감각
- 장점: 단순, **리드타임 단축**, 자동화(CI/CD) 친화, **선형 이력 + 배포 자동화**와 궁합.
- 단점: 릴리스 동결·대형 QA 스프린트가 필요한 조직에는 불편. 대규모/동시 기능 다발 시 PR 소음·충돌 가능.

## 2.4 운영 규칙(강력 권장)
- 보호 브랜치: `Require pull request reviews`, `Require status checks`, `Require linear history`.
- 병합 방식: **Squash and merge**(커밋 압축) 또는 **Rebase and merge**(선형 유지), 팀 표준화.
- **Feature flag**/config gate로 배포-노출 분리. **롤백 자동화**(버전 핀/헬스 체크/카나리).

---

# 3. 배포·버전·태깅 — 두 모델 공통의 필수 레이어

## 3.1 버전(semver)과 태깅
- 태그는 **빌드·아티팩트·릴리스 노트**의 기준점.
- 규칙 예: **semver** `MAJOR.MINOR.PATCH`.

간단한 버전 증가 규칙(개념 표현):
$$
\text{next\_patch}(x.y.z) = x.y.(z+1)
$$
$$
\text{next\_minor}(x.y.z) = x.(y+1).0
$$
$$
\text{next\_major}(x.y.z) = (x+1).0.0
$$

```bash
# 태그 생성/푸시
git tag -a v1.2.3 -m "Release 1.2.3"
git push origin v1.2.3
```

## 3.2 체인지로그(자동화 추천)
```bash
# Conventional Commits → 자동 생성(예: release-please, semantic-release)
feat: add login form
fix: handle empty email
chore: deps bump
```

---

# 4. CI/CD 파이프라인 예제(GitHub Actions)

## 4.1 GitHub Flow용(환경별 배포)
```yaml
# .github/workflows/ci.yml
name: ci
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
        with: { node-version: '20' }
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

## 4.2 Git Flow용(릴리스/핫픽스 태그 배포)
```yaml
# .github/workflows/release.yml
name: release
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

# 5. 핫픽스·백포트·릴리스 격리의 정석

## 5.1 Git Flow
- `hotfix/*` → `main` 병합/태그 → 동일 변경을 **`develop` 혹은 진행 중 `release/*`에 백포트**.
- 충돌 시 `cherry-pick` 또는 **백포트용 PR**로 명시적 기록.

## 5.2 GitHub Flow(지속 배포)
- `main`에서 긴급 수정 직행이 가능하지만, **PR 리뷰/CI 필수**.
- 프로덕션 문제라면 **릴리스 브랜치 단기 생성**(예: `release/2025-11-06.1`) 후 그 위에 핫픽스→배포→곧바로 `main` 병합.
- 장기적으로는 **feature flag** 로 노출 제어를 습관화.

---

# 6. 브랜치 보호·리뷰·서명·병합 정책

## 6.1 보호 브랜치 설정 아이디어
- Require pull request reviews: 1~2명 이상 승인
- Require status checks to pass: 유닛/통합/정적분석/보안스캔
- Require linear history: **merge commit 금지**(선형 유지)
- Require signed commits: GPG/SSH-sig
- Restrict who can push: CI만 푸시 허용(태그/릴리스도)

## 6.2 병합 전략 표준화
- **Squash and merge**: GitHub Flow에서 인기. 히스토리 깔끔, 커밋 1개.
- **Rebase and merge**: 커밋 세부 보존 + 선형. 팀의 리뷰 스타일에 따라 채택.
- 일반 Merge(commit) 사용은 `Require linear history` 와 상충.

---

# 7. 하이브리드 모델 — “간소화된 Git Flow”와 “릴리스 브랜치만 추가한 GitHub Flow”

## 7.1 간소화된 Git Flow
- `develop` 제거 → **`main`이 통합 브랜치**.
- 필요 시에만 `release/*` 생성, 대부분은 GitHub Flow처럼 운영.
- 긴 QA가 필요한 분기에서만 `release/*` 를 개설해 안정화.

## 7.2 GitHub Flow + 릴리스 브랜치
- 상시 GitHub Flow. 다만 **규모 큰 배포/이벤트** 앞에서는 `release/*` 를 잠깐 사용.
- 배포 후 즉시 `release/*` → `main` 병합, 브랜치 삭제.

---

# 8. 모노레포/다중 서비스에 대한 고려

## 8.1 모노레포에서의 플로우
- GitHub Flow 권장. **경로 기반 CI** 로 변경된 패키지/서비스만 빌드·배포.
```yaml
# paths-filter 예시
- uses: dorny/paths-filter@v3
  id: filter
  with:
    filters: |
      serviceA:
        - 'services/a/**'
      serviceB:
        - 'services/b/**'
```

## 8.2 독립 버전/릴리스
- Git Flow에서는 서비스별 `release/*` 운용이 과도해질 수 있음.
- 태그 네이밍에 **서비스 접두사** 사용: `a/v1.2.3`, `b/v3.4.5`.
- 또는 **워크스페이스/배포 파이프라인 분리**.

---

# 9. 전환 가이드 — Git Flow → GitHub Flow(또는 역방향)

## 9.1 Git Flow → GitHub Flow
1) `develop` 잔여 PR·커밋을 `main` 으로 정리(충돌 해결).
2) 보호 브랜치 활성화 + CI 강제 + 배포 자동화.
3) 병합 방식 `Squash` 또는 `Rebase` 표준화.
4) **릴리스 브랜치 최소화**(필요할 때만).
5) 문서/번역/QA는 **플래그·스테이징 환경·릴리스 컷**으로 제어.

## 9.2 GitHub Flow → Git Flow
- 릴리스 동결·규제 검증 필요 시: `develop` 복원, `release/*` 도입.
- 배포 승인·릴리스 노트·태깅을 강화.

---

# 10. 품질·보안·규정 준수(두 모델 공통의 필수 가드레일)

- PR 체크: 유닛/통합/정적분석(SAST)/의존성 스캔(SCA)/라이선스/시크릿 스캔.
- 환경별 승인 게이트: staging → prod 승인을 GitHub Environments로 관리.
```yaml
# .github/workflows/deploy-prod.yml
name: deploy-prod
on:
  workflow_dispatch:
    inputs:
      version: { required: true }
env:
  ENVIRONMENT: production
jobs:
  deploy:
    runs-on: ubuntu-latest
    environment: production   # 보호된 환경: 승인자 필요
    steps:
      - uses: actions/checkout@v4
      - run: ./scripts/deploy --version "${{ github.event.inputs.version }}"
```

---

# 11. 안티패턴/사고 유형과 대응

| 안티패턴 | 증상 | 대응 |
|---|---|---|
| 장수 feature 브랜치 | 대형 충돌/컨텍스트 상실 | 작은 PR, **trunk-like** 주기 |
| develop 과도 드리프트 | 릴리스/핫픽스 역병합 누락 | 병합 체크리스트/자동 백포트 액션 |
| 릴리스 노트 수작업 누락 | 변경 추적 실패 | Conventional Commits + 자동 체인지로그 |
| 직접 push to main | 검증 우회 | 보호 브랜치 + CI 요구 |
| 무분별한 force-push | 이력 소실 | `--force-with-lease` 강제, 보호 규칙 |
| 기능 비노출 제어 부재 | 롤백=재배포 필요 | **feature flag** 도입 |

---

# 12. 실제 시나리오 제안

## 12.1 스타트업 웹 서비스
- GitHub Flow.
- 작은 PR(200 라인 내외) 기준, **Squash merge**.
- 배포: main push 시 자동. feature flag 로 점진 노출.

## 12.2 모바일 앱(심사/런칭 캠페인)
- Git Flow 또는 하이브리드.
- `release/*` 에서 번들버전 고정/QA/로컬라이제이션.
- 태그→스토어 제출, 긴급한 경우 `hotfix/*`.

## 12.3 규제 산업(검증·추적)
- Git Flow 또는 GitHub Flow + 릴리스 브랜치/태깅 강화.
- 서명 커밋, 릴리스 아티팩트 보존, 변경 승인 워크플로우.

---

# 13. 브랜치/커밋 네이밍·PR 템플릿

## 13.1 브랜치 네이밍
```
feature/login-form
fix/payment-timeout
chore/infra-cache
release/1.4.0
hotfix/1.4.1
```

## 13.2 Conventional Commits
```
feat(login): add form and input validation
fix(auth): handle empty email gracefully
refactor(ui): extract banner component
chore(deps): bump axios to 1.7.0
```

## 13.3 PR 템플릿(.github/pull_request_template.md)
```markdown
## 목적
-

## 변경 사항
-

## 테스트
- [ ] 단위 테스트
- [ ] 통합 테스트(링크)

## 체크리스트
- [ ] 문서/체인지로그 갱신
- [ ] 플래그/롤백 경로 확인
```

---

# 14. 명령/설정 치트시트

```bash
# 공통
git switch -c feature/x main
git push -u origin feature/x

# Git Flow
git switch -c develop main
git switch -c release/1.0.0 develop
git switch -c hotfix/1.0.1 main

# 병합(권장: PR)
git merge --no-ff feature/x
git tag -a v1.0.0 -m "Release 1.0.0"

# GitHub Flow
# PR 머지 방식을 레포 설정에서 Squash/Rebase로 제한
# 보호 브랜치: Require linear history, status checks, reviews

# 태깅/배포
git tag -a v1.2.3 -m "Release 1.2.3"
git push origin v1.2.3
```

---

# 15. 결정 가이드 — 무엇을 선택할까?

- **릴리스 단위 검증/번역/캠페인**이 크다 → **Git Flow** 혹은 **하이브리드**
- **지속 배포·작게/자주**가 목표 → **GitHub Flow**
- 둘 사이의 회색지대 → GitHub Flow에 **단기 `release/*`** 를 보완

의사결정 트리(텍스트):
```
규제/심사/대형 QA 필요? ── 예 ──> Git Flow 또는 하이브리드
                         └─ 아니오 ──> GitHub Flow
```

---

# 16. 마무리

- Git Flow는 **격리와 릴리스 통제**, GitHub Flow는 **단순성과 속도**를 제공한다.
- 어느 쪽이든 **보호 브랜치/CI/병합 정책/태깅/체인지로그**가 품질을 좌우한다.
- 조직의 **배포 문화·리스크 허용도·규제**에 맞춰 선택하고, 필요 시 **하이브리드**로 균형을 맞추라.

---

## 참고
- Git Flow 원본 글(nvie): https://nvie.com/posts/a-successful-git-branching-model/
- GitHub Flow: https://docs.github.com/en/get-started/quickstart/github-flow
- Atlassian 비교: https://www.atlassian.com/git/tutorials/comparing-workflows
