---
layout: post
title: AspNet - NET Core CI/CD 구성
date: 2025-04-29 19:20:23 +0900
category: AspNet
---
# 🚀 ASP.NET Core CI/CD 구성 (with GitHub Actions)

---

## ✅ 1. CI/CD란?

| 용어 | 설명 |
|------|------|
| **CI (Continuous Integration)** | 소스 코드 변경 시 자동으로 **빌드 + 테스트** |
| **CD (Continuous Delivery/Deployment)** | 빌드 결과물을 **자동 배포** (또는 준비)

**목적**:  
- 사람 개입 없이 안정적으로 배포  
- 코드 품질 유지  
- 운영 환경과 개발 환경 일관성 보장

---

## 🧱 2. 주요 구성 요소

| 구성 요소 | 설명 |
|-----------|------|
| GitHub Actions | GitHub 제공 자동화 워크플로 |
| `dotnet build/test/publish` | ASP.NET Core 빌드/배포 명령 |
| Azure App Service, FTP, Docker | 배포 대상 |
| Secrets | 인증 정보 저장 (안전하게)

---

## 📁 3. 프로젝트 구조 예시

```bash
MyApp/
├── MyApp.csproj
├── Program.cs
├── ...
├── .github/
│   └── workflows/
│       └── ci-cd.yml   👈 CI/CD 정의 파일
```

---

## ✍️ 4. 기본 GitHub Actions 워크플로 예시

`.github/workflows/ci-cd.yml`

```yaml
name: ASP.NET Core CI/CD

on:
  push:
    branches: [ main ]
  pull_request:
    branches: [ main ]

jobs:
  build-deploy:
    runs-on: ubuntu-latest

    steps:
    - name: Checkout code
      uses: actions/checkout@v3

    - name: Setup .NET SDK
      uses: actions/setup-dotnet@v4
      with:
        dotnet-version: '8.0.x'

    - name: Restore dependencies
      run: dotnet restore

    - name: Build
      run: dotnet build --no-restore

    - name: Test
      run: dotnet test --no-build --verbosity normal

    - name: Publish
      run: dotnet publish -c Release -o ./publish

    - name: Deploy to Azure Web App
      uses: azure/webapps-deploy@v2
      with:
        app-name: ${{ secrets.AZURE_WEBAPP_NAME }}
        publish-profile: ${{ secrets.AZURE_WEBAPP_PUBLISH_PROFILE }}
        package: ./publish
```

---

## 🔐 5. Secrets 설정 방법

GitHub Repository → **Settings → Secrets → Actions** 에서 다음 키 추가:

| 키 | 설명 |
|----|------|
| `AZURE_WEBAPP_NAME` | 앱 서비스 이름 |
| `AZURE_WEBAPP_PUBLISH_PROFILE` | Azure Portal에서 Export한 배포용 Profile (XML 형식)

---

## ☁️ 6. Azure Publish Profile 받기

1. Azure Portal → App Service → **"배포 센터"**
2. "배포 자격 증명" → **"프로파일 다운로드"**
3. base64 인코딩 없이 XML을 그대로 **Secrets에 입력**

---

## 🧪 7. 테스트 자동화 예시

`dotnet test` 명령은 다음과 같은 구조를 지원:

```bash
MyApp.Tests/
├── MyApp.Tests.csproj
└── UnitTest1.cs
```

> GitHub Actions는 모든 `.csproj`에서 테스트를 자동으로 수행 가능

---

## 📦 8. 배포 대상 변경 예시

### ▶ Docker 이미지로 빌드 후 Docker Hub 배포

```yaml
- name: Build Docker image
  run: docker build -t myuser/myapp:latest .

- name: Push to Docker Hub
  run: |
    echo "${{ secrets.DOCKER_PASSWORD }}" | docker login -u "${{ secrets.DOCKER_USERNAME }}" --password-stdin
    docker push myuser/myapp:latest
```

---

## 🧠 9. 워크플로 커스터마이징 팁

| 기능 | 설명 |
|------|------|
| `matrix` | 여러 OS/.NET 버전 테스트 |
| `needs:` | 단계 간 의존성 설정 |
| `on:` 조건 수정 | push, pull_request, tag 등 다양한 조건 |
| 배포 분기 제한 | `if: github.ref == 'refs/heads/main'` 사용 |

---

## ✅ 10. 요약

| 항목 | 내용 |
|------|------|
| 자동화 도구 | GitHub Actions |
| 핵심 명령어 | `dotnet restore/build/test/publish` |
| 배포 방식 | Azure App Service, Docker, FTP 등 |
| 보안 | Secrets 사용 |
| 실무 활용 | 코드 머지 시 자동 배포, 테스트 포함 가능 |