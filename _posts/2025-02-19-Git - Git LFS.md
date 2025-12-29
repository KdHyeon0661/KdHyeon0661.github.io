---
layout: post
title: Git - Git LFS
date: 2025-02-19 21:20:23 +0900
category: Git
---
# Git LFS 완전 가이드: 대용량 파일 관리의 필수 도구

## Git LFS란 무엇인가?

Git LFS(Git Large File Storage)는 대용량 파일과 이진 파일을 효율적으로 관리하기 위한 Git 확장 도구입니다. 이미지, 동영상, 3D 모델, AI 가중치 파일, 압축 아카이브 등 diff(차이 비교)가 의미없고 저장소 크기를 급격히 증가시키는 파일들을 전문적으로 처리합니다.

### 왜 Git LFS가 필요한가?

일반 Git은 모든 파일 버전을 전체 히스토리에 저장하기 때문에 대용량 파일이 포함된 프로젝트에서는 몇 가지 심각한 문제가 발생합니다:

1. **저장소 폭발적 증가**: 각 커밋마다 대용량 파일의 새 복사본이 추가됨
2. **느린 클론/풀 작업**: 수백 MB 이상의 파일 히스토리를 다운로드하는 데 시간이 오래 걸림
3. **비효율적인 델타 압축**: 이진 파일은 텍스트 파일과 달리 효율적인 차이 저장이 어려움

Git LFS는 이러한 문제를 해결하기 위해 대용량 파일을 별도의 저장소에 보관하고, Git 저장소에는 작은 포인터 파일만 저장하는 방식을 사용합니다.

---

## 설치 및 설정

### 시스템별 설치 방법

**macOS:**
```bash
brew install git-lfs
```

**Ubuntu/Debian:**
```bash
sudo apt update
sudo apt install git-lfs
```

**Windows:**
- [git-lfs.com](https://git-lfs.com)에서 설치 프로그램 다운로드 및 실행

### 저장소 초기화
```bash
# Git LFS를 현재 저장소에 활성화
git lfs install

# 또는 사용자 전역 설정 (모든 저장소에 적용)
git lfs install --global
```

---

## 기본 작업 흐름

### 1. 파일 추적 패턴 설정
```bash
# 특정 파일 확장자 추적
git lfs track "*.psd"
git lfs track "*.mp4"
git lfs track "*.zip"

# 특정 디렉토리 내 파일 추적
git lfs track "models/*.h5"
git lfs track "assets/**/*.png"

# 현재 추적 중인 패턴 확인
git lfs track
```

### 2. .gitattributes 파일 커밋
```bash
# 추적 패턴을 .gitattributes 파일에 저장
git add .gitattributes
git commit -m "Add LFS tracking patterns"
```

`.gitattributes` 파일 예시:
```
*.psd filter=lfs diff=lfs merge=lfs -text
*.mp4 filter=lfs diff=lfs merge=lfs -text
models/*.h5 filter=lfs diff=lfs merge=lfs -text
```

### 3. 대용량 파일 작업
```bash
# 일반적인 Git 작업과 동일하게 진행
git add video.mp4 model.pt dataset.zip
git commit -m "Add large files via LFS"
git push origin main
```

---

## 작동 원리 이해하기

### 포인터 파일 시스템
Git LFS는 실제 대용량 파일을 Git 저장소 외부에 저장하고, Git에는 다음과 같은 포인터 파일을 저장합니다:

```
version https://git-lfs.github.com/spec/v1
oid sha256:4d7a214614ab2935c943f9e0ff69d22eadbb8f32b1258daaa5e2ca24d17e2393
size 123456789
```

### 필터 메커니즘
1. **Clean 필터**: `git add` 실행 시 대용량 파일을 LFS 서버로 업로드하고 포인터 파일로 변환
2. **Smudge 필터**: `git checkout` 실행 시 포인터 파일을 확인하고 실제 파일을 LFS 서버에서 다운로드

### 선택적 다운로드
```bash
# 스머지 필터 비활성화 (포인터 파일만 다운로드)
GIT_LFS_SKIP_SMUDGE=1 git clone https://github.com/user/repo.git

# 필요한 파일만 실제 다운로드
cd repo
git lfs fetch --include="models/*.pt"
git lfs checkout models/*.pt
```

---

## 기존 프로젝트 마이그레이션

### 기존 대용량 파일을 LFS로 전환
```bash
# 백업 브랜치 생성 (안전을 위해)
git branch backup/pre-lfs-migration

# 특정 파일 유형을 LFS로 마이그레이션
git lfs migrate import --include="*.mp4,*.mov,*.zip"

# 특정 경로의 파일 마이그레이션
git lfs migrate import --include="assets/**"

# 모든 커밋 히스토리에서 마이그레이션
git lfs migrate import --everything
```

**주의사항**: 마이그레이션은 커밋 히스토리를 재작성하므로 공유 브랜치에서 작업할 경우 팀원들과 사전 조율이 필요합니다.

### 강제 푸시 (필요 시)
```bash
git push --force-with-lease
```

---

## 고급 사용법

### CI/CD 환경에서의 LFS

**GitHub Actions 예제:**
```yaml
name: CI with LFS
on: [push, pull_request]
jobs:
  build:
    runs-on: ubuntu-latest
    steps:
    - uses: actions/checkout@v3
      with:
        lfs: true  # LFS 파일 자동 다운로드
        
    - name: LFS 파일 상태 확인
      run: git lfs ls-files
      
    - name: 빌드 및 테스트
      run: |
        npm ci
        npm run build
        npm test
```

### 선택적 파일 다운로드 (CI 최적화)
```yaml
steps:
- uses: actions/checkout@v3
  with:
    lfs: false  # LFS 파일 다운로드 비활성화
    
- name: 필요한 LFS 파일만 다운로드
  run: |
    git lfs install
    git lfs fetch --include="models/essential.pt"
    git lfs checkout models/essential.pt
```

### 전역 include/exclude 규칙 설정
```bash
# 특정 패턴만 다운로드하도록 설정
git config lfs.fetchinclude "models/*.pt,assets/images/*.png"

# 특정 패턴 제외 설정
git config lfs.fetchexclude "*.mp4,*.mov,videos/**"

# 설정 확인
git config --get lfs.fetchinclude
```

---

## 호스팅 서비스별 특징

### GitHub
- **무료 제한**: 1GB 저장소 + 1GB 대역폭/월
- **초과 시**: 추가 구매 필요
- **팀 플랜**: 조직 단위 할당량 관리 가능

### GitLab
- **자체 호스팅**: 무제한 LFS 지원 (저장소 크기 제한만 적용)
- **GitLab.com**: 무료 플랜에는 제한 있음

### Bitbucket
- **저장소 제한**: 2GB (LFS 포함)
- **대역폭**: 일일 제한 있음

**중요**: 각 서비스의 정책은 변경될 수 있으므로 공식 문서를 확인하세요.

---

## 자체 호스팅 LFS 서버

### GitLab CE/EE를 이용한 자체 호스팅
GitLab은 내장 LFS 서버를 제공하므로 추가 설정 없이 사용 가능합니다.

### MinIO/S3를 백엔드로 사용
```bash
# .lfsconfig 파일 생성 (저장소별 설정)
echo '[lfs]' > .lfsconfig
echo '    url = https://lfs.yourcompany.com/your/repo.git/info/lfs' >> .lfsconfig

# 설정 커밋
git add .lfsconfig
git commit -m "Configure custom LFS endpoint"
```

---

## 모범 사례

### 1. 적절한 파일 유형 선택
**LFS에 적합한 파일:**
- 이미지 파일 (.png, .jpg, .psd, .ai)
- 동영상/오디오 파일 (.mp4, .mov, .mp3)
- 3D 모델 및 텍스처 (.fbx, .obj, .blend)
- 머신러닝 모델 (.pt, .h5, .pkl)
- 데이터셋 압축 파일 (.zip, .tar.gz)

**LFS에 부적합한 파일:**
- 소스 코드 (.js, .py, .java)
- 설정 파일 (.json, .yml, .env)
- 문서 파일 (.md, .txt, .pdf)
- 작은 리소스 파일 (10MB 미만)

### 2. .gitattributes 관리 전략
```bash
# 체계적인 패턴 정의
# assets/ 디렉토리는 모두 LFS로 관리
assets/** filter=lfs diff=lfs merge=lfs -text

# 특정 확장자만 LFS로 관리
*.psd filter=lfs diff=lfs merge=lfs -text
*.mp4 filter=lfs diff=lfs merge=lfs -text

# 예외 처리
!*.md -text  # 마크다운 파일은 텍스트로 처리
```

### 3. 팀 협업 규칙
1. 모든 팀원이 Git LFS를 설치하도록 요구
2. 프로젝트 온보딩 문서에 LFS 설정 방법 포함
3. PR 템플릿에 대용량 파일 추가 확인 항목 포함

### 4. CI/CD 최적화
- 필요한 LFS 파일만 선택적 다운로드
- LFS 파일 캐싱 전략 구현
- 대역폭 사용량 모니터링

---

## 문제 해결 가이드

### 일반적인 오류 및 해결 방법

**오류 1**: "smudge filter lfs failed"
```bash
# 임시 해결: 스머지 필터 비활성화
GIT_LFS_SKIP_SMUDGE=1 git pull

# 필요한 파일만 수동 다운로드
git lfs fetch
git lfs checkout
```

**오류 2**: "This repository is over its data quota"
- 원인: 저장소 할당량 초과
- 해결: 불필요한 LFS 파일 정리 또는 할당량 증가

**오류 3**: "batch response: Authentication required"
- 원인: LFS 서버 인증 실패
- 해결: Git 자격 증명 확인 및 갱신

### 디버깅 도구
```bash
# LFS 환경 정보 확인
git lfs env

# LFS 파일 상태 확인
git lfs status

# LFS 파일 목록 확인
git lfs ls-files

# LFS 작업 로그 확인
git lfs logs last
```

### LFS 파일 크기 분석
```bash
# LFS 파일 크기별 정렬
git lfs ls-files | sort -k 3 -n

# 전체 LFS 사용량 확인
git lfs ls-files | awk '{sum += $3} END {print sum/1024/1024 " MB"}'
```

---

## Git LFS vs 기존 방법 비교

### Git LFS의 장점
1. **저장소 크기 관리**: 대용량 파일이 히스토리에 쌓이지 않음
2. **속도 향상**: 클론/풀 작업이 빠름
3. **버전 관리 유지**: 모든 파일 버전 추적 가능
4. **호스팅 서비스 통합**: GitHub, GitLab 등과 원활한 통합

### 대체 솔루션
1. **git-annex**: 더 복잡하지만 오프라인 작업과 분산 저장에 강점
2. **외부 저장소 연결**: 파일 서버, S3, Artifactory에 파일 저장
3. **릴리스 패키징**: 대용량 파일을 별도 패키지로 관리

### 선택 가이드라인
- **소규모 팀, 간단한 워크플로우**: Git LFS
- **복잡한 분산 환경, 오프라인 작업**: git-annex
- **아티팩트 관리 중심**: 외부 저장소 + 패키지 관리자

---

## 실전 예제 시나리오

### 게임 개발 프로젝트
```bash
# 게임 에셋 LFS 설정
git lfs track "Assets/**/*.png"
git lfs track "Assets/**/*.fbx"
git lfs track "Assets/**/*.wav"
git lfs track "Assets/**/*.mp4"

# 설정 커밋
git add .gitattributes
git commit -m "Track game assets with LFS"
```

### 머신러닝 프로젝트
```bash
# ML 모델 및 데이터셋 LFS 설정
git lfs track "models/*.h5"
git lfs track "models/*.pt"
git lfs track "models/*.pkl"
git lfs track "data/*.zip"
git lfs track "data/*.npz"

# CI에서 필요한 모델만 다운로드
git lfs fetch --include="models/production.pt"
git lfs checkout models/production.pt
```

### 디자인 에셋 프로젝트
```bash
# 디자인 파일 LFS 설정
git lfs track "*.psd"
git lfs track "*.ai"
git lfs track "*.sketch"
git lfs track "*.fig"

# 대용량 프레젠테이션 파일
git lfs track "presentations/*.pptx"
git lfs track "presentations/*.key"
```

---

## 명령어 레퍼런스

### 기본 명령어
```bash
# LFS 설치 및 초기화
git lfs install

# 파일 추적 패턴 추가
git lfs track "패턴"

# 추적 중인 패턴 확인
git lfs track

# LFS 파일 목록 확인
git lfs ls-files

# LFS 상태 확인
git lfs status
```

### 고급 명령어
```bash
# 특정 파일만 fetch
git lfs fetch --include="패턴"

# 특정 파일 제외하고 fetch
git lfs fetch --exclude="패턴"

# 포인터 파일을 실제 파일로 변환
git lfs checkout 패턴

# LFS 마이그레이션
git lfs migrate import --include="*.mp4,*.mov"

# LFS 환경 정보 확인
git lfs env

# LFS 로그 확인
git lfs logs show
```

### 문제 해결 명령어
```bash
# LFS 필터 재설정
git lfs update

# LFS 파일 재다운로드
git lfs pull

# LFS 캐시 정리
git lfs prune
```

---

## 결론

Git LFS는 대용량 파일을 포함하는 현대적인 소프트웨어 프로젝트에서 필수적인 도구입니다. 올바르게 사용하면 저장소 크기 관리, 작업 속도 향상, 팀 협업 효율성 등 여러 측면에서 혜택을 볼 수 있습니다.

### 핵심 요약

1. **적절한 사용**: LFS는 진정으로 대용량이고 버전 관리가 필요한 이진 파일에만 사용하세요.
2. **초기 설정**: 프로젝트 시작 단계에서 LFS를 설정하면 후속 문제를 예방할 수 있습니다.
3. **팀 협업**: 모든 팀원이 LFS를 이해하고 사용할 수 있도록 문서화와 교육이 필요합니다.
4. **CI/CD 통합**: 빌드 파이프라인에서 LFS 파일을 효율적으로 관리하는 전략을 수립하세요.
5. **비용 관리**: 호스팅 서비스의 할당량과 비용 구조를 이해하고 모니터링하세요.

### 최종 조언

Git LFS는 강력한 도구이지만, 모든 문제의 해결책은 아닙니다. 프로젝트의 특성과 요구사항에 맞게 LFS를 사용할지, 다른 솔루션을 선택할지 신중히 판단하세요. 작은 프로젝트에서는 LFS가 과도할 수 있으며, 매우 큰 파일(수십 GB 이상)은 전문적인 아티팩트 관리 시스템이 더 적합할 수 있습니다.

가장 중요한 것은 프로젝트 초기부터 파일 관리 전략을 수립하고, 팀원들과 명확하게 공유하는 것입니다. Git LFS를 올바르게 구현하면 대용량 파일로 인한 고통을 크게 줄이고, 더 효율적인 개발 워크플로우를 구축할 수 있습니다.