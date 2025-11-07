---
layout: post
title: Git - Git LFS
date: 2025-02-19 21:20:23 +0900
category: Git
---
# Git LFS (Large File Storage) 완전 정리

## 1) Git LFS란?

**Git LFS**는 대용량/이진 파일을 Git 리포지토리 밖(LFS 서버)에 저장하고, Git에는 작은 **포인터(pointer)** 파일만 저장하는 확장이다.  
이미지/동영상/3D/AI 모델/압축 아카이브 등 **diff가 사실상 무의미**하고 **이력 복제 비용이 큰 파일**을 다룰 때 필수적이다.

### 왜 필요한가?

- Git은 스냅샷 기반이라 **커밋할수록 전체 이력에 파일 사본(또는 델타)** 이 축적된다.
- 이진 파일은 델타 효율이 낮아 저장소가 **급격히 비대**해지고 **clone/pull 속도**가 악화된다.
- LFS는 큰 바이너리를 외부에 저장 → Git에는 소형 포인터만 추적 → **저장소 크기/속도/협업성** 개선.

간단한 크기 모델:
$$
\text{RepoSize}_{\text{plain}} \approx \sum_{t=1}^{N} \text{bin}_t,\quad
\text{RepoSize}_{\text{LFS}} \approx \sum_{t=1}^{N} \text{ptr}_t \quad (\text{ptr}_t \ll \text{bin}_t)
$$

---

## 2) 설치 및 초기화

### macOS / Linux

```bash
# macOS
brew install git-lfs

# Ubuntu/Debian
sudo apt update && sudo apt install -y git-lfs
```

### Windows

- https://git-lfs.com 에서 설치 프로그램 다운로드

### LFS 활성화(리포지토리별 또는 전역)

```bash
git lfs install                # 현재 사용자 환경에 LFS 필터 등록
# 또는 리포지토리 내에서만
git lfs install --local
```

---

## 3) 핵심 동작: track → add/commit → push

### 3.1 추적 패턴 등록

```bash
git lfs track "*.mp4"
git lfs track "*.psd"
git lfs track "weights/*.bin"
```

이 명령은 `.gitattributes`에 LFS 필터 규칙을 추가한다:

```
*.mp4 filter=lfs diff=lfs merge=lfs -text
*.psd filter=lfs diff=lfs merge=lfs -text
weights/*.bin filter=lfs diff=lfs merge=lfs -text
```

> `.gitattributes` 자체를 반드시 커밋/푸시해야 팀 전체가 동일 규칙을 사용한다.

### 3.2 커밋 & 푸시

```bash
git add .gitattributes movie.mp4 assets/logo.psd weights/model.bin
git commit -m "Add media & model via LFS"
git push origin main
```

- 실제 대용량 바이너리는 **LFS 서버**로 업로드된다.
- Git에는 **아래와 같은 포인터 텍스트 파일**이 저장된다(예시):

```
version https://git-lfs.github.com/spec/v1
oid sha256:3f7c7...d88c   # 실제 바이너리의 콘텐츠 해시
size 48293021
```

---

## 4) LFS 포인터/필터(클린·스머지) 메커니즘 심화

- **clean 필터**: `git add` 시 원본 바이너리를 포인터 파일로 바꾸어 저장소에 기록, 실제 바이너리는 LFS 스토리지로 업로드.
- **smudge 필터**: `git checkout/clone/pull` 시 포인터를 감지하면 **원본 바이너리**를 LFS 서버에서 내려받아 작업 디렉터리에 복원.

환경변수로 스머지 생략 가능:
```bash
# 체크아웃 시 원본을 받지 않고 포인터만 두기(후속 수동 fetch를 위해)
GIT_LFS_SKIP_SMUDGE=1 git clone https://github.com/org/repo.git
# 또는
git config lfs.fetchexclude "*.mp4" "*.zip"
```

선택 다운로드:
```bash
git lfs fetch --include="assets/*.png" --exclude="*.mp4"
git lfs checkout assets/*.png     # 포인터 → 실제 파일로 복원
```

---

## 5) 기존 커밋된 대형 파일을 LFS로 **마이그레이션**

이미 Git 이력에 박혀 있는 대용량 파일을 LFS로 옮기려면 **히스토리 재작성**이 필요하다.

### 5.1 `git lfs migrate import`

```bash
# 모든 과거 커밋에서 zip/mp4를 LFS 포인터로 치환
git lfs migrate import --include="*.zip,*.mp4"
# 또는 특정 경로만
git lfs migrate import --include="assets/**"
```

- **공유 저장소**라면 반드시 팀과 조율 후 진행(히스토리 재작성 → 강제 푸시 필요).
- 수행 전 백업 브랜치 생성 권장:

```bash
git branch backup/pre-lfs-migrate
```

### 5.2 BFG Repo-Cleaner 대안

- 오래된 대형 파일 삭제/치환에 특화. LFS 전환도 가능.  
- 단, 프로젝트 상황에 따라 `git lfs migrate`가 더 직관적일 수 있다.

---

## 6) 클론/페치 전략 — 부분 다운로드 & 대역폭 최적화

### 6.1 스머지 스킵 & 지연 다운로드

```bash
GIT_LFS_SKIP_SMUDGE=1 git clone https://github.com/org/repo.git
cd repo
git lfs fetch --include="weights/**"
git lfs checkout weights/**         # 필요한 것만 실제로 받기
```

### 6.2 include/exclude 규칙

```bash
# 전역/로컬 설정
git config lfs.fetchinclude "assets/**,weights/**"
git config lfs.fetchexclude "*.mp4,*.mov"
```

### 6.3 shallow clone과의 조합

일반 Git의 얕은 클론(`--depth`)은 **Git 이력**을 줄여주지만, LFS 객체는 별도의 정책으로 내려받는다.  
대형 프로젝트에서는 **멀티 레이어 최적화**(얕은 Git + LFS include/exclude)를 병용하라.

---

## 7) GitHub/GitLab/Bitbucket & 요금/쿼터 고려

| 항목 | GitHub LFS(예시) |
|---|---|
| 저장/전송 쿼터 | 저장/전송 각각 기본 소량 제공, 초과 시 결제 필요 |
| 초과 시 증상 | push/clone 시 오류(“over quota”) |
| 팀 정책 | 리포 단위/Org 단위로 LFS 사용량 모니터링 & 예산 관리 필수 |

> 실제 쿼터·요금은 시점·플랜별로 상이하므로 프로젝트 시작 전 반드시 확인 후 정책 수립.

**실무 팁**

- **자주 바뀌는 거대 파일**(예: 매시간 생성되는 장기 보관 로그)은 LFS보다 외부 스토리지/Signed URL 접근이 더 경제적일 수 있다.
- **최소한의 LFS**: 모델 체크포인트 등 핵심 아티팩트만 LFS, 나머지는 아티팩트 스토리지(예: S3, GCS, Artifactory) 활용.

---

## 8) CI/CD에서의 LFS

### 8.1 GitHub Actions 예제

```yaml
# .github/workflows/ci.yml
name: CI
on: [push, pull_request]
jobs:
  build-test:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
        with:
          lfs: true          # LFS 파일을 자동으로 pull
          fetch-depth: 0     # 필요시 전체 이력
      - name: LFS 상태 확인
        run: git lfs ls-files
      - name: Build
        run: |
          npm ci
          npm run build
      - name: Test
        run: npm test -- --ci
```

- 캐시 최적화: LFS 오브젝트는 일반 캐시(Action cache)로 재활용이 어려운 편이라, **아티팩트 업로드/다운로드** 또는 **전용 캐시 전략**을 별도 설계.

### 8.2 “선택 다운로드”로 런타임 절약

```yaml
- uses: actions/checkout@v4
  with:
    lfs: false
- name: Fetch only model weights
  run: |
    git lfs install --local
    git lfs fetch --include="weights/**"
    git lfs checkout weights/**
```

---

## 9) 자가 호스팅 LFS — S3/MinIO/회사 내부

- **GitLab CE/EE**는 LFS 서버 내장(자체 호스팅 용이).
- **GitHub Enterprise Server**도 LFS 지원(버전/플랜 확인).
- 직접 LFS 서버(예: `lfs-test-server`, go-git-lfs) + **S3/MinIO** 백엔드 구성 가능.

### 9.1 예시: MinIO를 LFS 백엔드로

1) MinIO 서버(버킷: `lfs-bucket`) 준비  
2) 커스텀 LFS 서버에서 MinIO에 put/get 프록시  
3) 클라이언트 측 설정:

```bash
git config -f .lfsconfig lfs.url https://lfs.company.local/your/repo.git/info/lfs
git add .lfsconfig
git commit -m "Configure custom LFS endpoint"
```

`.lfsconfig`는 리포지토리 내 LFS 엔드포인트를 정의하여 **fork/clone 시에도 유지**되게 한다.

---

## 10) 모노레포/서브모듈과 LFS

- **모노레포**: 패키지별로 LFS 패턴을 세분화(예: `apps/web/assets/**`, `ml/weights/**`).  
  CI에서 **변경 감지** 후 필요한 LFS만 fetch/checkout.
- **서브모듈**: 서브모듈에서도 LFS 사용 가능. 상위 CI에서 `submodules: true`, `lfs: true` 조합으로 체크아웃 설정.

---

## 11) 보안/권한/가드레일

- **모든 협업자**가 LFS를 설치해야 한다. 미설치 시 포인터 텍스트만 내려받아 “파일이 깨진 것처럼” 보인다.
- **브랜치 보호 규칙**:  
  - LFS 포인터가 아닌 대형 바이너리가 실수로 Git에 들어오는 걸 막기 위해 **프리리시브 서버 훅** 또는 **CI 검사**(예: “100MB 이상 바이너리는 LFS 사용 여부 검사”)를 두자.
- **시크릿/민감 데이터**는 애초에 커밋 금지(포인터도 메타데이터를 노출할 수 있다).

예: CI에서 “LFS 포인터가 아닌 대형 파일” 차단

```bash
# scripts/check-large-non-lfs.sh
#!/usr/bin/env bash
set -euo pipefail
MAX=100000000 # 100MB
git ls-files -s | awk '{print $2, $4}' | while read mode path; do
  size=$(wc -c < "$path" 2>/dev/null || echo 0)
  if [ "$size" -gt "$MAX" ]; then
    # 포인터 파일 여부(첫 라인이 'version https://git-lfs.github.com/spec/v1'인지)
    if ! head -n1 "$path" | grep -q "git-lfs.github.com/spec/v1"; then
      echo "Found large non-LFS file: $path ($size bytes)"
      exit 1
    fi
  fi
done
```

---

## 12) 운영 팁 — 팀 컨벤션

- `.gitattributes`에서 **포맷을 가능한 좁게** 지정(불필요한 확장자 포함 금지).
- 자주 교체되는 거대 파일은 “버전 관리”보다 **아티팩트 스토리지**·**릴리스 페이지**·**패키지 레지스트리**로 이관 고려.
- **PR 템플릿**에 “대형 파일 추가 시 LFS 사용했는지 체크” 항목 추가.

---

## 13) 자주 만나는 오류 & 해결

### 13.1 “Encountered N file(s) that should have been pointers, but were not”

- LFS 추적 대상인데 **포인터가 아닌 원본**이 커밋됨.
- 해결:
  ```bash
  # 해당 파일을 LFS 포인터로 교체
  git lfs track "*.mp4"
  git add .gitattributes
  git add path/to/badfile.mp4
  git commit -m "Fix: track mp4 via LFS"
  # 과거 커밋까지 정정 필요 시:
  git lfs migrate import --include="*.mp4"
  git push --force-with-lease
  ```

### 13.2 “This repository is over its data quota”

- LFS 저장/전송 쿼터 초과. 결제/추가 데이터 팩 필요. 임시로는 **선택 fetch** 로 런너 트래픽 절약.

### 13.3 “smudge filter lfs failed”

- 네트워크/인증 문제 또는 LFS 엔드포인트/토큰 만료.
- 우회: 스머지 스킵 후 수동 fetch/checkout
  ```bash
  GIT_LFS_SKIP_SMUDGE=1 git pull
  git lfs fetch --include="needed/**"
  git lfs checkout needed/**
  ```

### 13.4 “batch response: rate limit exceeded”

- 짧은 시간에 대량 다운로드. **캐시**, **선택적 fetch**, **미러 LFS 서버** 고려.

---

## 14) Git LFS vs git-annex 간단 비교

| 항목 | Git LFS | git-annex |
|---|---|---|
| 철학 | “대형 파일은 외부, 포인터만 Git” | 콘텐츠 주소화로 다중 리모트/오프라인 전송에 강함 |
| 학습 곡선 | 낮음 | 높음 |
| 호스팅 | Git 호스팅과 자연스러운 통합 | 별도 생태계/운영 모델 |
| 일반적 웹/앱 팀 | 적합 | 과함 |
| 연구/필드 복제/대용량 분산 | 충분 | 더 유연 |

---

## 15) 실전 시나리오

### 15.1 3D/게임 에셋 프로젝트

```bash
git lfs track "*.fbx"
git lfs track "textures/**/*.png"
git add .gitattributes
git add textures models
git commit -m "Add assets via LFS"
git push
```

### 15.2 ML 모델/체크포인트

```bash
git lfs track "weights/**/*.pt"
git lfs track "datasets/**/*.zip"
git add .gitattributes
git commit -m "Track weights/datasets in LFS"
```

CI에서 필요한 가중치만:
```yaml
- uses: actions/checkout@v4
  with: { lfs: false }
- run: |
    git lfs install --local
    git lfs fetch --include="weights/resnet50.pt"
    git lfs checkout weights/resnet50.pt
```

---

## 16) 명령어 치트시트

```bash
# 설치/초기화
git lfs install

# 추적 패턴 등록
git lfs track "*.mp4" "weights/*.bin"

# 추적 목록 보기
git lfs ls-files

# 선택적 다운로드
GIT_LFS_SKIP_SMUDGE=1 git clone URL
git lfs fetch --include="weights/**"
git lfs checkout weights/**

# 마이그레이션(과거 이력 교체)
git lfs migrate import --include="*.zip,*.mp4"

# 상태/디버그
git lfs status
git lfs env

# 제거(필터 비활성화, 기존 포인터/객체는 남음)
git lfs uninstall
```

---

## 17) “처음부터 잘 쓰는” 체크리스트

- 프로젝트 첫 커밋 전에 **`.gitattributes`에 LFS 패턴 작성** → 초기부터 안전
- 팀 온보딩 문서에 **LFS 설치**를 포함
- CI에서 **LFS 필수 체크아웃** 또는 **선택 다운로드** 전략 문서화
- 대형 파일 방지 스크립트/서버 훅/브랜치 보호로 **가드레일** 구축
- 정기적으로 **LFS 사용량/트래픽** 점검 및 예산 관리

---

## 참고

- Git LFS 공식: https://git-lfs.com/
- GitHub Docs — LFS: https://docs.github.com/en/repositories/working-with-files/managing-large-files/about-large-files-on-github
- Git 호스팅별 LFS/과금 정책은 시점·플랜별 상이하므로 **프로젝트 시작 전 최신 문서 확인** 권장