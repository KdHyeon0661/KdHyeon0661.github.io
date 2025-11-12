---
layout: post
title: Git - Git Tag
date: 2025-02-26 19:20:23 +0900
category: Git
---
# Git Tag와 버전 릴리스

## 0. 왜 태그인가?

- **의미 있는 버전 이름**으로 커밋 해시를 치환해 **사람이 읽을 수 있는 기준점**을 만든다.  
- **빌드/배포 파이프라인**의 트리거(조건)로 사용하기 좋다.  
- GitHub/GitLab의 **Releases 페이지**와 연결해 **릴리스 노트·아티팩트**를 배포한다.  
- **재현 가능 빌드**: 과거 버전 회귀 시 태그만으로 정확한 커밋을 지목한다.

---

## 1. 태그의 두 가지 유형

| 유형 | 저장 내용 | 일반 용도 | 장점 | 단점 |
|---|---|---|---|---|
| Lightweight tag | 이름 → 커밋 포인터 | 임시 마일스톤, 내부 빌드 | 생성 빠름 | 메타데이터 부족(작성자/메시지/서명 없음) |
| Annotated tag | 작성자, 날짜, 메시지, 서명(선택) + 커밋 | **공식 릴리스**, 공개 배포 | **릴리스에 최적**, 서명 가능 | 생성 시 약간 번거로움 |

> 공식 릴리스에는 **Annotated tag**를 권장한다.

---

## 2. 기본 명령 요약

```bash
# 생성
git tag v1.0.0                         # lightweight
git tag -a v1.0.0 -m "Release 1.0.0"   # annotated

# 특정 커밋에 태그
git tag -a v1.1.0 <commit> -m "Bugfix release"

# 보기
git tag
git show v1.0.0

# 푸시
git push origin v1.0.0
git push origin --tags                 # 모든 태그

# 삭제
git tag -d v1.0.0                      # 로컬
git push origin :refs/tags/v1.0.0      # 원격

# 이동(재지정, 위험)
git tag -fa v1.0.0 -m "Retag to new commit" <new-sha>
git push --force origin v1.0.0
```

**주의**: 이미 공개된 태그를 **덮어쓰는 행동은 강하게 비권장**. 불가피할 때는 팀에 공지·이관 계획을 반드시 포함한다(§11 참고).

---

## 3. 서명 태그(서명된 릴리스)

### 3.1 GPG 서명 태그

```bash
# GPG 키가 등록되어 있어야 함(별도 gpg --full-generate-key, git config user.signingkey ...)
git tag -s v1.2.0 -m "Release 1.2.0 (signed)"
git show v1.2.0       # 서명/작성자/메시지 확인
git push origin v1.2.0
```

- GitHub의 **Verified** 배지를 기대할 수 있다(계정에 공개 키 등록 필요).
- 서명 검증:

```bash
git tag -v v1.2.0
```

### 3.2 SSH 서명 태그(최근 Git)
- Git 2.34+에서 커밋/태그에 **SSH 서명**을 사용할 수 있다.
- 조직 정책상 **GPG 관리 부담**이 있을 때 대안.

---

## 4. SemVer와 태그 네이밍

**Semantic Versioning**(권장 규칙):

- **MAJOR.MINOR.PATCH** (예: `v2.3.1`)
  - `MAJOR`: 하위 호환 깨짐
  - `MINOR`: 기능 추가(하위 호환)
  - `PATCH`: 버그 수정

### 4.1 프리릴리스/빌드 메타데이터

- 프리릴리스 예: `v2.0.0-rc.1`, `v1.3.0-beta.2`  
- 빌드 메타: `v1.3.0+build.20251106`

```bash
# RC 태그
git tag -a v2.0.0-rc.1 -m "Release Candidate 1"
git push origin v2.0.0-rc.1
```

> 레지스트리/패키지 에코시스템마다 프리릴리스 태그 처리 방식(예: npm의 dist-tag)이 다르므로 출고 전략을 문서화하자.

---

## 5. GitHub Releases와 연동

### 5.1 수동 릴리스(웹 UI)

1. 저장소 → **Releases** → **Draft a new release**  
2. Tag version 입력(`v1.2.3`) 또는 UI에서 새 태그 생성  
3. 릴리스 노트 작성(Changelog 붙여넣기)  
4. 바이너리/zip 등 **Assets 첨부**  
5. 공개(또는 Draft 저장)

### 5.2 CLI로 자동 릴리스 (`gh`)

```bash
# GitHub CLI 설치 후
gh release create v1.3.0 \
  --title "v1.3.0" \
  --notes-file CHANGELOG.md \
  dist/myapp-linux-amd64.tar.gz#linux \
  dist/myapp-windows-amd64.zip#windows
```

---

## 6. CI/CD: 태그 푸시→빌드→릴리스 자동화

### 6.1 태그 푸시에 릴리스 생성(최소 예시)

```yaml
# .github/workflows/release.yml
name: Release
on:
  push:
    tags:
      - 'v*.*.*'

jobs:
  release:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
        with:
          fetch-depth: 0

      - name: Build
        run: |
          make build
          ls -al dist

      - name: Create GitHub Release
        uses: softprops/action-gh-release@v1
        with:
          generate_release_notes: true
          files: |
            dist/*.tar.gz
            dist/*.zip
```

### 6.2 Draft → QA → Publish 전략

- 태그 푸시 시 **Draft Release**를 만들고, QA 승인 후 게시:

```yaml
- name: Create draft
  uses: softprops/action-gh-release@v1
  with:
    draft: true
    generate_release_notes: true
```

- 별도 워크플로에서 `gh release edit --draft=false`로 게시.

### 6.3 Conventional Commits 기반 changelog 자동 생성

- 커밋 메시지 규칙(feat/fix/breaking change)을 바탕으로 릴리스 노트 자동화:

```yaml
- name: Generate Changelog
  run: npx conventional-changelog -p angular -i CHANGELOG.md -s
```

> PR 라벨/범주화가 필요하면 Release Drafter 사용을 검토.

---

## 7. 릴리스 브랜치/핫픽스 플로우와 태그

### 7.1 릴리스 브랜치가 있는 팀(Git Flow 유형)

1. `release/1.4` 브랜치에서 QA/문서/버전업 수행  
2. 확정 시 `main`에 머지 → **태그 `v1.4.0`** → 배포  
3. `develop`에도 역머지  
4. 핫픽스: `hotfix/1.4.1` → `main` → **태그 `v1.4.1`** → `develop` 역머지

### 7.2 GitHub Flow(메인 직행)

- 안정적인 `main` 기준 PR → 머지 즉시 태깅/배포  
- 자동화: PR 머지 시 봇이 `vX.Y.Z` 생성/릴리스

---

## 8. 모노레포/다언어 에코시스템 주의

- **Go**: 모듈 경로에 따라 태그 규칙(`v2+`는 `/v2` 디렉터리 필요).  
- **npm**: package.json의 `version`과 태그의 정합성, `npm publish --tag next` 등.  
- **Python**: `setuptools_scm`로 태그에서 버전 파생.  
- **Cargo**: 워크스페이스 개별 크레이트 버전 동기화.

**모노레포**에서는 패키지별 **폴더 기준** 태그/릴리스가 필요할 수 있다(스코프 프리픽스 예: `ui/v1.0.0`, `api/v1.0.0`). 워크플로에서 변경된 경로만 빌드/릴리스하도록 조건 분기(`paths` 필터)한다.

---

## 9. git describe — 빌드 넘버링

- 태그 + 커밋 거리를 사용해 **스냅샷 버전**을 부여:

```bash
git describe --tags --dirty --always
# v1.2.0-15-g7f9b2cd  (v1.2.0 이후 15커밋, g7f9b2cd)
```

CI에서 바이너리/도커 이미지 태깅에 활용 가능(`:v1.2.0-15`).

---

## 10. 태그 보호 정책(Branch Protection과 유사)

- **Protected tags**: 특정 패턴(`v*`)에 대해 **생성/삭제 권한 제한**.  
- GitHub → Settings → Rules → **Tag protection rules**:  
  - 누가 생성/삭제 가능한지  
  - 필수 서명 여부(서명된 태그만 허용)  
- **강제 푸시로 태그 변경**을 위험하게 사용하는 팀은 반드시 보호 규칙을 설정한다.

---

## 11. 태그 교체(이동)의 위험과 처리 절차

- 이미 배포된 `v1.2.3`을 다른 커밋으로 옮기면:
  - 사용자·CI·레지스트리 캐시가 **서로 다른 의미의 같은 버전**을 갖게 된다.
  - **최후 수단**: `v1.2.3`을 삭제 후 **`v1.2.3+build.2`** 또는 **`v1.2.4`**로 재배포.
- 만약 정말 교체해야 한다면:
  1) 태그 교체 이전 버전 **deprecated 공지**  
  2) 팀/고객/CI 구독 채널 알림  
  3) 보호 규칙 예외적으로 해제 후 작업, 즉시 재보호

---

## 12. 일반적인 릴리스 체크리스트

- [ ] 릴리스 브랜치 또는 main의 **빌드 녹색**  
- [ ] 문서/마이그레이션 가이드 정리  
- [ ] **CHANGELOG.md** 업데이트(자동 생성 또는 수동 정제)  
- [ ] 버전 범프(언어별 매니페스트 sync)  
- [ ] **Annotated + (서명)** 태그 생성  
- [ ] GitHub Release 작성(자동/수동) 및 **Assets 첨부**  
- [ ] 알림(슬랙/메일) 및 이슈/PR 닫기 자동화  
- [ ] 태그 보호 규칙 준수 확인

---

## 13. 실전 예제 모음

### 13.1 태그로 버전 업(수동)

```bash
# 1. 버전 범프(예: package.json 수정)
git commit -am "chore(release): bump to v1.4.0"

# 2. 서명 태그
git tag -s v1.4.0 -m "Release v1.4.0"

# 3. 푸시
git push origin main
git push origin v1.4.0
```

### 13.2 PR 머지 시 자동 태깅(봇)

```yaml
on:
  push:
    branches: [main]
jobs:
  tagger:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
        with: { fetch-depth: 0 }
      - name: Determine next version (example)
        run: |
          # 예시: 스크립트로 다음 버전 계산(Conventional Commits 파싱 등)
          echo "v1.5.0" > VERSION
      - name: Create tag
        run: |
          git config user.name "release-bot"
          git config user.email "bot@example.com"
          git tag -a "$(cat VERSION)" -m "Automated release"
          git push origin "$(cat VERSION)"
```

### 13.3 프리릴리스 → 정식 릴리스

```bash
git tag -a v2.0.0-rc.1 -m "RC1 for 2.0.0"
git push origin v2.0.0-rc.1

# 안정화 후
git tag -a v2.0.0 -m "Final 2.0.0"
git push origin v2.0.0
```

---

## 14. 트러블슈팅

### 14.1 “태그가 안 보인다”
- CI가 **얕은 클론(`fetch-depth: 1`)**으로 태그를 받지 못하는 경우:
  ```yaml
  - uses: actions/checkout@v4
    with:
      fetch-depth: 0      # 전체 이력+태그
  ```
- 로컬에서도:
  ```bash
  git fetch --tags
  ```

### 14.2 “버전 정렬이 엉망”
- 쉘 정렬은 사전식. **버전 정렬**은 `sort -V` 사용:
  ```bash
  git tag | sort -V | tail -n 5
  ```

### 14.3 “Detached HEAD에서 태그가 안 보임”
- 현재 체크아웃한 커밋과 무관한 태그는 `git tag --contains` 등으로 필터링.  
- **브랜치 이동** 후 다시 확인.

### 14.4 “태그는 있는데 changelog가 비어 있음”
- 커밋 메시지 규칙이 일관되지 않거나(Conventional Commits 미사용) PR 라벨 체계 없음.  
- 릴리스 노트 자동화 도입(Release Drafter / conventional-changelog).

### 14.5 “서명 검증 실패”
- 공개키 등록 미흡(계정 설정), 서명 키 만료, 서명자 이메일 불일치.  
- `git config user.email`, 키 재발급/재등록 점검.

---

## 15. 보안/컴플라이언스

- **서명 태그 강제**: 릴리스 공급망 신뢰.  
- **릴리스 승인 단계**(환경 보호 규칙, 필수 리뷰/체크): 실수·악의적 변경 억제.  
- **SBOM/서드파티 라이선스 보고**를 릴리스와 연계해 증적 남기기.

---

## 16. 요약 테이블

| 주제 | 핵심 |
|---|---|
| 태그 유형 | 공식 릴리스는 **Annotated(+서명)** 권장 |
| 네이밍 | **SemVer** + 프리릴리스(`-rc.1`)·메타(`+build`) |
| GitHub 연동 | Releases 페이지에 노트·아티팩트 첨부 |
| CI/CD | 태그 푸시 → 빌드 → 자동 릴리스(드래프트/게시) |
| 보호 | **Protected tags**로 생성/삭제·서명 강제 |
| 주의 | **태그 교체 금지**(불가피 시 절차 수반) |
| 대규모 팀 | 릴리스 브랜치/핫픽스 플로우 + changelog 자동화 |

---

## 참고 링크

- Git 공식 문서: **git-tag** — https://git-scm.com/docs/git-tag  
- Semantic Versioning — https://semver.org/  
- GitHub Releases — https://docs.github.com/en/repositories/releasing-projects-on-github/about-releases