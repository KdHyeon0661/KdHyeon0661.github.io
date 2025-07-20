---
layout: post
title: Docker - Docker 설치
date: 2024-12-28 20:20:23 +0900
category: Docker
---
# 🐳 Docker 설치 방법 (Windows / Linux)

---

## 🪟 Windows에서 Docker 설치

### ✅ 설치 전 확인 사항

| 항목 | 설명 |
|------|------|
| 운영체제 | Windows 10 Pro, Enterprise, Education 이상 (Home도 가능하나 WSL2 필요) |
| 가상화 지원 | BIOS/UEFI에서 Virtualization (VT-x, AMD-V) 활성화 필요 |
| Hyper-V | Windows 10 Pro 이상이면 Hyper-V 사용 가능 |
| Windows 10 Home | Hyper-V는 불가. 대신 **WSL2(Windows Subsystem for Linux 2)** 기반 설치 필요 |

---

### ✅ 설치 방법 1: Windows 10 Pro 이상 (Hyper-V 지원)

1. [Docker Desktop 공식 사이트](https://www.docker.com/products/docker-desktop/)에서 다운로드
2. 설치 실행
3. `Hyper-V` 및 `WSL2` 옵션이 함께 활성화됨
4. 설치 완료 후 재부팅
5. Docker Desktop 실행 → 트레이 아이콘에서 정상 작동 확인

---

### ✅ 설치 방법 2: Windows 10 Home (WSL2 기반)

#### 🔹 1. WSL2 활성화

```powershell
wsl --install
```

> Windows 10 Home에서는 WSL2가 필수입니다.

#### 수동 설치 절차 (구형 버전 호환용):

```powershell
dism.exe /online /enable-feature /featurename:Microsoft-Windows-Subsystem-Linux /all /norestart
dism.exe /online /enable-feature /featurename:VirtualMachinePlatform /all /norestart
```

- 위 명령 후 재부팅
- 최신 WSL2 커널 업데이트 패키지 다운로드:
  [https://aka.ms/wsl2kernel](https://aka.ms/wsl2kernel)

#### 🔹 2. WSL2 기본 버전으로 설정

```powershell
wsl --set-default-version 2
```

#### 🔹 3. 우분투 등 리눅스 배포판 설치

```powershell
wsl --install -d Ubuntu
```

> 또는 Microsoft Store에서 `Ubuntu`, `Debian`, `Alpine` 등 설치 가능

#### 🔹 4. Docker Desktop 설치

- [https://www.docker.com/products/docker-desktop/](https://www.docker.com/products/docker-desktop/) 에서 설치
- 설치 중 "Use WSL 2 based engine" 선택
- 설치 후 Docker Desktop 실행 → 우측 하단 아이콘에서 `Settings > Resources > WSL Integration` 활성화

---

### ⚠️ Docker Desktop 설치 후 주의사항

- Docker CLI는 `PowerShell`, `CMD`, `Windows Terminal`, 또는 `WSL2 터미널`에서 사용 가능
- `docker version`, `docker run hello-world`로 설치 확인
- Windows Home에서는 Hyper-V 기능이 없으므로 반드시 **WSL2가 활성화되어야** 함

---

## 🐧 Linux에서 Docker 설치 (예: Ubuntu)

### ✅ 1. 기존 패키지 제거 (필요시)

```bash
sudo apt remove docker docker-engine docker.io containerd runc
```

---

### ✅ 2. 저장소 설정

```bash
sudo apt update
sudo apt install -y ca-certificates curl gnupg lsb-release

# Docker GPG 키 추가
sudo mkdir -p /etc/apt/keyrings
curl -fsSL https://download.docker.com/linux/ubuntu/gpg \
  | sudo gpg --dearmor -o /etc/apt/keyrings/docker.gpg

# 저장소 등록
echo \
  "deb [arch=$(dpkg --print-architecture) \
  signed-by=/etc/apt/keyrings/docker.gpg] \
  https://download.docker.com/linux/ubuntu \
  $(lsb_release -cs) stable" \
  | sudo tee /etc/apt/sources.list.d/docker.list > /dev/null
```

---

### ✅ 3. Docker 설치

```bash
sudo apt update
sudo apt install -y docker-ce docker-ce-cli containerd.io docker-buildx-plugin docker-compose-plugin
```

---

### ✅ 4. Docker 권한 설정 (sudo 없이 실행 가능하게)

```bash
sudo usermod -aG docker $USER
# 변경 사항 적용을 위해 로그아웃 또는 재부팅
```

---

### ✅ 5. 설치 확인

```bash
docker --version
docker run hello-world
```

---

## ✅ 요약

| OS | 설치 방식 | 하이퍼바이저 사용 여부 |
|----|-----------|-------------------------|
| Windows 10 Pro 이상 | Docker Desktop (Hyper-V) | O |
| Windows 10 Home | Docker Desktop (WSL2) | X (Hyper-V 없음) |
| Ubuntu (Linux) | 패키지로 직접 설치 | X (커널 직접 사용) |

---

## 🔍 참고

- [공식 Docker 설치 가이드](https://docs.docker.com/get-docker/)
- WSL2에서 Docker를 사용하는 경우, 컨테이너는 WSL2 배포판 안에서 실행됩니다.
- Linux는 Docker가 **가장 자연스럽고 성능도 가장 좋은 환경**입니다.