---
layout: post
title: Git - Git LFS
date: 2025-02-19 21:20:23 +0900
category: Git
---
# 💾 Git LFS (Large File Storage) 완전 정리

---

## 📌 Git LFS란?

> Git LFS는 Git 저장소에서 **대용량 파일을 별도로 저장**하고,  
> Git에는 해당 파일의 **포인터만 저장**하는 방식입니다.

### ✅ 왜 필요할까?

- Git은 모든 파일의 **버전을 저장**함
- 이로 인해 **이미지, 동영상, AI 모델** 같은 파일을 커밋할수록 저장소가 **급격히 커지고 느려짐**
- Git LFS는 이런 파일을 Git 외부에 저장하고, Git은 **작은 링크(포인터)만 추적**함

---

## 📦 설치 방법

### ✅ macOS / Linux

```bash
brew install git-lfs  # macOS
sudo apt install git-lfs  # Ubuntu
```

### ✅ Windows

- [https://git-lfs.com](https://git-lfs.com) 에서 설치 프로그램 다운로드

### ✅ 초기화

```bash
git lfs install
```

---

## 🧰 사용 방법

### 1. 추적할 파일 유형 등록

```bash
git lfs track "*.mp4"
git lfs track "*.psd"
```

- 이는 `.gitattributes` 파일에 다음과 같이 기록됩니다:

```
*.mp4 filter=lfs diff=lfs merge=lfs -text
```

### 2. 해당 파일 추가 및 커밋

```bash
git add movie.mp4
git commit -m "Add video"
git push origin main
```

> 실제 `.mp4` 파일은 **LFS 서버**에 업로드되고, Git에는 포인터 파일이 저장됩니다.

---

## 🧪 동작 방식 요약

- `git lfs track "*.ext"` → 특정 확장자 LFS 추적
- Git 커밋 시: 실제 파일 → LFS 서버, 포인터 → Git 저장소
- Git 클론 시: 포인터 파일 기반으로 원본 파일도 자동 다운로드

---

## 🔁 기존 Git 파일을 LFS로 이전하려면?

기존 커밋된 대용량 파일도 LFS로 옮기려면 다음 명령어 사용:

```bash
git lfs migrate import --include="*.zip,*.mp4"
```

> 기존 Git 히스토리를 변경하므로 **백업 필수!**

---

## 🔗 Git LFS 저장소 종류

| 저장소 | 설명 |
|--------|------|
| GitHub LFS | GitHub 기본 지원, 무료 용량 1GB, 월 1GB 전송 |
| GitLab LFS | GitLab도 자체 호스팅 가능 |
| Bitbucket LFS | 제한 있음 |
| 자체 LFS 서버 | 기업 환경에서 사용, S3, MinIO 등으로 구현 가능 |

---

## 💡 주의사항 및 한계

| 주의점 | 설명 |
|--------|------|
| 무료 용량 제한 | GitHub: 1GB 저장, 월 1GB 전송 → 초과 시 요금 발생 |
| 이력 보기 불편 | Git에서 포인터만 보이므로 diff, blame 불편함 |
| 이진파일 충돌 관리 어려움 | 수동으로 병합하거나 덮어씌워야 함 |
| 모든 사용자가 LFS 설치 필요 | 설치하지 않으면 파일 깨져 보임 (pointer만 노출됨) |

---

## 🧪 실무 예시

### 예: 3D 게임 프로젝트

```bash
git lfs track "*.fbx"
git lfs track "*.png"
git add .gitattributes
git add assets/
git commit -m "Add textures and models"
git push
```

---

## 🧰 CLI 명령어 정리

| 명령어 | 설명 |
|--------|------|
| `git lfs install` | 초기화 |
| `git lfs track "*.psd"` | 확장자 등록 |
| `git lfs ls-files` | 추적 중인 파일 목록 |
| `git lfs migrate import --include="*.zip"` | 기존 파일 LFS로 옮기기 |
| `git lfs uninstall` | LFS 제거 |
| `git lfs status` | LFS 파일 상태 확인 |

---

## 🧪 Git LFS vs Git 비교

| 항목 | Git | Git LFS |
|------|-----|---------|
| 저장 방식 | 파일 전체 + 이력 | 포인터 저장, 실제 파일은 별도 저장소 |
| 대용량 처리 | 느려짐 | 쾌적함 |
| 이진파일 | diff 불가 | 저장은 가능 |
| 저장소 크기 | 빠르게 증가 | 상대적으로 작음 |

---

## 🔐 실무 팁

- 프로젝트 초기부터 LFS를 설정해두는 것이 좋음
- `.gitattributes`는 **Git으로 함께 커밋되어 팀 전체 공유**
- 협업 시 모든 팀원이 LFS를 설치해야 함

---

## 🔗 참고 링크

- [Git LFS 공식 사이트](https://git-lfs.com/)
- [GitHub Docs - Git LFS](https://docs.github.com/en/repositories/working-with-files/managing-large-files/about-large-files-on-github)
- [GitHub 요금제 - LFS 사용량](https://github.com/pricing)