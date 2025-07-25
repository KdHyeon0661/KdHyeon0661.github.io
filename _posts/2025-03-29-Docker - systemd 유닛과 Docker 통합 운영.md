---
layout: post
title: Docker - systemd 유닛과 Docker 통합 운영
date: 2025-03-29 20:20:23 +0900
category: Docker
---
# 🛠️ systemd 유닛과 Docker 통합 운영

---

## 🔍 1. 왜 systemd와 연동하는가?

Docker는 기본적으로 `docker run`으로 컨테이너를 실행합니다.  
그러나 운영 환경에서는 다음 이유로 **systemd와의 통합이 중요**합니다:

| 이유 | 설명 |
|------|------|
| 자동 시작 | 시스템 부팅 시 자동으로 컨테이너 시작 가능 |
| 재시작 정책 | 실패 시 재시작, 충돌 복구 |
| 종속 제어 | 특정 서비스가 먼저 실행된 후 컨테이너 실행 가능 |
| 보안 컨텍스트 | 특정 사용자/그룹으로 컨테이너 실행 |
| 로깅 통합 | journald 로깅과 통합 (`journalctl`) |

---

## 🧱 2. 기본 systemd 서비스 파일 구조

보통 아래 경로에 `.service` 파일을 생성합니다:

```bash
/etc/systemd/system/myapp.service
```

### ✅ 기본 예시: nginx 컨테이너 실행

```ini
[Unit]
Description=My Docker Nginx container
After=docker.service
Requires=docker.service

[Service]
Restart=always
ExecStart=/usr/bin/docker run --rm --name nginx -p 80:80 nginx
ExecStop=/usr/bin/docker stop nginx

[Install]
WantedBy=multi-user.target
```

- `After=`, `Requires=`: Docker 데몬보다 이후에 실행
- `Restart=always`: 서비스 종료 시 자동 재시작
- `--rm`: 정지 시 삭제되므로 일반적으로는 사용하지 않음

---

## 📦 3. `docker run` 대신 `docker compose`로 실행하고 싶다면?

```ini
[Service]
ExecStart=/usr/local/bin/docker-compose -f /home/user/app/docker-compose.yml up
ExecStop=/usr/local/bin/docker-compose -f /home/user/app/docker-compose.yml down
WorkingDirectory=/home/user/app
Restart=always
```

→ `WorkingDirectory`를 반드시 지정해야 `docker-compose`가 `yml`을 찾을 수 있음

---

## 🔁 4. 자동 재시작 정책

systemd 단독 설정:

```ini
[Service]
Restart=on-failure
RestartSec=5
```

또는 Docker 자체에 지정:

```bash
docker run --restart=unless-stopped ...
```

| 옵션 | 설명 |
|------|------|
| `no` | 실패해도 재시작 안함 (기본) |
| `on-failure` | 비정상 종료 시만 재시작 |
| `always` | 항상 재시작 |
| `unless-stopped` | 수동 중단 시 제외하고 항상 재시작 |

---

## 📁 5. 여러 컨테이너 종속 관계 설정

```ini
[Unit]
Description=App container
After=postgresql-container.service
Requires=postgresql-container.service
```

→ `postgresql-container.service`가 먼저 실행되고, 준비된 이후에 앱 컨테이너 실행

---

## 🧪 6. 실제 예제: Flask 앱 실행

```ini
[Unit]
Description=Flask App Container
After=docker.service
Requires=docker.service

[Service]
Restart=always
ExecStart=/usr/bin/docker run --name flask-app -p 5000:5000 my-flask-image
ExecStop=/usr/bin/docker stop flask-app
ExecStopPost=/usr/bin/docker rm flask-app

[Install]
WantedBy=multi-user.target
```

---

## 🔐 7. 비root 사용자로 컨테이너 실행 (rootless Docker)

Docker 자체를 rootless로 실행하거나, podman/systemd를 사용할 수 있습니다.

```ini
[Service]
User=devuser
ExecStart=/usr/bin/docker run ...
```

> 단, 해당 유저가 `docker` 그룹에 있어야 하며, 권한 문제가 없도록 설정 필요

---

## 📌 8. systemd 명령 정리

| 명령 | 설명 |
|------|------|
| `sudo systemctl daemon-reload` | 서비스 파일 변경 시 리로드 |
| `sudo systemctl start myapp.service` | 시작 |
| `sudo systemctl enable myapp.service` | 부팅 시 자동 시작 등록 |
| `sudo systemctl status myapp.service` | 상태 확인 |
| `sudo journalctl -u myapp.service` | 로그 보기 |

---

## 🚫 9. 주의사항

| 항목 | 설명 |
|------|------|
| `--rm` 옵션 | 자동 삭제되므로 서비스 재시작 시 오류 발생 가능 |
| 로그 축적 | journald 로그가 커질 수 있음 → `logrotate` 검토 |
| `docker ps` 불일치 | systemd로 실행한 컨테이너는 `docker run`과 다를 수 있음 |
| `docker compose`는 상대 경로 주의 | `WorkingDirectory`로 해결 |

---

## 📚 참고 링크

- [Docker와 systemd 통합 공식 문서](https://docs.docker.com/config/containers/start-containers-automatically/)
- [systemd 서비스 유닛 문서](https://www.freedesktop.org/software/systemd/man/systemd.service.html)
- [journalctl 사용법](https://www.freedesktop.org/software/systemd/man/journalctl.html)

---

## ✅ 요약

| 항목 | 내용 |
|------|------|
| 서비스 생성 위치 | `/etc/systemd/system/` |
| 주요 명령 | `ExecStart`, `ExecStop`, `Restart` 등 |
| 구성 연동 | `After=docker.service`, `Requires=docker.service` |
| 실행 관리 | `systemctl start/stop/status`, `journalctl` |
| 보안 설정 | `User=`, `group=docker` |
| 여러 컨테이너 연동 | 종속성 선언으로 순차 실행 가능 |