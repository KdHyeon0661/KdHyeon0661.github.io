---
layout: post
title: AWS - EC2에 Nginx 웹 서버 설치하기
date: 2025-07-12 21:20:23 +0900
category: AWS
---
# 🌐 EC2에 Nginx 웹 서버 설치하기 (Amazon Linux 2 기준)

---

## 🧭 목표

Amazon EC2 인스턴스를 생성하고, 해당 인스턴스에 **Nginx 웹 서버를 설치**하여  
브라우저에서 `http://<EC2 퍼블릭 IP>`로 접속 시 **“Welcome to Nginx” 페이지**가 보이도록 합니다.

---

## 🛠️ 사전 준비 사항

### ✅ 준비물 체크리스트

| 항목 | 상태 |
|------|------|
| AWS 계정 | ✅ |
| EC2 인스턴스 (Amazon Linux 2) | ✅ |
| 키 페어(.pem) | ✅ |
| 퍼블릭 IP | ✅ |
| 포트 80 허용 (보안 그룹) | ✅ |

> 이 글에서는 EC2가 이미 생성되어 있고, 퍼블릭 IP로 접속 가능한 상태라고 가정합니다.

---

## 1️⃣ EC2 접속하기

터미널 또는 SSH 클라이언트를 사용해 EC2에 접속합니다.

### Linux/macOS/Mac에서 접속

```bash
chmod 400 MyKey.pem
ssh -i MyKey.pem ec2-user@<퍼블릭-IP>
```

### Windows에서 접속

- PuTTY 사용 시 `.pem → .ppk` 변환 필요
- 또는 Windows Terminal에서 WSL 사용 가능

---

## 2️⃣ 시스템 패키지 업데이트

EC2 인스턴스는 최신 상태가 아닐 수 있으므로, 먼저 패키지를 업데이트합니다.

```bash
sudo yum update -y
```

---

## 3️⃣ Nginx 설치

Amazon Linux 2의 기본 저장소를 통해 설치합니다.

```bash
# nginx 설치
sudo amazon-linux-extras enable nginx1
sudo yum install nginx -y
```

설치가 완료되면 Nginx 설정 파일과 관련 파일들이 다음 위치에 생성됩니다:

| 위치 | 용도 |
|------|------|
| `/etc/nginx/nginx.conf` | 메인 설정 파일 |
| `/usr/share/nginx/html/index.html` | 기본 웹 페이지 |
| `/var/log/nginx/` | 로그 디렉터리 |
| `/etc/nginx/conf.d/` | 추가 설정 |

---

## 4️⃣ Nginx 서비스 시작

```bash
# nginx 시작
sudo systemctl start nginx

# 자동 시작 설정 (재부팅 후에도 실행됨)
sudo systemctl enable nginx

# 상태 확인
sudo systemctl status nginx
```

상태가 `active (running)`이면 정상적으로 실행 중입니다.

---

## 5️⃣ 보안 그룹 설정 확인 (포트 80)

### AWS 콘솔에서 확인
1. EC2 → 인스턴스 선택 → 보안 → 보안 그룹
2. **인바운드 규칙 편집** 클릭
3. 다음 규칙 추가 (이미 있다면 생략):

| 유형 | 프로토콜 | 포트 범위 | 소스 |
|------|----------|------------|------|
| HTTP | TCP | 80 | 0.0.0.0/0 (모두 허용) |

> **⚠️ 보안상 SSH(22)는 반드시 내 IP로만 제한하세요.**

---

## 6️⃣ 브라우저에서 접속 확인

웹 브라우저를 열고 다음 주소로 접속해보세요:

```
http://<EC2 퍼블릭 IP>
```

### 예상 결과

```html
Welcome to nginx!
If you see this page, the nginx web server is successfully installed and working.
```

위 페이지가 보이면 성공입니다 ✅

---

## 7️⃣ Nginx 관련 유용 명령어

| 명령어 | 설명 |
|--------|------|
| `sudo systemctl start nginx` | 서비스 시작 |
| `sudo systemctl stop nginx` | 서비스 중지 |
| `sudo systemctl restart nginx` | 서비스 재시작 |
| `sudo systemctl status nginx` | 상태 확인 |
| `sudo nginx -t` | 설정파일 문법 확인 |
| `sudo tail -f /var/log/nginx/access.log` | 접속 로그 실시간 보기 |

---

## 8️⃣ Nginx 기본 웹 페이지 수정

기본 index.html을 수정하면 웹 서버에 표시되는 페이지를 바꿀 수 있습니다.

```bash
# index.html 파일 열기
sudo nano /usr/share/nginx/html/index.html
```

예시:

```html
<html>
  <head><title>Hello AWS!</title></head>
  <body>
    <h1>🎉 EC2에서 Nginx가 잘 작동합니다!</h1>
  </body>
</html>
```

파일 저장 후 브라우저 새로고침하면 새로운 메시지가 표시됩니다.

---

## 9️⃣ 방화벽 서비스 확인 (선택 사항)

Amazon Linux 2는 기본적으로 `firewalld`가 비활성화되어 있지만, 만약 활성화되어 있다면 HTTP 트래픽을 허용해야 합니다.

```bash
# 상태 확인
sudo systemctl status firewalld

# 허용 (활성화되어 있다면)
sudo firewall-cmd --zone=public --permanent --add-service=http
sudo firewall-cmd --reload
```

---

## 🔁 전체 명령어 요약 스크립트

아래는 EC2에 접속 후 실행할 전체 설치 스크립트입니다.

```bash
# 시스템 업데이트
sudo yum update -y

# Nginx 설치
sudo amazon-linux-extras enable nginx1
sudo yum install nginx -y

# Nginx 시작 및 활성화
sudo systemctl start nginx
sudo systemctl enable nginx

# 상태 확인
sudo systemctl status nginx
```

---

## ✅ 설치 완료 후 체크리스트

| 항목 | 확인 여부 |
|------|------------|
| EC2 인스턴스 생성 완료 | ✅ |
| SSH 접속 성공 | ✅ |
| Nginx 설치 및 실행 | ✅ |
| 포트 80 열림 확인 | ✅ |
| 브라우저에서 접속 성공 | ✅ |

---

## 🧠 추가 팁

### 🔒 HTTPS 적용하기
- Let’s Encrypt + Certbot 사용으로 무료 SSL 인증서 적용 가능
- EC2에서 도메인 연결 후 진행

### 🚀 CI/CD 배포 자동화
- Nginx를 정적 웹 서버로 사용하면, GitHub Actions나 CodePipeline을 통해 자동 배포 가능

### 📦 Docker로 설치도 가능
```bash
sudo yum install docker -y
sudo systemctl start docker
sudo docker run --name nginx -p 80:80 -d nginx
```