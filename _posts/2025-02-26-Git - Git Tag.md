---
layout: post
title: Git - Git Tag
date: 2025-02-26 19:20:23 +0900
category: Git
---
# Git 태그와 버전 릴리스 관리 가이드

## Git 태그의 중요성

Git 태그는 프로젝트의 중요한 지점에 의미 있는 이름을 부여하는 방법입니다. 커밋 해시 대신 사람이 읽기 쉬운 버전 번호를 사용하여 다음과 같은 이점을 제공합니다:

- **명확한 버전 참조**: `v1.2.3` 같은 직관적인 이름으로 특정 릴리스를 지목할 수 있습니다.
- **배포 자동화**: CI/CD 파이프라인이 특정 태그를 기준으로 빌드와 배포를 트리거할 수 있습니다.
- **릴리스 관리**: GitHub/GitLab의 릴리스 기능과 연동하여 변경사항과 다운로드 파일을 관리할 수 있습니다.
- **재현 가능성**: 정확한 버전의 코드를 손쉽게 체크아웃할 수 있습니다.

---

## 태그의 두 가지 유형

### Lightweight 태그 (간단한 태그)
특정 커밋을 가리키는 간단한 포인터입니다. 추가 정보 없이 이름만 저장합니다.

```bash
git tag v1.0.0
```

### Annotated 태그 (주석이 있는 태그)
태그 작성자, 날짜, 메시지, 서명 등 메타데이터를 포함합니다. 공식 릴리스에 적합합니다.

```bash
git tag -a v1.0.0 -m "릴리스 1.0.0: 사용자 인증 기능 추가"
```

**실무 팁**: 공식 릴리스에는 Annotated 태그를 사용하세요. 메시지와 작성자 정보가 포함되어 협업과 추적에 도움이 됩니다.

---

## 기본 태그 명령어

### 태그 생성하기
```bash
# 현재 커밋에 태그 생성
git tag v1.0.0
git tag -a v1.0.0 -m "릴리스 메시지"

# 특정 커밋에 태그 생성
git tag -a v1.1.0 abc1234 -m "특정 커밋에 태그"
```

### 태그 조회하기
```bash
# 모든 태그 목록 보기
git tag

# 패턴으로 태그 검색
git tag -l "v1.*"

# 태그 상세 정보 보기
git show v1.0.0
```

### 태그 공유하기 (원격 저장소에 푸시)
```bash
# 특정 태그 푸시
git push origin v1.0.0

# 모든 태그 푸시
git push origin --tags
```

### 태그 삭제하기
```bash
# 로컬 태그 삭제
git tag -d v1.0.0

# 원격 태그 삭제
git push origin :refs/tags/v1.0.0
```

**중요**: 이미 공개된 태그를 수정하거나 삭제하는 것은 피해야 합니다. 다른 사람들이 이미 해당 태그를 사용하고 있을 수 있습니다.

---

## 의미 있는 버전 번호: Semantic Versioning

의미 있는 버전 관리를 위해 Semantic Versioning(SemVer)을 따르는 것이 좋습니다:

### 기본 형식: `MAJOR.MINOR.PATCH`
- **MAJOR**: 하위 호환성을 깨는 변경사항이 있을 때 증가
- **MINOR**: 하위 호환성을 유지하면서 기능을 추가할 때 증가
- **PATCH**: 하위 호환성을 유지하면서 버그를 수정할 때 증가

예: `v2.3.1` (주요 버전 2, 부 버전 3, 패치 버전 1)

### 사전 릴리스 태그
```bash
# 릴리스 후보 (Release Candidate)
git tag -a v2.0.0-rc.1 -m "릴리스 후보 1"

# 베타 버전
git tag -a v1.3.0-beta.2 -m "베타 버전 2"

# 알파 버전
git tag -a v1.2.0-alpha.1 -m "알파 버전 1"
```

---

## 서명된 태그로 보안 강화

서명된 태그는 릴리스의 무결성을 보장합니다. GPG 키로 태그에 서명할 수 있습니다:

### GPG 서명 태그 생성
```bash
# 서명된 태그 생성
git tag -s v1.2.0 -m "서명된 릴리스 1.2.0"

# 서명 검증
git tag -v v1.2.0
```

### GitHub에서 Verified 표시 얻기
1. GPG 키 생성: `gpg --full-generate-key`
2. 공개 키를 GitHub 계정에 등록
3. Git에 서명 키 설정: `git config user.signingkey [키ID]`
4. 태그 생성 시 서명: `git tag -s v1.0.0 -m "릴리스"`

---

## GitHub 릴리스와의 연동

### 수동으로 릴리스 생성하기
1. GitHub 저장소의 **Releases** 섹션으로 이동
2. **Draft a new release** 클릭
3. 태그 버전 입력 (기존 태그 선택 또는 새 태그 생성)
4. 릴리스 제목과 설명 작성
5. 바이너리 파일 첨부 (선택사항)
6. **Publish release** 클릭

### GitHub CLI로 릴리스 생성하기
```bash
gh release create v1.3.0 \
  --title "버전 1.3.0 릴리스" \
  --notes-file CHANGELOG.md \
  dist/app.tar.gz
```

---

## CI/CD와의 통합

태그 푸시를 트리거로 사용하여 자동화된 빌드와 릴리스를 설정할 수 있습니다:

### GitHub Actions 예제
```yaml
name: 릴리스
on:
  push:
    tags:
      - 'v*.*.*'  # v로 시작하는 모든 태그에 반응

jobs:
  릴리스:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      
      - name: 빌드
        run: |
          npm install
          npm run build
          
      - name: GitHub 릴리스 생성
        uses: softprops/action-gh-release@v1
        with:
          generate_release_notes: true
          files: |
            dist/*
            build/*
```

### 사전 릴리스와 정식 릴리스 구분
```yaml
# 사전 릴리스 태그 (rc, beta, alpha) 처리
on:
  push:
    tags:
      - 'v*-rc*'  # 릴리스 후보
      - 'v*-beta*' # 베타 버전
      - 'v*-alpha*' # 알파 버전

jobs:
  프리릴리스:
    runs-on: ubuntu-latest
    steps:
      - name: 프리릴리스 빌드
        run: |
          # 프리릴리스용 특별한 빌드 과정
          npm run build:prerelease
          
      - name: 프리릴리스 생성
        uses: softprops/action-gh-release@v1
        with:
          prerelease: true  # 프리릴리스로 표시
```

---

## 변경 로그 자동 생성

Conventional Commits를 사용하면 변경 로그를 자동으로 생성할 수 있습니다:

### 자동 변경 로그 생성 설정
```yaml
- name: 변경 로그 생성
  run: |
    npm install -g conventional-changelog-cli
    conventional-changelog -p angular -i CHANGELOG.md -s
```

### 커밋 메시지 컨벤션 예시
```
feat: 새로운 기능 추가
fix: 버그 수정
docs: 문서 업데이트
style: 코드 스타일 변경 (기능에 영향 없음)
refactor: 리팩토링
test: 테스트 추가 또는 수정
chore: 빌드 과정 또는 보조 도구 변경
```

---

## 실전 워크플로우 예제

### 예제 1: 수동 릴리스 프로세스
```bash
# 1. 릴리스 준비 브랜치 생성
git checkout -b release/v1.4.0

# 2. 버전 번호 업데이트 (package.json, pom.xml 등)
# 3. 변경 로그 업데이트
# 4. 커밋
git commit -am "chore: 버전 1.4.0으로 업데이트"

# 5. 메인 브랜치에 병합
git checkout main
git merge --no-ff release/v1.4.0

# 6. 태그 생성
git tag -a v1.4.0 -m "릴리스 1.4.0"

# 7. 푸시
git push origin main
git push origin v1.4.0

# 8. GitHub에서 릴리스 노트 작성
```

### 예제 2: 자동화된 릴리스 프로세스
```yaml
# .github/workflows/release.yml
name: 자동 릴리스
on:
  push:
    branches: [main]

jobs:
  릴리스-체크:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
        with:
          fetch-depth: 0
          
      - name: 다음 버전 확인
        id: 버전확인
        run: |
          # 커밋 메시지 분석으로 다음 버전 결정
          NEXT_VERSION="v1.5.0"
          echo "version=$NEXT_VERSION" >> $GITHUB_OUTPUT
          
      - name: 태그 생성 및 푸시
        if: steps.버전확인.outputs.version != ''
        run: |
          git config user.name "GitHub Actions"
          git config user.email "actions@github.com"
          git tag -a ${{ steps.버전확인.outputs.version }} -m "자동 릴리스"
          git push origin ${{ steps.버전확인.outputs.version }}
```

---

## 태그 관련 문제 해결

### 태그가 보이지 않을 때
```bash
# 모든 태그 가져오기
git fetch --tags

# 특정 태그 확인
git show v1.0.0
```

### 버전 순서 정렬하기
```bash
# 버전 번호 순으로 태그 정렬
git tag | sort -V

# 최신 5개 태그 보기
git tag | sort -V | tail -5
```

### 특정 커밋에 어떤 태그가 있는지 확인
```bash
# 커밋에 연결된 태그 찾기
git tag --contains [커밋해시]

# 태그가 가리키는 커밋 찾기
git rev-list -n 1 v1.0.0
```

---

## 보안과 모범 사례

### 1. 태그 보호 정책 설정
GitHub/GitLab에서 태그 보호 규칙을 설정하여 중요한 태그가 실수로 삭제되거나 수정되는 것을 방지하세요.

### 2. 서명된 태그 사용
공식 릴리스에는 항상 서명된 태그를 사용하여 무결성을 보장하세요.

### 3. 태그 이동 금지
이미 공개된 태그를 다른 커밋으로 이동시키지 마세요. 대신 새 버전을 릴리스하세요.

### 4. 일관된 네이밍 규칙
팀 전체가 일관된 태그 네이밍 규칙(SemVer 권장)을 따르도록 하세요.

### 5. 릴리스 노트 작성
모든 릴리스에는 변경사항, 알려진 문제, 업그레이드 지침이 포함된 릴리스 노트를 첨부하세요.

---

## 마무리

Git 태그는 소프트웨어 개발 라이프사이클에서 중요한 역할을 합니다. 올바르게 사용하면 다음과 같은 이점을 얻을 수 있습니다:

### 핵심 장점
1. **명확한 버전 관리**: Semantic Versioning을 통해 버전 변화를 명확하게 전달할 수 있습니다.
2. **자동화 가능성**: CI/CD 파이프라인과 통합하여 반복적인 작업을 자동화할 수 있습니다.
3. **재현 가능성**: 정확한 버전의 코드를 언제든지 재현할 수 있습니다.
4. **협업 강화**: 팀원들이 동일한 기준점에서 작업할 수 있습니다.

### 실무 권장사항
- **릴리스 태그는 Annotated 태그로**: 메타데이터가 포함되어야 합니다.
- **서명된 태그 사용**: 보안과 무결성을 보장합니다.
- **자동화 활용**: 가능한 한 많은 과정을 자동화하여 실수를 줄이세요.
- **문서화**: 릴리스 과정과 태그 사용 규칙을 문서화하세요.
- **팀 협의**: 태그 네이밍 규칙과 릴리스 프로세스를 팀원들과 합의하세요.

Git 태그는 단순한 기술적 도구를 넘어서, 팀의 릴리스 문화와 소프트웨어 품질 관리 방식까지 반영합니다. 체계적인 태그 전략을 수립하고 일관되게 적용하면, 더 안정적이고 관리하기 쉬운 소프트웨어 개발 과정을 구축할 수 있습니다.

태그 관리는 처음에는 다소 번거롭게 느껴질 수 있지만, 일단 습관화되면 개발 워크플로우의 핵심적인 부분이 되어 소중한 시간을 절약하고 실수를 예방하는 데 큰 도움이 될 것입니다.