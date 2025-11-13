---
layout: post
title: Docker - systemd 유닛과 Docker 통합 운영
date: 2025-03-29 20:20:23 +0900
category: Docker
---
# systemd 유닛과 Docker 통합 운영

## 0. 왜 systemd 연동인가? (운영 관점 재정의)

| 요구 | Docker만 | systemd 연동 시 이점 |
|---|---|---|
| 부팅 시 자동 시작 | `--restart`에 의존 | **부팅 타겟·의존성** 기반 제어 (`After=`, `Requires=`) |
| 장애 복구 | 컨테이너 정책 중심 | **StartLimit/RestartSec/Watchdog** 등 세밀 제어 |
| 종속 서비스 | 수동 스크립트 | **다른 유닛과 의존관계** (`Requires=`, `BindsTo=`, `PartOf=`) |
| 로깅/감사 | Docker 로깅 드라이버 | **journald 통합** + `journalctl -u` 표준 운영 절차 |
| 보안 맥락 | 컨테이너 격리 위주 | **systemd 샌드박싱**(CLI 실행 컨텍스트 하드닝) |
| 배포 파이프라인 | 스크립트 | **pull→run 순서화**, 이미지 롤백·그레이스풀 스톱 |

---

## 1. 기초: 단일 컨테이너를 systemd로 실행

### 1.1 최소 예시 (nginx)
`/etc/systemd/system/nginx-container.service`
```ini
[Unit]
Description=Nginx container (Docker)
After=network-online.target docker.service
Wants=network-online.target
Requires=docker.service

[Service]
Type=exec
Restart=always
RestartSec=5s
ExecStart=/usr/bin/docker run --name nginx \
  -p 80:80 \
  --health-cmd='curl -fsS http://127.0.0.1/ || exit 1' \
  --health-interval=10s --health-retries=3 --health-timeout=2s \
  nginx:1.25
ExecStop=/usr/bin/docker stop -t 10 nginx
ExecStopPost=/usr/bin/docker rm -f nginx

[Install]
WantedBy=multi-user.target
```

> 팁
> - `--rm`는 **재시작 시 컨테이너가 없어** 실패할 수 있으므로 대신 `ExecStopPost=rm`로 정리.
> - HealthCheck를 넣어 **컨테이너 내부 상태**를 운영 로그에서 추적 가능.

활성화/시작:
```bash
sudo systemctl daemon-reload
sudo systemctl enable --now nginx-container.service
sudo systemctl status nginx-container.service
sudo journalctl -u nginx-container.service -f
```

---

## 2. 운영형 패턴: 사전 pull + 존재 시 중복 제거 + 명시적 옵션

단일 파일 내 *pull → run* 절차를 넣으면 **업그레이드/재배포**가 간결해진다.

`/etc/systemd/system/web.service`
```ini
[Unit]
Description=Web API container (Docker)
After=network-online.target docker.service
Wants=network-online.target
Requires=docker.service

[Service]
Type=exec
Restart=always
RestartSec=5s
# 1. 이미지 최신화(혹은 특정 태그 유지)
ExecStartPre=/usr/bin/docker pull my-registry.example.com/myorg/web:1.2.3
# 2. 기존 컨테이너 정리(있을 때만)
ExecStartPre=/usr/bin/docker rm -f web || true
# 3. 실행
ExecStart=/usr/bin/docker run --name web \
  --label app=web --label version=1.2.3 \
  --log-driver=json-file --log-opt max-size=10m --log-opt max-file=5 \
  --cpus=1.0 --memory=512m --pids-limit=256 \
  --read-only --tmpfs /tmp \
  --cap-drop=ALL --cap-add=NET_BIND_SERVICE \
  -p 8080:8080 \
  my-registry.example.com/myorg/web:1.2.3
# 4. 그레이스풀 종료 및 후처리
ExecStop=/usr/bin/docker stop -t 15 web
ExecStopPost=/usr/bin/docker rm -f web

# systemd 자체 리소스 한도(선택)
LimitNOFILE=65535
TasksMax=4096

[Install]
WantedBy=multi-user.target
```

> 포인트
> - **이미지 태그 고정**으로 재현성 확보. 롤백은 태그만 바꿔 재로드.
> - Docker 로깅 회전 옵션을 명시해 **디스크 폭주 방지**.
> - **리소스 제한(CPU/메모리/pids)** 을 유닛 단계에서 명시하면 운영 기준이 명확해진다.

---

## 3. `.env` / 환경변수 파일과 시크릿 연동

### 3.1 systemd `EnvironmentFile` + Docker env 전달
`/etc/systemd/system/api.service`
```ini
[Unit]
Description=API container
After=docker.service
Requires=docker.service

[Service]
EnvironmentFile=/etc/myapp/api.env
ExecStartPre=/usr/bin/docker rm -f api || true
ExecStart=/usr/bin/docker run --name api \
  -e DB_HOST=${DB_HOST} -e DB_USER=${DB_USER} -e DB_PASS=${DB_PASS} \
  myorg/api:2.0.0
ExecStop=/usr/bin/docker stop -t 10 api
ExecStopPost=/usr/bin/docker rm -f api

Restart=always
RestartSec=5s

[Install]
WantedBy=multi-user.target
```

`/etc/myapp/api.env`
```
DB_HOST=postgresql.internal
DB_USER=apiuser
DB_PASS=change_me
```

> 보안: 민감정보는 **파일 권한 0640(root:root)** 로, 선호는 외부 비밀 저장소/Vault.
> Swarm/K8s 사용 시 **Secrets 리소스** 권장.

---

## 4. `docker compose`를 systemd로 제어

Compose v2는 `docker compose`(단일 바이너리) 호출을 권장한다.

`/etc/systemd/system/myproject.service`
```ini
[Unit]
Description=Compose stack: myproject
After=network-online.target docker.service
Wants=network-online.target
Requires=docker.service

[Service]
Type=exec
WorkingDirectory=/srv/myproject
# 파일 권한 적용(선택)
UMask=0027
# 환경파일 주입(선택)
EnvironmentFile=/srv/myproject/.env
# Compose Up: 포그라운드 모드로 systemd에 붙임
ExecStart=/usr/bin/docker compose --ansi=never up --no-color
# Down 시그널(그레이스풀)
ExecStop=/usr/bin/docker compose down
Restart=always
RestartSec=3s

[Install]
WantedBy=multi-user.target
```

> 주의
> - `WorkingDirectory`를 **반드시** 설정(상대경로 문제 방지).
> - 로그는 `journalctl -u myproject.service` 로 확인(각 컨테이너 로그는 `docker logs` 병행).
> - HealthCheck는 compose 파일 내 각 서비스에 정의하고, 의존관계는 `depends_on` + **healthcheck 조건**을 활용.

---

## 5. 종속성/순서 제어 (DB → 앱 → 프록시)

### 5.1 유닛 간 의존
```ini
[Unit]
Description=App container
After=postgres.service
Requires=postgres.service
```
- `After=`: **실행 순서** 제어
- `Requires=`: **존재/가용성** 요구(없으면 실패)

### 5.2 DB 준비 대기(Health 기반)
DB 유닛이 끝났다고 DB가 준비된 것은 아니다. 앱 유닛에서 `ExecStartPre`로 대기:
```ini
ExecStartPre=/usr/bin/bash -lc 'for i in {1..30}; do nc -z 127.0.0.1 5432 && exit 0; sleep 1; done; exit 1'
```
혹은 DB 컨테이너에 **HEALTHCHECK**를 정의하고, 앱 유닛이 컨테이너 Health를 조회하는 방식도 가능(스크립트 필요).

---

## 6. 템플릿 유닛으로 반복 줄이기

여러 인스턴스를 돌릴 때 `@` 템플릿이 유용하다.

`/etc/systemd/system/app@.service`
```ini
[Unit]
Description=App container %i
After=docker.service
Requires=docker.service

[Service]
Type=exec
Restart=always
RestartSec=4s
Environment=IMAGE=myorg/app:1.0.0
ExecStartPre=/usr/bin/docker rm -f app-%i || true
ExecStart=/usr/bin/docker run --name app-%i -p %i:8080 ${IMAGE}
ExecStop=/usr/bin/docker stop -t 10 app-%i
ExecStopPost=/usr/bin/docker rm -f app-%i

[Install]
WantedBy=multi-user.target
```

인스턴스별 포트만 바꿔 실행:
```bash
sudo systemctl enable --now app@8081.service
sudo systemctl enable --now app@8082.service
```

---

## 7. Watchdog/StartLimit로 플래핑(무한 재시작) 제어

```ini
[Service]
Restart=always
RestartSec=2s
StartLimitIntervalSec=60
StartLimitBurst=5
```
- 60초에 5회 초과 재시작 시 **단기간 차단**. 운영자 알람 기준으로 좋다.

---

## 8. systemd 샌드박싱(실행 컨텍스트 하드닝)

컨테이너 속 **애플리케이션 보안**은 컨테이너 옵션으로 다루지만, Docker CLI 자체 실행 맥락도 다듬을 수 있다(특히 rootless/전용 계정 운용 시).

```ini
[Service]
User=svc-docker       # 전용 사용자
Group=docker          # docker 그룹 포함 필요 (rootless 제외)
NoNewPrivileges=true
PrivateTmp=true
ProtectSystem=full    # /usr, /boot 등 보호(클라이언트 관점)
ProtectHome=true
RestrictRealtime=true
LockPersonality=true
RestrictSUIDSGID=true
SystemCallFilter=@system-service
```

> 주의: 위 옵션은 **docker CLI 프로세스**에 적용되는 것이지 컨테이너 내부가 아니다. 컨테이너 하드닝은 `--cap-drop`, `--read-only`, seccomp/AppArmor/SELinux로 처리.

---

## 9. 롤링 업그레이드/롤백 절차 (단일 호스트)

### 9.1 버전 올리기
1) 유닛 파일의 이미지 태그 변경(`version=1.2.3 → 1.2.4`)
2) `daemon-reload` + 재시작
```bash
sudo systemctl daemon-reload
sudo systemctl restart web.service
```
3) 상태/로그 확인
```bash
sudo systemctl status web.service
sudo journalctl -u web.service -n 200 --no-pager
```

### 9.2 롤백
- 이전 태그로 되돌려 같은 절차 반복
- 또는 레지스트리에서 `:stable` 태그를 **원자적 전환** 후 `restart`(Pull 전제)

---

## 10. 로그 운영: journald + 회전/필터

- 유닛 로그:
```bash
journalctl -u web.service -f
```
- 기간/레벨 필터:
```bash
journalctl -u web.service --since "2025-11-01" -p warning
```
- journald 저장 크기(시스템 전역):
`/etc/systemd/journald.conf`
```
SystemMaxUse=2G
SystemMaxFileSize=200M
```
```bash
sudo systemctl restart systemd-journald
```

---

## 11. Docker 데몬과의 올바른 의존성

- **항상** `After=docker.service` + `Requires=docker.service` 지정
- 네트워크 준비 필요 시 `network-online.target` 사용 (`systemd-networkd-wait-online.service` 확보)
- Docker 데몬 자체 튜닝은 `/etc/docker/daemon.json`과 `docker.service` 드롭인으로 조정

---

## 12. Rootless Docker / 사용자 단위 유닛

Rootless 환경에선 **user** systemd를 사용한다.

예) 일반 사용자로:
```bash
systemctl --user enable --now myproject.service
systemctl --user status myproject.service
journalctl --user -u myproject.service -f
```
- 소켓: `$XDG_RUNTIME_DIR/docker.sock`
- `User=` 불필요(이미 사용자 세션)
- 부팅 시 자동 실행: `loginctl enable-linger <username>`

---

## 13. 정리·청소 자동화: Timer/Path 유닛

### 13.1 이미지/컨테이너 정리(주간 prune)
`/etc/systemd/system/docker-prune.service`
```ini
[Unit]
Description=Docker prune (safe)

[Service]
Type=oneshot
ExecStart=/usr/bin/docker system prune -f --volumes
```

`/etc/systemd/system/docker-prune.timer`
```ini
[Unit]
Description=Weekly docker prune

[Timer]
OnCalendar=Sun 03:00
Persistent=true

[Install]
WantedBy=timers.target
```
활성화:
```bash
sudo systemctl enable --now docker-prune.timer
```

### 13.2 설정 파일 변경 감지 후 자동 재시작(Path 유닛)
`/etc/systemd/system/web-reload.path`
```ini
[Unit]
Description=Restart web.service on config change

[Path]
PathChanged=/etc/myapp/web-config.yaml

[Install]
WantedBy=multi-user.target
```
`/etc/systemd/system/web-reload.service`
```ini
[Unit]
Description=Restart web on config change

[Service]
Type=oneshot
ExecStart=/usr/bin/systemctl restart web.service
```
활성화:
```bash
sudo systemctl enable --now web-reload.path
```

---

## 14. HealthCheck와 systemd 연동 (심화)

컨테이너가 unhealthy일 때 **강제 재시작**하려면 ExecStartPost로 간단한 감시 루프를 둘 수 있다(간단 구현 예).

`/usr/local/bin/watch-health.sh`
{% raw %}
```bash
#!/usr/bin/env bash
set -euo pipefail
NAME="$1"
INTERVAL="${2:-10}"

while sleep "$INTERVAL"; do
  STATUS="$(docker inspect --format='{{json .State.Health.Status}}' "$NAME" 2>/dev/null || echo null)"
  if [[ "$STATUS" == "\"unhealthy\"" ]]; then
    echo "container $NAME unhealthy -> restarting" >&2
    systemctl restart "${SERVICE_NAME:-$NAME}.service" || true
    exit 0
  fi
done
```
{% endraw %}
유닛에 추가:
```ini
ExecStartPost=/usr/bin/env SERVICE_NAME=web /usr/local/bin/watch-health.sh web 10
```
> 운영에선 외부 헬스 모니터(프로메테우스 알림/오토힐러) 권장.

---

## 15. 보안·격리 모범값(컨테이너 실행 옵션)

컨테이너 하드닝은 **유닛 내부의 docker run 라인**에서 통합한다.

체크리스트:
- `--user`(비루트), `--read-only`, `--tmpfs /tmp`, **쓰기 경로 명시 볼륨**
- `--cap-drop ALL` + 필요 최소 `--cap-add`
- `--security-opt no-new-privileges:true`
- **seccomp/AppArmor/SELinux**(배포판별로)
- **리소스 제한**: `--cpus`, `--memory`, `--pids-limit`, ulimit
- **로그 회전**: `--log-driver=json-file --log-opt max-size=10m --log-opt max-file=5`
- **민감정보**: env 대신 파일/외부 시크릿, `EnvironmentFile` 권한 제한

샘플:
```ini
ExecStart=/usr/bin/docker run --name secure-app \
  --user 10001:10001 \
  --read-only --tmpfs /tmp \
  --cap-drop=ALL --cap-add=NET_BIND_SERVICE \
  --security-opt no-new-privileges:true \
  --log-driver=json-file --log-opt max-size=10m --log-opt max-file=5 \
  --cpus=1.0 --memory=512m --pids-limit=256 \
  -p 8443:8443 \
  myorg/secure-app:3.4.1
```

---

## 16. 트러블슈팅 베스트 프랙티스

| 증상 | 확인/조치 |
|---|---|
| 유닛이 바로 실패 | `systemctl status`, `journalctl -u`, `ExecStartPre` 스크립트 권한/경로 |
| 컨테이너는 있는데 유닛은 실패 | `--rm` 사용 여부, `ExecStopPost=rm`로 통일 |
| 네트워크 지연 | `network-online.target`/대기 스크립트 도입 |
| 재시작 폭주 | `StartLimitIntervalSec/StartLimitBurst` 조정, 리소스/포트 충돌 점검 |
| 권한 에러 | rootless 여부, `docker` 그룹, `User=`/폴더 권한 |
| 로그 폭증 | journald/컨테이너 로그 회전 동시 점검 |

---

## 17. 종합 샘플 — API + DB + Nginx 리버스 프록시

```
/etc/systemd/system/
  db.service
  api.service
  proxy.service
```

`db.service` (PostgreSQL 컨테이너)
```ini
[Unit]
Description=Postgres container
After=docker.service
Requires=docker.service

[Service]
Restart=always
RestartSec=3s
ExecStartPre=/usr/bin/docker rm -f pg || true
ExecStart=/usr/bin/docker run --name pg \
  -e POSTGRES_PASSWORD=secret \
  -v pgdata:/var/lib/postgresql/data \
  -p 5432:5432 \
  postgres:16
ExecStop=/usr/bin/docker stop -t 15 pg
ExecStopPost=/usr/bin/docker rm -f pg

[Install]
WantedBy=multi-user.target
```

`api.service`
```ini
[Unit]
Description=API container
After=db.service
Requires=db.service

[Service]
Restart=always
RestartSec=3s
EnvironmentFile=/etc/myapp/api.env
ExecStartPre=/usr/bin/docker rm -f api || true
ExecStartPre=/usr/bin/bash -lc 'for i in {1..30}; do nc -z 127.0.0.1 5432 && exit 0; sleep 1; done; exit 1'
ExecStart=/usr/bin/docker run --name api \
  -e DB_HOST=127.0.0.1 -e DB_USER=${DB_USER} -e DB_PASS=${DB_PASS} \
  --read-only --tmpfs /tmp --cap-drop=ALL \
  --log-driver=json-file --log-opt max-size=10m --log-opt max-file=5 \
  -p 8080:8080 \
  myorg/api:2.0.1
ExecStop=/usr/bin/docker stop -t 10 api
ExecStopPost=/usr/bin/docker rm -f api

[Install]
WantedBy=multi-user.target
```

`proxy.service` (Nginx가 API로 프록시)
```ini
[Unit]
Description=Reverse proxy container
After=api.service
Requires=api.service

[Service]
Restart=always
RestartSec=3s
ExecStartPre=/usr/bin/docker rm -f proxy || true
ExecStart=/usr/bin/docker run --name proxy \
  -p 80:80 \
  -v /etc/nginx/conf.d:/etc/nginx/conf.d:ro \
  nginx:1.25
ExecStop=/usr/bin/docker stop -t 10 proxy
ExecStopPost=/usr/bin/docker rm -f proxy

[Install]
WantedBy=multi-user.target
```

활성화:
```bash
sudo systemctl daemon-reload
sudo systemctl enable --now db.service api.service proxy.service
```

---

## 18. 요약 체크리스트

- 유닛 위치: `/etc/systemd/system/*.service` → 변경 시 `daemon-reload`
- 항상 `After=docker.service` + `Requires=docker.service`
- 배포 루틴: `pull → rm(old) → run(new)` 를 유닛 내 Pre/Post로 표준화
- 로그: journald + 컨테이너 회전 옵션 동시 관리
- 보안: `--user`, `--read-only`, `--cap-drop ALL`, `no-new-privileges`
- 리소스: `--cpus`, `--memory`, `--pids-limit`, `LimitNOFILE`
- 의존성: DB 헬스/포트 대기 루틴으로 **실제 준비 여부** 보장
- Compose 사용 시 `WorkingDirectory` 필수, `docker compose up` 포그라운드
- 템플릿 유닛으로 반복 최소화, Timer/Path로 운영 자동화

---

## 참고
- systemd service: https://www.freedesktop.org/software/systemd/man/systemd.service.html
- journalctl: https://www.freedesktop.org/software/systemd/man/journalctl.html
- Docker 자동 시작: https://docs.docker.com/config/containers/start-containers-automatically/

---
```bash
# 운영에 유용한 명령 모음
sudo systemctl daemon-reload
sudo systemctl enable --now <unit>.service
sudo systemctl restart <unit>.service
sudo systemctl status <unit>.service
sudo journalctl -u <unit>.service -f
```
