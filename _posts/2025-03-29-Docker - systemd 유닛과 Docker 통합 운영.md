---
layout: post
title: Docker - systemd 유닛과 Docker 통합 운영
date: 2025-03-29 20:20:23 +0900
category: Docker
---
# systemd와 Docker 통합 운영: 완벽 가이드

## systemd와 Docker 통합의 필요성

Docker 컨테이너를 운영 환경에서 안정적으로 관리하기 위해서는 systemd와의 통합이 필수적입니다. Docker의 기본 재시작 정책만으로는 부족한 운영 요구사항을 systemd가 보완해줍니다:

| 운영 요구사항 | Docker 단독 사용의 한계 | systemd 통합 시 이점 |
|---|---|---|
| 시스템 부팅 시 자동 시작 | `--restart` 플래그에 의존적 | 부팅 타겟과 의존성 기반의 세밀한 제어(`After=`, `Requires=`) |
| 장애 복구 및 재시작 제어 | 컨테이너 재시작 정책만으로 제한 | `StartLimitBurst`, `RestartSec`, `Watchdog` 등 다양한 제어 옵션 |
| 서비스 간 종속성 관리 | 수동 스크립트로 복잡한 관리 필요 | 다른 systemd 유닛과의 의존관계 정의(`Requires=`, `BindsTo=`, `PartOf=`) |
| 통합 로깅 및 감사 | Docker 로깅 드라이버에 의존 | journald 통합을 통한 표준화된 로그 관리(`journalctl -u`) |
| 보안 컨텍스트 | 컨테이너 격리에 주로 집중 | systemd 샌드박싱을 통한 CLI 실행 환경 보안 강화 |
| 배포 파이프라인 | 사용자 정의 스크립트 필요 | pull→run 순서화, 이미지 롤백, 그레이스풀 종료 표준화 |

---

## 기본 개념: 단일 컨테이너를 systemd 유닛으로 실행하기

### Nginx 컨테이너의 기본 systemd 서비스 예제

`/etc/systemd/system/nginx-container.service` 파일을 다음과 같이 생성합니다:

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

**중요한 점**: `--rm` 플래그 대신 `ExecStopPost`에서 컨테이너를 제거하는 방식을 사용합니다. 이렇게 하면 서비스 재시작 시 컨테이너가 이미 존재하지 않아 발생하는 실패를 방지할 수 있습니다. 또한 HealthCheck를 추가하여 컨테이너 내부 상태를 운영 로그에서 모니터링할 수 있습니다.

활성화 및 실행:
```bash
sudo systemctl daemon-reload
sudo systemctl enable --now nginx-container.service
sudo systemctl status nginx-container.service
sudo journalctl -u nginx-container.service -f
```

---

## 운영 환경을 위한 고급 패턴

### 사전 이미지 풀링과 명시적 옵션 설정

단일 서비스 파일 내에서 이미지 풀링부터 실행까지의 전체 절차를 정의하면 업그레이드와 재배포가 간소화됩니다:

`/etc/systemd/system/web.service`:
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

# 최신 이미지 풀링 (또는 특정 태그 유지)
ExecStartPre=/usr/bin/docker pull my-registry.example.com/myorg/web:1.2.3

# 기존 컨테이너 정리 (존재할 경우에만)
ExecStartPre=/usr/bin/docker rm -f web || true

# 컨테이너 실행
ExecStart=/usr/bin/docker run --name web \
  --label app=web --label version=1.2.3 \
  --log-driver=json-file --log-opt max-size=10m --log-opt max-file=5 \
  --cpus=1.0 --memory=512m --pids-limit=256 \
  --read-only --tmpfs /tmp \
  --cap-drop=ALL --cap-add=NET_BIND_SERVICE \
  -p 8080:8080 \
  my-registry.example.com/myorg/web:1.2.3

# 그레이스풀 종료 및 후속 처리
ExecStop=/usr/bin/docker stop -t 15 web
ExecStopPost=/usr/bin/docker rm -f web

# systemd 자체 리소스 제한 (선택 사항)
LimitNOFILE=65535
TasksMax=4096

[Install]
WantedBy=multi-user.target
```

**이 패턴의 장점**:
- **재현성 보장**: 고정된 이미지 태그를 사용하여 동일한 버전의 컨테이너가 항상 실행됩니다. 롤백 시에는 태그만 변경하여 재시작하면 됩니다.
- **디스크 관리**: Docker 로그 회전 옵션을 명시적으로 설정하여 디스크 공간 폭주를 방지합니다.
- **명확한 운영 기준**: 컨테이너 수준과 systemd 수준 모두에서 리소스 제한을 설정하여 운영 기준이 명확해집니다.

---

## 환경 변수와 비밀 정보 관리

### systemd EnvironmentFile과 Docker 환경 변수 통합

`/etc/systemd/system/api.service`:
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

환경 변수 파일 `/etc/myapp/api.env`:
```
DB_HOST=postgresql.internal
DB_USER=apiuser
DB_PASS=change_me
```

**보안 권장사항**: 민감한 정보가 포함된 환경 파일은 `0640` 권한(`root:root`)으로 설정해야 합니다. 프로덕션 환경에서는 외부 비밀 저장소나 Vault와 같은 솔루션을 사용하는 것이 더 안전합니다. Docker Swarm이나 Kubernetes를 사용하는 경우 해당 플랫폼의 Secrets 리소스를 활용하는 것이 좋습니다.

---

## Docker Compose와 systemd 통합

Docker Compose v2의 단일 바이너리 방식을 사용하여 Compose 스택을 systemd로 관리할 수 있습니다:

`/etc/systemd/system/myproject.service`:
```ini
[Unit]
Description=Compose stack: myproject
After=network-online.target docker.service
Wants=network-online.target
Requires=docker.service

[Service]
Type=exec
WorkingDirectory=/srv/myproject

# 파일 권한 설정 (선택 사항)
UMask=0027

# 환경 파일 로드 (선택 사항)
EnvironmentFile=/srv/myproject/.env

# Compose를 포그라운드 모드로 실행하여 systemd에 연결
ExecStart=/usr/bin/docker compose --ansi=never up --no-color

# 그레이스풀 종료
ExecStop=/usr/bin/docker compose down

Restart=always
RestartSec=3s

[Install]
WantedBy=multi-user.target
```

**주의사항**:
- `WorkingDirectory`를 반드시 설정해야 Compose 파일의 상대 경로가 올바르게 해석됩니다.
- 서비스 로그는 `journalctl -u myproject.service`로 확인할 수 있으며, 개별 컨테이너 로그는 `docker logs` 명령으로 병행 확인할 수 있습니다.
- HealthCheck는 Compose 파일 내 각 서비스에 정의하고, 서비스 간 의존성은 `depends_on`과 healthcheck 조건을 조합하여 관리합니다.

---

## 서비스 간 종속성 및 순서 제어

### 유닛 간 의존성 설정

```ini
[Unit]
Description=App container
After=postgres.service
Requires=postgres.service
```
- `After=`는 실행 순서를 제어합니다.
- `Requires=`는 다른 유닛의 존재와 가용성을 요구합니다(해당 유닛이 없거나 실패하면 이 유닛도 실패합니다).

### 데이터베이스 준비 상태 확인

데이터베이스 컨테이너가 실행되었다고 해서 데이터베이스 서비스가 준비된 것은 아닙니다. 애플리케이션 서비스에서 `ExecStartPre`를 사용하여 데이터베이스가 실제로 연결 가능한 상태인지 확인할 수 있습니다:

```ini
ExecStartPre=/usr/bin/bash -lc 'for i in {1..30}; do nc -z 127.0.0.1 5432 && exit 0; sleep 1; done; exit 1'
```

또는 데이터베이스 컨테이너에 HealthCheck를 정의하고, 애플리케이션 서비스가 해당 HealthCheck 상태를 조회하는 방식도 가능합니다(추가 스크립트 필요).

---

## 템플릿 유닛을 통한 반복 작업 최소화

여러 인스턴스를 실행해야 할 때 systemd의 템플릿 유닛 기능을 활용하면 효율적으로 관리할 수 있습니다:

`/etc/systemd/system/app@.service`:
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

다양한 포트로 인스턴스 실행:
```bash
sudo systemctl enable --now app@8081.service
sudo systemctl enable --now app@8082.service
```

---

## 재시작 제어 및 제한 설정

과도한 재시작을 방지하기 위해 다음과 같이 제한을 설정할 수 있습니다:

```ini
[Service]
Restart=always
RestartSec=2s
StartLimitIntervalSec=60
StartLimitBurst=5
```

이 설정은 60초 동안 5번을 초과하여 재시작이 발생하면 일시적으로 서비스 시작을 차단합니다. 이는 운영자에게 알람을 트리거하기 좋은 기준점이 됩니다.

---

## systemd 샌드박싱을 통한 보안 강화

컨테이너 내부 애플리케이션 보안은 Docker 옵션으로 관리하지만, Docker CLI 자체의 실행 환경도 systemd 샌드박싱으로 보호할 수 있습니다(특히 rootless 모드나 전용 계정을 사용할 때):

```ini
[Service]
User=svc-docker       # 전용 사용자 지정
Group=docker          # docker 그룹 멤버십 필요 (rootless 제외)
NoNewPrivileges=true
PrivateTmp=true
ProtectSystem=full    # /usr, /boot 등 시스템 디렉터리 보호 (클라이언트 관점)
ProtectHome=true
RestrictRealtime=true
LockPersonality=true
RestrictSUIDSGID=true
SystemCallFilter=@system-service
```

**주의**: 이러한 옵션은 Docker CLI 프로세스 자체에 적용되는 것이지 컨테이너 내부에 적용되는 것이 아닙니다. 컨테이너 내부 보안은 `--cap-drop`, `--read-only`, seccomp, AppArmor, SELinux와 같은 Docker 옵션으로 처리해야 합니다.

---

## 롤링 업데이트 및 롤백 절차 (단일 호스트 환경)

### 버전 업그레이드 절차

1. 서비스 유닛 파일의 이미지 태그를 변경합니다(`version=1.2.3 → 1.2.4`)
2. systemd 데몬 재로드 및 서비스 재시작:
```bash
sudo systemctl daemon-reload
sudo systemctl restart web.service
```
3. 상태 및 로그 확인:
```bash
sudo systemctl status web.service
sudo journalctl -u web.service -n 200 --no-pager
```

### 롤백 절차

- 이전 태그로 되돌린 후 동일한 절차를 반복합니다.
- 또는 레지스트리에서 `:stable` 태그를 원자적으로 전환한 후 서비스를 재시작합니다(이미지 풀링이 전제되어야 함).

---

## 로그 관리: journald 통합

### 기본 로그 확인
```bash
# 서비스 로그 실시간 모니터링
journalctl -u web.service -f

# 특정 기간 및 로그 레벨 필터링
journalctl -u web.service --since "2025-11-01" -p warning
```

### journald 저장소 크기 구성
`/etc/systemd/journald.conf` 파일을 수정합니다:
```
SystemMaxUse=2G
SystemMaxFileSize=200M
```
변경 사항 적용:
```bash
sudo systemctl restart systemd-journald
```

---

## Docker 데몬과의 올바른 의존성 설정

- **필수**: 항상 `After=docker.service`와 `Requires=docker.service`를 지정합니다.
- 네트워크 준비가 필요한 경우 `network-online.target`을 사용합니다(`systemd-networkd-wait-online.service`가 필요할 수 있음).
- Docker 데몬 자체 튜닝은 `/etc/docker/daemon.json`과 `docker.service` 드롭인 파일을 통해 조정합니다.

---

## Rootless Docker 환경에서의 systemd 통합

Rootless Docker 환경에서는 사용자 단위 systemd를 사용합니다:

```bash
# 일반 사용자로 서비스 관리
systemctl --user enable --now myproject.service
systemctl --user status myproject.service
journalctl --user -u myproject.service -f
```

**특징**:
- 소켓 경로: `$XDG_RUNTIME_DIR/docker.sock`
- `User=` 설정 불필요 (이미 사용자 세션에서 실행)
- 부팅 시 자동 실행 활성화: `loginctl enable-linger <username>`

---

## 정기적인 정리 및 모니터링 자동화

### 주간 Docker 시스템 정리

`/etc/systemd/system/docker-prune.service`:
```ini
[Unit]
Description=Docker prune (safe)

[Service]
Type=oneshot
ExecStart=/usr/bin/docker system prune -f --volumes
```

`/etc/systemd/system/docker-prune.timer`:
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

### 설정 파일 변경 감지 및 자동 재시작

`/etc/systemd/system/web-reload.path`:
```ini
[Unit]
Description=Restart web.service on config change

[Path]
PathChanged=/etc/myapp/web-config.yaml

[Install]
WantedBy=multi-user.target
```

`/etc/systemd/system/web-reload.service`:
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

## HealthCheck와 systemd 연동 (고급)

컨테이너가 unhealthy 상태일 때 강제로 재시작하려면 `ExecStartPost`에 간단한 모니터링 스크립트를 추가할 수 있습니다:

`/usr/local/bin/watch-health.sh`:
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

서비스 유닛에 추가:
```ini
ExecStartPost=/usr/bin/env SERVICE_NAME=web /usr/local/bin/watch-health.sh web 10
```

**운영 권장사항**: 프로덕션 환경에서는 외부 헬스 모니터링 시스템(프로메테우스 알림 관리자나 오토힐러 등)을 사용하는 것이 더 안정적입니다.

---

## 컨테이너 보안 강화 모범 사례

컨테이너 보안 설정은 서비스 유닛 내의 `docker run` 명령어에서 통합적으로 관리합니다:

**보안 체크리스트**:
- `--user`: 비루트 사용자로 실행
- `--read-only` + `--tmpfs /tmp`: 읽기 전용 파일 시스템에 임시 파일 시스템 추가
- `--cap-drop ALL` + 필요한 최소한의 `--cap-add`: 불필요한 Linux Capabilities 제거
- `--security-opt no-new-privileges:true`: 권한 상승 방지
- seccomp/AppArmor/SELinux 프로필 적용: 배포판에 맞는 보안 모듈 사용
- 리소스 제한: `--cpus`, `--memory`, `--pids-limit`, ulimit 설정
- 로그 회전: `--log-driver=json-file --log-opt max-size=10m --log-opt max-file=5`
- 비밀 정보 관리: 환경 변수 대신 파일 또는 외부 시크릿 저장소 사용, `EnvironmentFile` 권한 제한

**보안 강화된 실행 예시**:
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

## 문제 해결 가이드

| 증상 | 확인 및 조치 항목 |
|---|---|
| 서비스 유닛이 즉시 실패 | `systemctl status`, `journalctl -u`, `ExecStartPre` 스크립트의 권한 및 경로 확인 |
| 컨테이너는 실행 중이지만 서비스는 실패 상태 | `--rm` 옵션 사용 여부 확인, `ExecStopPost`에서 명시적으로 컨테이너 제거 |
| 네트워크 연결 지연 | `network-online.target` 및 네트워크 대기 스크립트 도입 |
| 재시작 루프 발생 | `StartLimitIntervalSec`/`StartLimitBurst` 조정, 리소스 및 포트 충돌 점검 |
| 권한 오류 | rootless 모드 여부, `docker` 그룹 멤버십, `User=` 설정 및 폴더 권한 확인 |
| 로그 저장소 급증 | journald와 컨테이너 로그 회전 설정 동시 점검 |

---

## 종합 실전 예제: API + 데이터베이스 + 리버스 프록시 아키텍처

다음은 PostgreSQL 데이터베이스, API 서버, Nginx 리버스 프록시로 구성된 전체 스택의 systemd 통합 예제입니다:

```
/etc/systemd/system/
├── db.service      # PostgreSQL 컨테이너
├── api.service     # API 서버 컨테이너
└── proxy.service   # Nginx 리버스 프록시 컨테이너
```

### 데이터베이스 서비스 (`db.service`)
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

### API 서비스 (`api.service`)
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

### 리버스 프록시 서비스 (`proxy.service`)
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

**전체 스택 활성화**:
```bash
sudo systemctl daemon-reload
sudo systemctl enable --now db.service api.service proxy.service
```

---

## 결론: systemd와 Docker 통합 운영의 핵심 원칙

systemd와 Docker의 통합은 단순한 기술적 결합을 넘어 운영의 안정성, 보안성, 유지보수성을 크게 향상시키는 전략적 접근법입니다. 이 가이드에서 제시한 원칙과 모범 사례를 바탕으로 효과적인 운영 체계를 구축할 수 있습니다:

1. **일관성 있는 배포 패턴 확립**: `pull → 제거 → run`의 표준화된 절차를 모든 서비스에 적용하여 배포 프로세스를 일관되게 유지합니다. 이미지 태그 고정을 통해 재현성을 보장하고, 롤백 시에는 태그 변경만으로 이전 버전으로 복귀할 수 있어야 합니다.

2. **다층적 보안 체계 구현**: 컨테이너 수준(`--user`, `--read-only`, `--cap-drop`)과 systemd 수준(`NoNewPrivileges`, `PrivateTmp`, `ProtectSystem`)에서의 보안 설정을 조합하여 다층 방어 체계를 구축합니다. 각 수준에서의 보안 책임을 명확히 이해하고 적용해야 합니다.

3. **종속성과 상태 관리의 명확한 분리**: 서비스 실행 순서(`After=`)와 실제 준비 상태 확인(HealthCheck, 포트 검사)을 구분하여 관리합니다. 데이터베이스가 컨테이너로 실행되었다고 해서 연결 가능한 상태인 것은 아니므로, 실제 서비스 준비 여부를 확인하는 메커니즘을 마련해야 합니다.

4. **통합된 관측성 체계 구축**: journald와 Docker 로그 드라이버를 조화롭게 구성하여 로그 관리 체계를 단일화합니다. 로그 회전 정책을 명시적으로 설정하고, 로그 저장소 크기를 적절히 제한하여 디스크 공간 문제를 방지합니다.

5. **자동화와 자가 치유 능력 강화**: Timer 유닛을 활용한 정기적인 정리 작업, Path 유닛을 통한 설정 변경 감지, HealthCheck 기반의 자동 복구 메커니즘을 도입하여 운영 부담을 줄이고 시스템의 회복 탄력성을 높입니다.

6. **운영 복잡도 관리**: 템플릿 유닛을 활용하여 유사한 서비스의 반복 구성을 최소화하고, Docker Compose를 systemd와 통합하여 복잡한 멀티 컨테이너 애플리케이션을 체계적으로 관리합니다.

7. **단계적 개선 접근**: 처음부터 모든 것을 완벽하게 구현하기보다 핵심 서비스부터 시작하여 점진적으로 개선해 나갑니다. 각 배포 사이클에서 하나의 운영 개선 사항(로깅 향상, 보안 강화, 모니터링 추가 등)을 도입하는 방식을 채택합니다.

systemd와 Docker의 통합은 두 기술의 강점을 결합하여 단일 호스트 환경에서도 프로덕션 수준의 컨테이너 운영을 가능하게 합니다. 이 가이드의 원칙과 예제를 시작점으로 삼아 팀의 특정 요구사항과 환경에 맞게 조정해 나간다면, 더욱 견고하고 관리하기 쉬운 컨테이너 인프라를 구축할 수 있을 것입니다.

기술은 계속 발전하지만, 일관성, 보안, 관측성, 자동화라는 기본 원칙은 변하지 않습니다. 이러한 원칙에 충실하면서도 유연하게 도구와 환경을 조정해 나가는 것이 지속 가능한 운영 체계의 핵심입니다.

---

## 참고 자료

- systemd 서비스 유닛 문서: https://www.freedesktop.org/software/systemd/man/systemd.service.html
- journalctl 사용 가이드: https://www.freedesktop.org/software/systemd/man/journalctl.html
- Docker 자동 시작 구성: https://docs.docker.com/config/containers/start-containers-automatically/

---

## 핵심 운영 명령어 요약

```bash
# 서비스 유닛 관리
sudo systemctl daemon-reload                     # 유닛 파일 변경 후 적용
sudo systemctl enable --now <unit>.service       # 활성화 및 즉시 시작
sudo systemctl restart <unit>.service            # 서비스 재시작
sudo systemctl status <unit>.service             # 서비스 상태 확인

# 로그 관리
sudo journalctl -u <unit>.service -f             # 실시간 로그 모니터링
sudo journalctl -u <unit>.service --since "2 hours ago"  # 시간별 로그 필터링
sudo journalctl -u <unit>.service -n 100         # 최근 100개 로그 확인
```