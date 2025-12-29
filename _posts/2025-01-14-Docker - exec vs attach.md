---
layout: post
title: Docker - exec vs attach
date: 2025-01-14 19:20:23 +0900
category: Docker
---
# docker exec vs docker attach

## 개요

`docker exec`와 `docker attach`는 Docker 컨테이너와 상호작용하는 두 가지 핵심 명령어입니다. 비슷해 보일 수 있지만, 작동 방식과 사용 목적에서 근본적인 차이가 있습니다. 이 가이드를 통해 두 명령어의 정확한 차이점과 적절한 사용 시기를 이해하실 수 있습니다.

## 핵심 비교

| 구분 | `docker exec` | `docker attach` |
|---|---|---|
| 개념 | 컨테이너 내부에 **새로운 프로세스**를 생성하여 실행 | 컨테이너의 **기본 프로세스(PID 1)의 표준 입출력에 연결** |
| 대상 | 새로 생성되는 개별 프로세스 | 이미 실행 중인 PID 1 프로세스 |
| 동시 실행 | 여러 exec 프로세스를 동시에 실행 가능 | 여러 터미널에서 동시에 attach 가능 (동일 입출력 공유) |
| 종료 영향 | exec 프로세스만 종료 (컨테이너는 계속 실행) | PID 1 프로세스 종료 시 컨테이너 종료 |
| 입력 조건 | `-i` 옵션만 있으면 입력 가능 | 컨테이너가 `-i` 옵션으로 시작되었어야 입력 가능 |
| 주용도 | 디버깅, 셸 진입, 일회성 명령 실행 | PID 1 프로그램 직접 관찰 및 제어 |
| 신호 처리 | exec로 생성된 프로세스에 신호 전달 | PID 1 프로세스에 신호 전달 |
| 운영 권장도 | **높음** (안전) | **낮음** (주의 필요) |

중요한 점은 `attach`는 여러 터미널에서 동시에 연결할 수 있지만, 모든 연결이 동일한 표준 입출력을 공유하므로 입력과 출력이 혼합될 수 있어 운영 환경에서는 위험할 수 있습니다.

---

## 작동 방식 이해

### docker exec - 새 프로세스 생성

```
[호스트 터미널] --HTTP API--> [dockerd] --(exec)--> [container: PID 1]
                                                      └───> [PID X: bash/top/... ← 이걸 내가 띄움]
```

`docker exec`는 컨테이너 내부에 새로운 프로세스(PID X)를 생성한 후, 사용자의 터미널과 해당 프로세스의 입출력을 연결합니다. 기존에 실행 중이던 PID 1 프로세스는 영향을 받지 않고 계속 실행됩니다.

### docker attach - 기존 프로세스에 연결

```
[호스트 터미널] --attach--> [container: PID 1 (애플리케이션)]
```

`docker attach`는 컨테이너의 PID 1 프로세스의 표준 입력, 출력, 에러 스트림에 직접 연결합니다. PID 1이 bash 같은 셸인 경우, `exit` 명령어를 실행하면 컨테이너가 종료됩니다.

---

## 실전 사용 패턴

### 셸 진입 및 디버깅 (docker exec 권장)

```bash
# 다양한 컨테이너 셸 접근
docker exec -it web bash           # Debian/Ubuntu 기반 컨테이너
docker exec -it web sh             # BusyBox/Alpine 컨테이너
docker exec -it --user 65532:65532 web sh     # 비루트 사용자로 접근

# 환경 변수 및 디렉토리 확인
docker exec -it -e DEBUG=1 web env | sort     # 환경 변수 확인
docker exec -it -w /var/log web sh            # 특정 작업 디렉토리 지정
```

### PID 1 직접 조작 (docker attach - 주의 필요)

```bash
# PID 1이 bash인 컨테이너에 연결 (비권장: exit 시 컨테이너 종료)
docker attach c_bash

# 안전하게 분리하기: Ctrl + P, Ctrl + Q
```

### 로그 확인 (docker logs 권장)

```bash
# attach 대신 logs 명령어 사용
docker logs -f --tail=200 web      # 실시간 로그 모니터링
docker logs --since=10m web        # 지난 10분간의 로그 확인
```

로그 열람에는 `attach` 대신 `logs` 명령어를 사용하는 것이 안전합니다. 출력이 섞이거나 입력이 오염되는 것을 방지할 수 있습니다.

---

## 옵션과 터미널 설정

### -i와 -t 옵션의 의미

- `-i` (interactive): 표준 입력을 열어 입력을 전송할 수 있게 합니다.
- `-t` (TTY): 의사 터미널을 할당하여 줄바꿈, 컬러, 커서 UI 등이 정상적으로 작동하게 합니다.

```bash
# 일반적인 사용 예시
docker exec -it web bash           # 대화형 셸 실행

# stdin만 필요한 경우
docker exec -i web sh -lc 'cat -v' # tty 없이 stdin만 사용
```

### attach 시 입력 문제 해결

`attach` 명령어에서 입력이 전달되지 않는 경우, 해당 컨테이너가 처음 실행될 때 `-i` 옵션 없이 시작되었을 수 있습니다. `attach`는 컨테이너 생성 시 열려있던 표준 입력을 사용하기 때문에, 처음부터 `-i` 옵션으로 실행된 컨테이너여야 입력이 가능합니다.

---

## 신호 처리와 PID 1의 특수성

### 컨테이너 종료 방법

```bash
docker stop web         # SIGTERM 전송 후 grace period 동안 대기 후 SIGKILL
docker kill web         # 즉시 SIGKILL 전송
docker kill --signal=HUP web   # HUP 신호 전달 (로그 회전 등에 사용)
```

`attach` 중에 Ctrl+C를 누르면 SIGINT 신호가 PID 1 프로세스에 전달될 수 있습니다. 애플리케이션은 SIGTERM 신호를 적절히 처리하는 정상 종료 루틴을 구현해야 안전합니다.

### PID 1의 특별한 동작

PID 1 프로세스는 좀비 프로세스 처리와 신호 전달 동작에서 일반 프로세스와 다르게 동작할 수 있습니다. 이를 위해 `--init` 옵션을 사용하거나 ENTRYPOINT를 올바른 exec form으로 작성하는 것이 권장됩니다.

```bash
# init 프로세스 사용
docker run -d --init --name api myorg/api:1.0
```

```Dockerfile
# 올바른 exec form 사용
ENTRYPOINT ["gunicorn"]
CMD ["-w","2","-b","0.0.0.0:8080","app:app"]
```

---

## 안전한 분리와 사용자 설정

### 기본 분리 키

- `attach` 모드에서 분리하기: `Ctrl + P` → `Ctrl + Q`
- `exec`로 실행한 셸에서 나가기: `exit` 명령어 실행

### 분리 키 사용자 정의

`~/.docker/config.json` 파일에서 분리 키 조합을 변경할 수 있습니다.

```json
{
  "detachKeys": "ctrl-p,ctrl-q"
}
```

---

## 보안과 권한 관리

### 최소 권한 원칙 적용

```bash
# 비루트 사용자로 접근
docker exec -it --user 65532:65532 web sh

# 권한 제한 명시적 설정
docker exec -it --privileged=false web sh
```

`exec` 명령어로 루트 셸에 쉽게 접근할 수 있으므로, RBAC(Role-Based Access Control)와 적절한 감사 시스템이 필요합니다.

### 읽기 전용 파일 시스템 활용

```bash
docker run --rm \
  --read-only \
  --tmpfs /tmp --tmpfs /run \
  --cap-drop ALL --security-opt no-new-privileges \
  --user 65532:65532 \
  myimg:prod
```

읽기 전용 루트 파일 시스템과 제한된 임시 파일 시스템을 사용하면, `exec`로 접근했을 때의 영향을 최소화할 수 있습니다.

---

## 문제 해결 가이드

| 증상 | 원인 | 해결 방법 |
|---|---|---|
| attach 입력 미반응 | 컨테이너가 `-i` 없이 시작 | 컨테이너를 `-i` 옵션으로 재시작, 로그는 `logs` 명령어 사용 |
| attach에서 exit 시 컨테이너 종료 | PID 1이 셸/REPL | PID 1을 데몬/서버 프로세스로 설정, 셸은 exec로만 사용 |
| 화면 깨짐/컬러 이상 | TTY 미할당 | `-t` 옵션 추가, 비대화식 명령에는 `-t` 제거 |
| Ctrl+C가 동작 안 함 | 신호 전파 문제 | `--init` 옵션 사용, exec form ENTRYPOINT, 애플리케이션 신호 처리 구현 |
| 여러 사용자 접속 시 입력 혼합 | 다중 attach | attach 사용 제한, `logs` 명령어나 APM 도구 사용 |
| attach 로그 확인 시 출력 끊김 | 표준 출력 버퍼 문제 | `docker logs -f` 명령어 사용 |

---

## 실전 시나리오 레시피

### 서비스 실행 유지하며 내부 점검

```bash
docker exec -it web sh -lc 'nginx -v; ls -al /usr/share/nginx/html; ss -ltn'
```

### 잘못된 ENTRYPOINT 수정

```Dockerfile
# 문제 있는 예시 (shell form)
ENTRYPOINT service nginx start

# 올바른 예시 (exec form)
ENTRYPOINT ["nginx","-g","daemon off;"]
```

### 실행 중인 컨테이너 모니터링

```bash
docker exec -it web top                    # 시스템 모니터링
docker exec -it web tail -f /var/log/nginx/access.log  # 로그 실시간 확인
```

### 필요한 경우 attach 사용

```bash
docker attach --sig-proxy=true web    # 신호를 PID 1에 전달
# 분리: Ctrl+P, Ctrl+Q
```

---

## 실제 사례 분석

### Nginx 컨테이너 관리

```bash
docker run -d --name web -p 8080:80 nginx:alpine
docker logs -f web                # 로그 확인 (안전)
docker exec -it web sh            # 내부 점검 (안전)
# attach 사용 시 Ctrl+C가 PID1에 SIGINT 전달 → 종료 위험
```

### 학습용 컨테이너 (PID 1 = bash)

```bash
docker run -it --name lab ubuntu bash   # PID1이 bash
# 다른 터미널에서:
docker attach lab
# 여기서 'exit' 입력 시 컨테이너 종료
# 안전한 분리: Ctrl+P, Ctrl+Q
```

---

## 명령어 치트시트

```bash
# 셸 접근 (권장)
docker exec -it <컨테이너> bash|sh

# 일회성 진단 명령
docker exec -it <컨테이너> sh -lc 'uname -a; env | sort | head -20'

# 로그 확인 (안전)
docker logs -f --tail=200 <컨테이너>

# attach (주의 필요)
docker attach <컨테이너>     # 분리: Ctrl+P, Ctrl+Q

# 신호 전송 및 종료
docker stop -t 20 <컨테이너>        # SIGTERM 후 20초 유예
docker kill --signal=HUP <컨테이너> # HUP 신호 전달

# 보안 강화 실행
docker run --rm --read-only --tmpfs /tmp --cap-drop ALL \
  --security-opt no-new-privileges --user 65532:65532 myimg
```

---

## 결론

`docker exec`와 `docker attach`는 목적과 동작 방식에서 근본적인 차이가 있는 명령어입니다. 운영 환경에서는 `docker exec`가 더 안전하고 유연한 선택입니다. 이 명령어는 컨테이너 내부에 새로운 프로세스를 생성하여 기존 서비스에 영향을 주지 않고 디버깅과 관리를 가능하게 합니다.

반면 `docker attach`는 컨테이너의 주요 프로세스(PID 1)에 직접 연결되므로, 실수로 컨테이너를 종료시키거나 입출력이 혼합되는 문제가 발생할 수 있습니다. 따라서 `attach`는 특별한 경우에만 사용하고, 대부분의 작업에는 `exec`를 사용하는 것이 권장됩니다.

로그 확인에는 `docker logs` 명령어를, 컨테이너 제어에는 `exec`를, 종료에는 `stop`이나 `kill`을 사용하는 등 각 도구의 역할을 명확히 구분하여 사용하면 운영 중 발생할 수 있는 문제를 최소화할 수 있습니다. 이러한 구분은 Docker 컨테이너를 안정적으로 운영하는 데 중요한 원칙입니다.