---
layout: post
title: Docker - Docker 컨테이너
date: 2025-01-11 19:20:23 +0900
category: Docker
---
# Docker 컨테이너란? 상태와 수명주기

## 1. 컨테이너란?

Docker 컨테이너는 **이미지로부터 생성된 실행 인스턴스**입니다.
이미지의 읽기 전용 레이어 위에, 컨테이너별 **쓰기 가능한 레이어**가 얹혀 실행됩니다(OverlayFS, COW).

### 핵심 특징(복습)
- **이미지 기반**: 이미지 = 불변 템플릿, 컨테이너 = 실행 상태를 갖는 동적 객체
- **격리**: 파일시스템, 네트워크, PID, UTS, IPC 등 **네임스페이스**로 분리
- **경량**: **호스트 커널 공유**, VM 대비 빠른 생성/자원 효율
- **일시성**: 컨테이너 레이어는 휘발. **볼륨**/외부 스토리지를 사용하면 데이터 영속 가능

---

## 2. 구조(파일시스템 레이어)

```
+-----------------------------------+
| Writable Container Layer (upper)  |  ← 컨테이너 전용 쓰기 레이어
+-----------------------------------+
| Read-only Image Layers (lower)    |  ← FROM ~ RUN ~ COPY로 생긴 이미지 레이어
+-----------------------------------+
| Host Kernel (shared)              |
+-----------------------------------+
```

- **COW(Copy-on-Write)**: 읽기 전용 레이어의 파일을 수정하려 하면 upper로 복사 후 수정
- 여러 컨테이너/이미지가 **공통 레이어**를 공유 → 디스크 절약·빠른 배포

---

## 3. 컨테이너 상태(State)와 전이

| 상태 | 설명 | 주요 명령 |
|---|---|---|
| `created` | 이미지로부터 생성했으나 아직 시작 전 | `docker create` |
| `running` | 애플리케이션 프로세스 실행 중 | `docker start`, `docker run` |
| `paused` | cgroup freezer 등으로 스케줄링 정지 | `docker pause` / `unpause` |
| `exited` | 프로세스 종료(Exit 0/비0 포함) | `docker stop` / `docker kill` 결과 |
| `dead` | 내부 오류 등 비정상 잔존 상태(희귀) | 엔진/저장소 이슈 시 |

상태 확인:
```bash
docker ps -a
docker inspect <컨테이너> | jq '.[0].State'
```

### 상태 흐름(요약)

```
이미지
  ↓
create  →  [created]
  ↓
start   →  [running] → stop(SIGTERM) → [exited] → rm → 삭제
                 └─ kill(SIGKILL) ───┘
pause → [paused] → unpause → [running]
```

---

## 4. 수명주기(Lifecycle) 명령 실습

```bash
# 1. 실행 없이 생성만
docker create --name demo ubuntu sleep 600     # [created]

# 2. 시작
docker start demo                               # [running]

# 3. 상태/로그/세부 정보
docker ps -a
docker logs demo                                # ENTRYPOINT/CMD가 stdout을 낼 때 유효
docker inspect demo | jq '.[0].State, .[0].Mounts'

# 4. 일시 중지/재개
docker pause demo
docker unpause demo

# 5. 정상 정지(TERM → grace 후 KILL)
docker stop demo     # 기본 10초 타임아웃
# 5-1) 즉시 강제 종료
docker kill demo

# 6. 삭제(정지 상태여야 함)
docker rm demo
```

> `docker run` = `create` + `start` + (필요 시 `attach`) 의 축약입니다.

---

## 5. 신호와 종료: stop vs kill, 종료 코드

- `docker stop`: **SIGTERM** 전송 → **grace period**(기본 10s) → 미종료 시 **SIGKILL**
- `docker kill`: 즉시 **SIGKILL**
- 종료 코드: 0(정상), 그 외(오류). 재시작 정책 판단, CI/CD에서 중요

타임아웃 조정:
```bash
docker stop -t 30 web    # 30초까지 정상 종료 대기
```

특정 신호 전송:
```bash
docker kill --signal=HUP web
```

> 애플리케이션이 **SIGTERM 처리**(정리/플러시)하도록 구현되어 있어야 데이터 일관성에 유리.

---

## 6. `run` 옵션과 실행 습관(보안/운영)

```bash
# 대화형/디버깅
docker run -it --name u1 ubuntu bash

# 서버형 프로세스 + 포트 + 이름
docker run -d --name web -p 8080:80 nginx:alpine

# 일회성 잡(종료 시 컨테이너 자동 삭제)
docker run --rm alpine sh -c 'date && echo done'

# 최소 권한/자원 제한/읽기 전용 루트FS
docker run --rm \
  --read-only \
  --tmpfs /tmp --tmpfs /run \
  --cap-drop ALL --security-opt no-new-privileges \
  --user 65532:65532 \
  --pids-limit 128 --cpus 1 --memory 256m \
  nginx:alpine
```

핵심 옵션 요약:
- `-d`, `--name`, `-p H:C`, `-v SRC:DST[:옵션]`, `--env/-e`
- 보안: `--user`, `--read-only`, `--cap-drop ALL`, `--security-opt no-new-privileges`
- 자원: `--cpus`, `--memory`, `--pids-limit`

---

## 7. 재시작 정책(Availability 향상)

애플리케이션 장애나 도커 데몬 재시작 시 **자동 재기동** 설정:

| 정책 | 동작 |
|---|---|
| `no` | 기본값. 자동 재시작 없음 |
| `on-failure[:N]` | 비0 종료 시 N회까지 재시도 |
| `always` | 항상 재시작(수동 stop 제외) |
| `unless-stopped` | 사용자가 멈추지 않는 한 재시작 |

```bash
docker run -d --restart=unless-stopped --name api -p 5000:5000 myapp:prod
```

간단 가용성 근사:
$$
\text{Availability} \approx \frac{\text{MTBF}}{\text{MTBF} + \text{MTTR}}
$$
- MTTR을 재시작 정책으로 줄이면(장애 복구 시간↓) 가용성↑

---

## 8. 컨테이너 내부 진입/디버깅

```bash
# 실행 중 컨테이너에 셸
docker exec -it web sh
docker exec -it web bash      # bash가 있는 이미지

# attach vs exec
docker attach web             # PID 1 표준입출력에 연결(주의)
```

> 일반적으로 **`exec`가 안전**(별도 셸 프로세스 생성), `attach`는 PID 1에 직접 붙어 제어가 꼬일 수 있음.

---

## 9. 로그·이벤트·상세 정보

```bash
# 로그(최근 100줄, 팔로우)
docker logs -f --tail=100 web

# 컨테이너 이벤트(생성/시작/정지 등)
docker events --filter 'container=web'

# 자세한 상태/네트워크/볼륨/환경
docker inspect web | jq '.[0] | {Name: .Name, State: .State, Mounts: .Mounts, NetworkSettings: .NetworkSettings.Ports}'
```

리소스 관찰:
```bash
docker stats             # CPU/메모리/네트워크/블록 I/O
docker top web           # 컨테이너 내부 프로세스 목록
```

---

## 10. 네트워킹(요약 실습)

컨테이너 간 통신을 위해 사용자 정의 브릿지를 활용하면 **이름 기반 DNS**가 가능합니다.

```bash
docker network create appnet

docker run -d --name api --network appnet -p 8081:8080 ghcr.io/acme/api:1.0
docker run --rm --network appnet curlimages/curl http://api:8080/health

docker network inspect appnet | jq '.[0].Containers'
docker network rm appnet     # 사용 중이면 삭제 불가
```

---

## 11. 데이터 영속화(컨테이너 레이어 vs 볼륨)

컨테이너 레이어는 **휘발**. 데이터는 볼륨/바인드 마운트 사용:

```bash
# 명명된 볼륨
docker volume create pgdata
docker run -d --name pg \
  -e POSTGRES_PASSWORD=secret \
  -v pgdata:/var/lib/postgresql/data \
  postgres:16-alpine

# 바인드 마운트(호스트 디렉터리)
mkdir -p ~/site && echo "hi" > ~/site/index.html
docker run -d --name web -p 8080:80 \
  -v ~/site:/usr/share/nginx/html:ro \
  nginx:alpine
```

> **상태 데이터(DB 등)** 는 반드시 볼륨으로 분리.

---

## 12. 컨테이너 수명주기 “확장” 시나리오

### 12.1 롤링 업데이트(수동)
```bash
# 새 이미지 빌드/풀
docker pull ghcr.io/acme/api:2.0

# 기존 컨테이너와 포트 충돌 없게 새 이름으로 띄움
docker run -d --name api-v2 --network appnet -p 8082:8080 ghcr.io/acme/api:2.0

# 헬스 확인 뒤 트래픽 전환(프록시/라우팅)
# 전환 완료 후 구버전 제거
docker rm -f api
docker rename api-v2 api
```

### 12.2 헬스체크를 이용한 상태 기반 제어
```Dockerfile
# Dockerfile 예: /health 응답을 헬시로 판단
HEALTHCHECK --interval=30s --timeout=3s --retries=3 \
  CMD curl -fsS http://localhost:8080/health || exit 1
```
```bash
docker inspect api | jq '.[0].State.Health'
```

---

## 13. 자원 제한과 QoS(cgroups)

```bash
docker run -d --name svc \
  --cpus 1.5 \
  --memory 512m \
  --pids-limit 256 \
  mysvc:latest
```

- **OOM(메모리 초과)** 시 커널 OOM Killer가 프로세스 종료 → 로그/알람 확인
- `docker stats` 로 지속 관찰, 병목 구간은 애플리케이션 튜닝 권장

---

## 14. 컨테이너와 PID 1(Init) 이슈

컨테이너의 첫 프로세스(PID 1)는 **신호 처리/좀비 reaping** 책임이 있습니다.
일부 런타임/언어는 PID 1에서 **신호 전파가 누락**될 수 있으므로, **init 래퍼** 사용을 권장:

```bash
docker run -d --init --name app myapp:latest
```

또는 ENTRYPOINT를 올바른 **exec form**으로 지정:

```Dockerfile
ENTRYPOINT ["gunicorn"]
CMD ["-w","2","-b","0.0.0.0:8080","app:app"]
```

---

## 15. 컨테이너 복구·정리 루틴

```bash
# 일괄 상태 보기
docker ps -a
docker system df

# 정지/삭제
docker stop <name> && docker rm <name>

# 미사용 자원(주의)
docker system prune -f
docker image prune -a -f
docker volume prune -f
```

> 운영 서버에서는 `prune -a` 사용 시 **필요 이미지 제거 위험**. CI 에이전트/개발 머신에서 선호.

---

## 16. 트러블슈팅 표(원인→진단→해결)

| 증상/오류 | 가능 원인 | 진단 | 해결 |
|---|---|---|---|
| 컨테이너 즉시 종료 | CMD가 끝남/오류 | `docker logs`, `docker inspect .State.ExitCode` | 장기 실행 커맨드로 교체, 오류 수정 |
| stop이 오래 걸림 | SIGTERM 처리 미구현 | `docker stop -t`, 애플리케이션 로그 | SIGTERM 핸들러 구현, 타임아웃 조정 |
| 포트 접근 불가 | `-p` 누락/방화벽 | `docker ps`, `docker port`, `curl` | `-p` 명시, 방화벽/NAT 규칙 확인 |
| 파일 권한 오류 | 비root + 읽기전용 FS | `ls -al`, 마운트 옵션 | `--tmpfs` 제공, `--user` UID/GID 조정 |
| OOM 재시작 | 메모리 한도 초과 | `docker stats`, 컨테이너 dmesg | 메모리 상향/코드 튜닝 |
| DNS 안 됨 | 사용자 네트워크/DNS 설정 | `docker exec ... cat /etc/resolv.conf` | 엔진/네트워크 설정 조정 |
| 느린 I/O(WSL2) | 호스트↔WSL 경계 | 경로 확인 | **WSL2 내부 경로**에서 빌드/마운트 |

---

## 17. 실습: 상태·수명주기 종합 예제

```bash
# 1. 네트워크 준비
docker network create appnet

# 2. API 컨테이너 실행(백그라운드)
docker run -d --name api --restart=unless-stopped \
  --network appnet -p 8080:8080 ghcr.io/acme/api:1.0

# 3. 상태/로그/리소스
docker ps
docker logs -f --tail=50 api
docker stats --no-stream

# 4. 내부 확인
docker exec -it api sh -lc 'uname -a; ps -ef | wc -l; ss -ltn'

# 5. 정상 정지 vs 강제 종료
docker stop -t 20 api      # SIGTERM → 20s 유예
docker rm api              # 정지 후 삭제

# 6. 청소(선택)
docker network rm appnet
```

---

## 18. 컨테이너 vs 이미지(요약 복습 표)

| 항목 | 이미지(Image) | 컨테이너(Container) |
|---|---|---|
| 성격 | 정적, 읽기 전용 | 동적, 상태 보유 |
| 생성 | `docker build` | `docker run/create` |
| 파일시스템 | 레이어 스택(RO) | + 쓰기 레이어(RW, 휘발) |
| 상태 | 없음 | created/running/paused/exited |
| 사용 | 배포 단위(템플릿) | 실행 단위(프로세스) |

---

## 19. 관련 명령어 요약(보강)

| 명령 | 설명/팁 |
|---|---|
| `docker run` | 컨테이너 생성+시작. `--rm`, `-d`, `-p`, `-v`, `--env`, `--restart` |
| `docker create` / `start` | 생성과 시작 분리 |
| `docker stop` / `kill` | TERM→KILL vs 즉시 KILL |
| `docker pause` / `unpause` | 스케줄링 일시 정지/해제 |
| `docker rm` | 정지된 컨테이너 삭제(`-f` 강제) |
| `docker ps -a` | 전체 컨테이너 목록 |
| `docker logs` | stdout/stderr 로그 조회 |
| `docker exec` | 내부 셸/명령 실행(디버깅) |
| `docker inspect` | 상태/네트워크/마운트/환경 상세 |
| `docker stats` / `top` | 리소스/프로세스 관찰 |
| `docker events` | 실시간 이벤트 스트림 |

---

## 20. 결론

- 컨테이너는 이미지 기반의 **실행 인스턴스**로, 상태를 가지며 파일시스템은 **COW** 방식으로 동작합니다.
- 수명주기는 `create → start → (pause/unpause) → stop/kill → rm` 로 간결하지만, **신호 처리/재시작 정책/자원 제한/데이터 영속화**를 더하면 운영 탄탄도가 크게 올라갑니다.
- 문제는 **관찰(로그/inspect/stats) → 가설 → 수정**의 빠른 루프와 **보안·권한 최소화** 습관으로 해결하십시오.

---

## 부록 A) 간단 수식(가용성 직관)
컨테이너 단일 인스턴스의 근사 가용성:
$$
\text{Availability} \approx \frac{\text{MTBF}}{\text{MTBF} + \text{MTTR}}
$$
`--restart` 정책, 헬스체크, 빠른 롤백/재배포로 **MTTR**을 줄이면 가용성이 높아집니다.

## 부록 B) 치트시트

```bash
# 생성/시작/정지/삭제
docker create --name c1 alpine sleep 1000
docker start c1
docker stop -t 20 c1
docker rm c1

# 즉시 실행(축약)
docker run --rm alpine echo ok

# 내부 진입/로그/상세
docker exec -it c2 sh
docker logs -f c2
docker inspect c2 | jq '.[0].State'

# 자원/프로세스
docker stats
docker top c2

# 네트워크/볼륨
docker network create appnet
docker volume create data1
```
