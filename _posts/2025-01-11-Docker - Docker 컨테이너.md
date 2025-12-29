---
layout: post
title: Docker - Docker 컨테이너
date: 2025-01-11 19:20:23 +0900
category: Docker
---
# Docker 컨테이너: 상태와 수명주기 완벽 이해

## 컨테이너란 무엇인가?

Docker 컨테이너는 Docker 이미지를 기반으로 생성된 실행 인스턴스입니다. 이미지가 애플리케이션과 그 실행 환경을 포함한 불변의 템플릿이라면, 컨테이너는 이 템플릿을 실행한 동적인 객체라고 할 수 있습니다.

### 컨테이너의 핵심 특성

- **이미지 기반**: 컨테이너는 항상 이미지로부터 생성되며, 이미지의 모든 내용을 포함합니다.
- **격리된 환경**: 각 컨테이너는 독립된 파일시스템, 네트워크, 프로세스 공간을 갖습니다.
- **경량성**: 가상머신과 달리 호스트의 커널을 공유하므로 시작이 빠르고 자원 효율이 높습니다.
- **일시성**: 기본적으로 컨테이너의 변경사항은 컨테이너가 삭제되면 사라집니다. 데이터를 영구적으로 보존하려면 볼륨을 사용해야 합니다.

---

## 컨테이너의 파일시스템 구조

컨테이너의 파일시스템은 레이어 구조로 구성되어 있으며, 다음과 같은 방식으로 동작합니다:

```
+-----------------------------------+
| Writable Container Layer (upper)  |  ← 컨테이너 전용 쓰기 레이어
+-----------------------------------+
| Read-only Image Layers (lower)    |  ← FROM ~ RUN ~ COPY로 생긴 이미지 레이어
+-----------------------------------+
| Host Kernel (shared)              |
+-----------------------------------+
```

**Copy-on-Write(COW) 방식**: 읽기 전용 레이어의 파일을 수정하려고 할 때, 해당 파일이 쓰기 가능한 레이어로 복사된 후 수정됩니다. 이 방식은 디스크 공간을 절약하고 여러 컨테이너가 동일한 베이스 이미지를 효율적으로 공유할 수 있게 합니다.

---

## 컨테이너의 상태와 상태 전이

Docker 컨테이너는 다음과 같은 상태를 가질 수 있으며, 이 상태들은 명령어에 따라 전이됩니다:

### 주요 컨테이너 상태

1. **created**: 이미지로부터 컨테이너가 생성되었지만 아직 시작되지 않은 상태
2. **running**: 컨테이너 내의 애플리케이션이 실행 중인 상태
3. **paused**: 컨테이너의 프로세스 실행이 일시 중지된 상태
4. **exited**: 컨테이너 내의 프로세스가 종료된 상태 (정상 또는 비정상 종료)
5. **dead**: 내부 오류 등으로 인한 비정상적인 상태 (드물게 발생)

### 상태 전이 흐름

```
이미지
  ↓
create  →  [created]
  ↓
start   →  [running] → stop(SIGTERM) → [exited] → rm → 삭제
                 └─ kill(SIGKILL) ───┘
pause → [paused] → unpause → [running]
```

### 상태 확인 방법
```bash
# 모든 컨테이너의 상태 확인
docker ps -a

# 특정 컨테이너의 상세 상태 확인
docker inspect <컨테이너명> | jq '.[0].State'
```

---

## 컨테이너 수명주기 관리 명령어

### 컨테이너 생성과 시작
```bash
# 컨테이너 생성만 (시작하지 않음)
docker create --name demo ubuntu sleep 600

# 컨테이너 시작
docker start demo

# 생성과 시작을 한 번에 (가장 일반적인 사용법)
docker run -d --name web -p 8080:80 nginx:alpine
```

**참고**: `docker run` 명령은 내부적으로 `create`와 `start`를 순차적으로 실행하는 축약형입니다.

### 컨테이너 일시 중지 및 재개
```bash
# 컨테이너 일시 중지
docker pause demo

# 컨테이너 재개
docker unpause demo
```

### 컨테이너 정지와 강제 종료
```bash
# 정상 종료 (SIGTERM 신호 전송)
docker stop demo  # 기본 10초 대기 후 강제 종료

# 강제 종료 (SIGKILL 신호 즉시 전송)
docker kill demo

# 정상 종료 대기 시간 조정
docker stop -t 30 demo  # 30초까지 대기
```

**중요**: 애플리케이션은 SIGTERM 신호를 적절히 처리하여 데이터 일관성을 유지하도록 구현되어야 합니다. SIGTERM 핸들러를 구현하면 정리 작업(예: 연결 종료, 데이터 플러시)을 수행할 수 있습니다.

### 컨테이너 삭제
```bash
# 컨테이너 삭제 (정지된 컨테이너만 삭제 가능)
docker rm demo

# 실행 중인 컨테이너 강제 삭제
docker rm -f demo

# 일회성 작업을 위한 실행 및 자동 삭제
docker run --rm alpine echo "작업 완료"
```

---

## 실무에서 자주 사용하는 실행 패턴

### 대화형 디버깅 모드
```bash
docker run -it --name temp-container ubuntu bash
```

### 백그라운드 서비스 실행
```bash
docker run -d --name web-server -p 8080:80 nginx:alpine
```

### 보안 강화를 위한 실행 옵션
```bash
docker run --rm \
  --read-only \                      # 루트 파일시스템 읽기 전용
  --tmpfs /tmp --tmpfs /run \        # 필요한 임시 디렉터리만 쓰기 가능
  --cap-drop ALL \                   # 모든 특권 제거
  --security-opt no-new-privileges \ # 새 권한 획득 방지
  --user 1000:1000 \                 # 비root 사용자로 실행
  --pids-limit 128 \                 # 최대 프로세스 수 제한
  --cpus 1 \                         # CPU 사용량 제한
  --memory 256m \                    # 메모리 사용량 제한
  nginx:alpine
```

---

## 컨테이너 재시작 정책

서비스 가용성을 높이기 위해 컨테이너의 자동 재시작 정책을 설정할 수 있습니다:

```bash
# 오류 발생 시 자동 재시작 (가장 일반적)
docker run -d --restart=on-failure:5 --name api myapi:latest

# 항상 재시작 (사용자가 명시적으로 중지하지 않는 한)
docker run -d --restart=always --name db postgres:latest

# 사용자가 중지하지 않은 경우에만 재시작 (권장)
docker run -d --restart=unless-stopped --name web nginx:alpine
```

### 재시작 정책 옵션
- **no**: 자동 재시작 없음 (기본값)
- **on-failure[:N]**: 비정상 종료 시 최대 N회 재시도
- **always**: 항상 재시작 (수동 중지 제외)
- **unless-stopped**: 사용자가 명시적으로 중지하지 않는 한 재시작

---

## 컨테이너 모니터링과 디버깅

### 로그 확인
```bash
# 기본 로그 출력
docker logs web-server

# 실시간 로그 모니터링
docker logs -f web-server

# 최근 100줄만 확인
docker logs --tail=100 web-server

# 최근 10분간의 로그 확인
docker logs --since=10m web-server
```

### 컨테이너 내부 접속
```bash
# 실행 중인 컨테이너에 셸 접속
docker exec -it web-server sh      # Alpine/BusyBox 기반
docker exec -it web-server bash    # Ubuntu/Debian 기반
```

**주의**: `docker attach`는 컨테이너의 주 프로세스에 직접 연결하므로, 일반적으로 `docker exec`를 사용하는 것이 더 안전하고 유연합니다.

### 리소스 사용량 모니터링
```bash
# 실시간 리소스 사용량 모니터링
docker stats

# 특정 컨테이너의 프로세스 목록 확인
docker top web-server
```

### 상세 정보 조회
```bash
# 컨테이너의 모든 상세 정보 조회
docker inspect web-server

# 특정 정보만 추출 (예: 네트워크 설정)
docker inspect web-server | jq '.[0].NetworkSettings'
```

---

## 컨테이너 데이터 관리

컨테이너의 파일시스템은 기본적으로 휘발성입니다. 데이터를 영구적으로 보존하려면 볼륨을 사용해야 합니다.

### 명명된 볼륨 사용
```bash
# 볼륨 생성
docker volume create db-data

# 볼륨을 사용하여 데이터베이스 실행
docker run -d --name database \
  -v db-data:/var/lib/postgresql/data \
  postgres:latest
```

### 호스트 디렉터리 마운트
```bash
# 호스트 디렉터리를 컨테이너에 마운트
mkdir -p ~/website
docker run -d --name web \
  -v ~/website:/usr/share/nginx/html:ro \
  nginx:alpine
```

---

## 네트워킹 기초

컨테이너 간 통신을 위해 사용자 정의 네트워크를 생성하면 컨테이너 이름으로 서로 접근할 수 있습니다.

```bash
# 사용자 정의 네트워크 생성
docker network create app-network

# 네트워크를 사용하여 컨테이너 실행
docker run -d --name api --network app-network -p 8080:8080 myapi:latest
docker run -d --name web --network app-network -p 80:80 nginx:alpine

# 동일 네트워크 내의 컨테이너는 이름으로 접근 가능
docker exec api curl http://web
```

---

## 일반적인 문제 해결

### 컨테이너가 즉시 종료되는 경우
```bash
# 로그 확인
docker logs [컨테이너명]

# 종료 코드 확인
docker inspect [컨테이너명] | jq '.[0].State.ExitCode'
```

### 포트 접근 문제
```bash
# 포트 매핑 확인
docker port [컨테이너명]

# 실행 중인 서비스 확인
docker exec [컨테이너명] netstat -tln
```

### 메모리 부족 문제
```bash
# 메모리 사용량 모니터링
docker stats

# 메모리 제한 설정
docker run -d --memory="512m" --memory-swap="1g" myapp:latest
```

---

## 실무 모범 사례 요약

1. **항상 컨테이너에 이름 지정**: `--name` 옵션을 사용하여 컨테이너에 의미 있는 이름을 부여하세요.
2. **적절한 재시작 정책 설정**: 서비스 유형에 맞는 재시작 정책을 선택하세요.
3. **리소스 제한 설정**: CPU, 메모리, 프로세스 수를 적절히 제한하여 호스트 시스템을 보호하세요.
4. **데이터는 볼륨에 저장**: 컨테이너의 중요한 데이터는 반드시 볼륨에 저장하세요.
5. **보안 강화**: 최소한의 권한으로 컨테이너를 실행하고, 가능한 경우 비root 사용자를 사용하세요.
6. **로그 관리**: 애플리케이션 로그는 stdout/stderr로 출력하여 Docker가 로그를 관리할 수 있게 하세요.

---

## 결론

Docker 컨테이너는 현대 애플리케이션 배포와 관리를 혁신적으로 단순화한 기술입니다. 컨테이너의 상태와 수명주기를 이해하는 것은 효과적인 Docker 사용을 위한 기초입니다.

컨테이너는 `생성 → 시작 → 실행 → 정지 → 삭제`의 간결한 생명주기를 가지지만, 재시작 정책, 리소스 관리, 데이터 영속화, 네트워킹, 보안 설정 등을 추가함으로써 프로덕션 환경에서도 견고하게 운영할 수 있습니다.

문제가 발생했을 때는 `docker logs`, `docker inspect`, `docker stats` 명령어를 활용하여 체계적으로 진단하고 해결하는 습관을 기르는 것이 중요합니다. 이러한 도구들을 효과적으로 활용하면 Docker 컨테이너를 자신 있게 관리하고 운영할 수 있을 것입니다.

마지막으로, Docker 컨테이너는 도구일 뿐 목적이 아님을 기억하세요. 컨테이너 기술은 개발자와 운영팀이 애플리케이션에 더 집중할 수 있도록 돕는 수단으로 활용되어야 합니다.