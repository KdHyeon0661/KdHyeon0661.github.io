---
layout: post
title: AWS - EC2에 Nginx 웹 서버 설치하기
date: 2025-07-12 21:20:23 +0900
category: AWS
---
# EC2에 Nginx 웹 서버 설치하기

## 목표

- 이미 퍼블릭 접속 가능한 EC2(AL2) 인스턴스가 있다고 가정  
- Nginx 설치 → 부팅 자동 시작 → 보안 그룹 확인 → 브라우저 검증  
- 선택: User Data로 자동 설치, 리버스 프록시 구성, 로그/모니터링/HTTPS 개요

---

## 사전 준비 사항

### 체크리스트

| 항목 | 상태 |
|---|---|
| AWS 계정 | 필요 |
| EC2 인스턴스 (Amazon Linux 2) | 필요 |
| 키 페어(.pem) | 필요 |
| 퍼블릭 IP 또는 퍼블릭 DNS | 필요 |
| 보안 그룹 인바운드 80/TCP 허용 | 필요 |

본문은 EC2가 이미 생성되었고 퍼블릭 IP로 접속 가능한 상태를 가정한다.

---

## 1. EC2 접속하기

### Linux/macOS/WSL

```bash
chmod 400 MyKey.pem
ssh -i MyKey.pem ec2-user@<퍼블릭-IP-또는-퍼블릭DNS>
```

### Windows(PuTTY) 참고
- .pem → .ppk 변환(PuTTYgen)
- 또는 Windows Terminal + OpenSSH 사용 시 Linux/macOS와 동일

---

## 2. 시스템 패키지 업데이트

```bash
sudo yum update -y
```

커널 업데이트가 포함될 수 있다. 필요 시 재부팅:

```bash
sudo reboot
```

---

## 3. Nginx 설치

Amazon Linux 2는 amazon-linux-extras를 통해 Nginx 스트림을 활성화 후 설치한다.

```bash
sudo amazon-linux-extras enable nginx1
sudo yum install -y nginx
```

### 주요 경로

| 경로 | 설명 |
|---|---|
| /etc/nginx/nginx.conf | 메인 설정 |
| /etc/nginx/conf.d/ | 가상 호스트/서브 설정 |
| /usr/share/nginx/html/index.html | 기본 문서 루트 |
| /var/log/nginx/access.log | 접속 로그 |
| /var/log/nginx/error.log | 에러 로그 |
| /etc/systemd/system/nginx.service | 서비스 유닛 |

---

## 4. Nginx 서비스 시작 및 부팅 자동화

```bash
sudo systemctl start nginx
sudo systemctl enable nginx
sudo systemctl status nginx
```

active (running)이면 실행 중이다.

---

## 5. 보안 그룹(인바운드) 확인

콘솔:
1) EC2 → 인스턴스 선택 → 보안 탭 → 보안 그룹  
2) 인바운드 규칙 예시  
   - HTTP(80/TCP): 0.0.0.0/0  
   - SSH(22/TCP): 내 IP만 허용

CLI 예시:

```bash
MYIP=$(curl -s https://checkip.amazonaws.com)/32
aws ec2 authorize-security-group-ingress \
  --group-id sg-xxxxxxxx \
  --ip-permissions \
  "IpProtocol=tcp,FromPort=22,ToPort=22,IpRanges=[{CidrIp=\"$MYIP\"}]" \
  "IpProtocol=tcp,FromPort=80,ToPort=80,IpRanges=[{CidrIp=\"0.0.0.0/0\"}]"
```

운영에서는 22/TCP를 닫고 SSM(Session Manager) 사용을 권장한다.

---

## 6. 브라우저에서 접속 확인

브라우저에서 다음 주소로 접속한다.

```
http://<퍼블릭-IP-또는-퍼블릭DNS>
```

기본 Welcome to nginx 페이지가 보이면 성공이다.

---

## 7. Nginx 유용 명령어

| 명령어 | 설명 |
|---|---|
| sudo systemctl start nginx | 시작 |
| sudo systemctl stop nginx | 중지 |
| sudo systemctl restart nginx | 재시작 |
| sudo systemctl reload nginx | 설정 리로드(무중단 반영) |
| sudo systemctl status nginx | 상태 |
| sudo nginx -t | 설정 문법 검사 |
| sudo tail -f /var/log/nginx/access.log | 접속 로그 실시간 |
| sudo tail -f /var/log/nginx/error.log | 에러 로그 실시간 |

---

## 8. 기본 웹 페이지 수정

```bash
sudo bash -c 'cat >/usr/share/nginx/html/index.html' <<'HTML'
<!doctype html>
<html>
<head><meta charset="utf-8"><title>Hello AWS</title></head>
<body>
<h1>EC2에서 Nginx가 동작합니다.</h1>
<p>Amazon Linux 2 + Nginx 기본 페이지를 교체했습니다.</p>
</body>
</html>
HTML

sudo systemctl reload nginx
```

---

## 9. 방화벽 서비스 확인(선택)

Amazon Linux 2는 기본적으로 firewalld 비활성인 경우가 많다. 만약 활성이라면 80/TCP 허용:

```bash
sudo systemctl status firewalld
sudo firewall-cmd --zone=public --permanent --add-service=http
sudo firewall-cmd --reload
```

---

## 10. 전체 설치 스크립트(핵심)

```bash
sudo yum update -y
sudo amazon-linux-extras enable nginx1
sudo yum install -y nginx
sudo systemctl start nginx
sudo systemctl enable nginx
sudo systemctl status nginx
```

---

## 11. User Data로 자동 설치(새 인스턴스에 적용)

```bash
#!/bin/bash
set -eux
yum update -y
amazon-linux-extras enable nginx1
yum install -y nginx
systemctl enable --now nginx
cat >/usr/share/nginx/html/index.html <<'EOF'
<!doctype html><h1>Hello from EC2 + Nginx (User Data)</h1>
EOF
```

User Data는 Idempotent하게 작성한다. 복잡한 구성은 이미지 빌더 또는 IaC로 이전한다.

---

## 12. 리버스 프록시 구성 예시

```bash
sudo bash -c 'cat >/etc/nginx/conf.d/app.conf' <<'CONF'
server {
    listen 80 default_server;
    server_name _;

    access_log /var/log/nginx/app_access.log;
    error_log  /var/log/nginx/app_error.log;

    location / {
        proxy_set_header Host $host;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_pass http://127.0.0.1:3000;
    }

    location /healthz {
        return 200 'ok';
        add_header Content-Type text/plain;
    }
}
CONF

sudo nginx -t
sudo systemctl reload nginx
```

간단한 백엔드 예시:

```bash
cat >server.js <<'JS'
const http = require('http');
http.createServer((req,res)=>{res.end('Hello App via Nginx');}).listen(3000);
JS
node server.js &
```

---

## 13. HTTPS 개요(두 가지 경로)

1) ALB에서 TLS 종료  
   - ACM(AWS Certificate Manager)로 인증서 발급  
   - ALB 443 리스너 → 대상 그룹(EC2) 80  
   - 장점: 자동 갱신, 관리 간편

2) Nginx에 직접 인증서 배포  
   - 도메인을 EC2(EIP 권장)에 연결  
   - Let’s Encrypt 또는 상용 인증서 설치  
   - 배포 자동화와 갱신 스케줄 관리 필요

실무 권장: ALB + ACM으로 TLS 종료하고 EC2는 내부 HTTP로 단순화.

---

## 14. 로그/모니터링(CloudWatch)

- CloudWatch Logs 에이전트 또는 CloudWatch Agent로 /var/log/nginx/*.log 적재  
- 주요 지표: 4xx/5xx 비율, 응답 지연, CPU/메모리/네트워크

간단 지표 알람(CPU 예):

```bash
aws cloudwatch put-metric-alarm \
  --alarm-name "EC2-High-CPU" \
  --metric-name CPUUtilization \
  --namespace AWS/EC2 \
  --statistic Average --period 60 \
  --threshold 80 --comparison-operator GreaterThanThreshold \
  --dimensions Name=InstanceId,Value=i-xxxxxxxx \
  --evaluation-periods 3 \
  --alarm-actions arn:aws:sns:ap-northeast-2:123456789012:ops
```

---

## 15. 단순 지연 모델(정적 파일)

$$
T_{\text{resp}} \approx T_{\text{net}} + T_{\text{accept}} + T_{\text{read}} + T_{\text{send}}
$$

- CDN(CloudFront)을 쓰면 \(T_{\text{net}}\) 감소  
- 압축/캐시로 \(T_{\text{send}}\) 감소  
- Keep-Alive로 \(T_{\text{accept}}\) 감소

---

## 16. 정리·보안·운영 체크리스트

보안
- SSH는 내 IP만, 가능하면 SSM 사용  
- 보안 그룹: 80/443만 공개, 백엔드는 프라이빗  
- IMDSv2 필수화  
- EBS 암호화, 스냅샷 접근 제어

운영
- nginx -t 후 systemctl reload nginx  
- 로그 로테이션/보존 기간 관리  
- ALB 헬스체크 엔드포인트 /healthz 상시 200

비용
- 소규모는 t3.micro/t4g.micro 고려  
- 유휴 인스턴스, 미사용 EIP, 스냅샷 정리  
- 야간 자동 정지(EventBridge + Lambda)

---

## 17. 문제 해결

| 증상 | 점검 포인트 | 해결 |
|---|---|---|
| 브라우저 접속 불가 | SG 80/TCP 허용, Nginx 실행, OS 방화벽 | SG 수정, systemctl status nginx, firewalld 확인 |
| nginx -t 에러 | 설정 문법 오류 | conf 수정 후 재검증 |
| 403/404 | 문서 루트, 권한, SELinux | 권한/루트 확인, 커스텀 환경은 컨텍스트 확인 |
| 502/504 | 백엔드 다운, 포트 불일치, 타임아웃 | 백엔드 상태/포트 확인, proxy_* 타임아웃 조정 |
| SSH 불가 | SG 22, 키 권한(400), 사용자명 | chmod 400, ec2-user, SSM로 우회 |

---

## 18. 확장: ALB + Auto Scaling 개요

- ALB 앞단, Target Group으로 EC2 묶기  
- ASG로 2개 AZ에 최소 2대 이상 유지, 헬스체크 실패 시 자동 교체  
- 정적 자산은 S3 + CloudFront로 분리해 EC2 부하/비용 절감

---

## 19. Docker로 Nginx 실행(대안)

```bash
sudo yum install -y docker
sudo systemctl enable --now docker

sudo docker run --name nginx -p 80:80 -d nginx

mkdir -p ~/site && echo "<h1>Docker Nginx</h1>" > ~/site/index.html
sudo docker rm -f nginx
sudo docker run --name nginx -p 80:80 -v ~/site:/usr/share/nginx/html:ro -d nginx
```

컨테이너 운영 시 보안 그룹, IAM, 로그 수집, 업데이트 전략을 별도로 정의한다.

---

## 20. 실습 요약 플로우

1) SSH 접속 → yum update -y  
2) amazon-linux-extras enable nginx1 && yum install -y nginx  
3) systemctl enable --now nginx  
4) SG 80/TCP 확인  
5) 브라우저 검증  
6) index.html 교체  
7) 필요 시 프록시 conf, /healthz 구현  
8) CloudWatch Logs/지표/알람 구성

---

## 설치 완료 후 체크리스트

| 항목 | 확인 |
|---|---|
| EC2 인스턴스 준비 | 완료 |
| SSH 또는 SSM 접속 성공 | 완료 |
| Nginx 설치 및 활성화 | 완료 |
| 인바운드 80/TCP 허용 | 완료 |
| 브라우저 접속 성공 | 완료 |
| 로그/알람/모니터링 구성 | 필요 시 진행 |
| 정적 파일 캐시/압축 설정 | 필요 시 진행 |
| HTTPS(ACM/ALB 또는 Nginx) | 필요 시 진행 |

---

## 부록 A. Nginx 압축/캐시 헤더 예시

```nginx
# /etc/nginx/conf.d/static.conf
server {
    listen 80;
    server_name _;

    root /usr/share/nginx/html;

    gzip on;
    gzip_types text/plain text/css application/javascript application/json image/svg+xml;
    gzip_min_length 1024;

    location ~* \.(css|js|png|jpg|jpeg|gif|svg)$ {
        expires 7d;
        add_header Cache-Control "public, max-age=604800";
    }

    location /healthz { return 200 'ok'; add_header Content-Type text/plain; }
}
```

```bash
sudo nginx -t && sudo systemctl reload nginx
```

---

## 부록 B. IMDSv2 강제

```bash
aws ec2 modify-instance-metadata-options \
  --instance-id i-xxxxxxxx \
  --http-endpoint enabled \
  --http-tokens required
```

---

## 부록 C. 간단 비용 근사

$$
\text{월 비용} \approx c_{\text{EC2}} \cdot H + C_{\text{EBS}} + C_{\text{데이터전송}}
$$

- \(c_{\text{EC2}}\): 인스턴스 시간당 단가(USD/h)  
- \(H\): 월 사용 시간(시간)  
- 테스트 종료 후 인스턴스, 볼륨, EIP, 스냅샷 잔존 여부를 점검한다.