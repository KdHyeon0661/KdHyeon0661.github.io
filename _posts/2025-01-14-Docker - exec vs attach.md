---
layout: post
title: Docker - exec vs attach
date: 2025-01-14 19:20:23 +0900
category: Docker
---
# `docker exec` vs `docker attach`

## 0. 한눈에 비교(정밀 보강판)

| 구분 | `docker exec` | `docker attach` |
|---|---|---|
| 핵심 개념 | **컨테이너 내부에 새 프로세스**를 생성해 실행 | 컨테이너 **PID 1(기본 프로세스)의 STDIN/STDOUT/STDERR에 연결** |
| 대상 | 개별 **새 프로세스** | **이미 실행 중인 PID 1** |
| 동시성 | **여러 개** exec 동시 실행 가능 | **여러 터미널이 동시에 attach 가능**(동일 PID 1 입출력 공유) |
| 종료 영향 | exec 프로세스만 종료됨(컨테이너는 계속) | PID 1 종료 시 컨테이너 **종료**(attach에서 `exit`가 PID 1을 끝내면 컨테이너 down) |
| 입력 가능 조건 | `-i`(exec 호출 측)만 충족하면 됨 | 컨테이너가 **`-i` 로 시작**(STDIN 열림)해야 입력 가능 |
| 주용도 | 디버깅 셸 진입, 일회성 명령, 점검 스크립트 | PID 1 프로그램 **직접 제어/관찰**, TTY 앱의 **정면 조작** |
| 신호 처리 | **exec로 띄운 프로세스**에 신호 | **PID 1**에 신호 전달(기본: proxy) |
| 권장도(운영) | **매우 높음**(안전) | **낮음**(주의 필요: PID 1 종료, 입력 충돌, 로그 뭉개짐) |

> 중요 정정: `attach`는 **동시 하나만**이 아니라, 여러 터미널에서 **동시에** 붙을 수 있습니다. 다만 **모두가 동일한 STDIN/STDOUT**을 공유하므로 입력/출력이 뒤섞일 수 있어 운영 현장에서는 위험합니다.

---

## 1. 멘탈 모델: 무엇이 어디에 붙는가?

### 1.1 `docker exec` — 새 프로세스 스폰
```
[호스트 터미널] --HTTP API--> [dockerd] --(exec)--> [container: PID 1]
                                                      └───> [PID X: bash/top/... ← 이걸 내가 띄움]
```
- **컨테이너 내부에 새로운 프로세스(PID X)** 를 띄운 뒤, 내 터미널과 **PID X**의 TTY/표준 IO를 연결.
- 원래 돌고 있던 PID 1(애플리케이션)은 **그대로**.

### 1.2 `docker attach` — PID 1의 표준 IO에 직결
```
[호스트 터미널] --attach--> [container: PID 1 (애플리케이션)]
```
- **PID 1**의 **STDIN/STDOUT/STDERR**에 직결.  
- PID 1이 `bash` 같은 셸이면, `exit`가 곧 컨테이너 종료가 됨.

---

## 2. 실전 사용 패턴

### 2.1 셸 진입/디버깅(권장: exec)
```bash
docker exec -it web bash           # Debian/Ubuntu 기반
docker exec -it web sh             # BusyBox/Alpine
docker exec -it --user 65532:65532 web sh     # 비루트로 진입
docker exec -it -e DEBUG=1 web env | sort     # 환경 변수 확인
docker exec -it -w /var/log web sh            # 작업 디렉토리 지정
```

### 2.2 PID 1 직접 조작(주의: attach)
```bash
# PID 1이 'bash'인 컨테이너에 붙기(매우 비권장: exit하면 컨테이너 다운)
docker attach c_bash

# 안전 분리(detach): Ctrl + P, Ctrl + Q
# Windows Terminal/PowerShell에서도 동일(키 전송만 다를 수 있음)
```

### 2.3 로그는 attach 말고 logs
```bash
docker logs -f --tail=200 web      # 안전하고 읽기 전용
docker logs --since=10m web
```
- **로그 열람에는 `attach` 대신 `logs`** 를 습관화(출력 섞임/입력 오염 방지).

---

## 3. 옵션·TTY·STDIN 핵심

### 3.1 `-i` / `-t` 의미
- `-i`(interactive): **STDIN을 열어둠**(입력 전송 가능)
- `-t`(TTY): **의사 터미널 할당**(줄바꿈/컬러/쿠르세스 UI 등 정상 동작)

예시:
```bash
docker exec -it web bash           # 보통 이 조합
docker exec -i web sh -lc 'cat -v' # stdin만 필요, tty 불필요
```

### 3.2 attach 시 입력이 안 들어간다면
- 해당 컨테이너가 **처음 run/start 때 `-i` 없이** 올라왔을 수 있음.  
  `attach`는 **컨테이너 생성 시 열어둔 STDIN**을 그대로 씁니다. 처음부터 `-i`로 시작해야 입력이 가능.

---

## 4. 신호(signals)와 PID 1

### 4.1 stop vs kill
```bash
docker stop web         # SIGTERM → grace 기간 → SIGKILL
docker kill web         # 즉시 SIGKILL
docker kill --signal=HUP web   # HUP 전달(로그 롤테이트 등)
```
- `attach` 중 **Ctrl+C**(SIGINT)가 PID 1에 전달될 수 있음(신호 proxy).  
- 애플리케이션은 **SIGTERM 처리**(정상 종료 루틴) 코드를 반드시 가져야 안전.

### 4.2 PID 1의 특수성
- PID 1은 **좀비 reaping/신호 전달** 동작이 일반 프로세스와 다를 수 있음.
- 권장: 컨테이너 실행 시 `--init` 사용 또는 ENTRYPOINT를 올바른 **exec form**으로.
```bash
docker run -d --init --name api myorg/api:1.0
```
```Dockerfile
# 올바른 exec form
ENTRYPOINT ["gunicorn"]
CMD ["-w","2","-b","0.0.0.0:8080","app:app"]
```

---

## 5. 안전 분리(detach)와 키 커스텀

### 5.1 기본 분리 키
- `attach`에서 **분리(detach)**: `Ctrl + P` → `Ctrl + Q`
- `exec` 쉘은 그냥 `exit`(프로세스 종료)로 빠져나오면 됨.

### 5.2 분리 키 커스텀
`~/.docker/config.json`:
```json
{
  "detachKeys": "ctrl-p,ctrl-q"
}
```
다른 조합으로 바꿀 수 있습니다(예: `"ctrl-\\,ctrl-\\")`.

---

## 6. 다중 attach와 출력 섞임

- **여러 터미널에서 동시에 attach 가능**합니다.  
- 단, **모두가 같은 STDIN/STDOUT**을 공유하므로 입력이 충돌하고 출력이 섞입니다.
- 운영 환경에서는 **attach 지양** + 관찰은 `logs -f`, 제어는 **API/관리 명령** 사용 권장.

---

## 7. 보안·권한·감사 포인트

### 7.1 최소 권한 원칙
```bash
docker exec -it --user 65532:65532 web sh    # 비루트 사용자로 exec
docker exec -it --privileged=false web sh    # (기본 false) 명시적
```
- `exec` 시 **루트 셸**을 쉽게 얻을 수 있으므로 **RBAC/감사** 필요.
- `--privileged`로 exec를 띄우면 **컨테이너 보안 모델 무력화** 위험. 운영 금지.

### 7.2 읽기 전용 루트FS + tmpfs
```bash
docker run --rm \
  --read-only \
  --tmpfs /tmp --tmpfs /run \
  --cap-drop ALL --security-opt no-new-privileges \
  --user 65532:65532 \
  myimg:prod
```
- exec로 들어갈 때도 쓰기 경로가 제한되어 **파급력 축소**.

---

## 8. 문제 해결(attach/exec 관련)

| 증상 | 원인 | 진단 | 해결 |
|---|---|---|---|
| attach 입력 미반응 | 컨테이너가 `-i` 없이 시작 | `docker inspect ... .Config.OpenStdin` | 처음부터 `-i`로 run, 로그는 `logs`로 |
| attach에서 `exit` 했더니 컨테이너 종료 | PID 1이 셸/REPL | `docker ps`, ENTRYPOINT 확인 | PID 1을 **데몬/서버 프로세스**로, 셸은 exec로만 |
| 화면 깨짐/컬러 이상 | TTY 미할당 | `-t` 유무 확인 | `-t` 추가, 비대화식이면 `-t` 제거 |
| Ctrl+C가 앱에 안 먹음 | 신호 전파 미구현/랩퍼 문제 | ENTRYPOINT/신호 핸들러 확인 | `--init` 사용, exec form, 앱에 SIGTERM/SIGINT 처리 |
| 여러 명이 붙어 입력 뒤섞임 | 다중 attach | `docker events`로 접속 확인 | attach 금지, `logs`/APM 대체 |
| `attach`로 로그 보니 끊김/혼합 | STDOUT 버퍼/다중 라이터 | `docker logs -f` | **항상 logs 사용** |

---

## 9. 상황별 레시피

### 9.1 “서비스는 계속, 내부만 보고 싶다”
```bash
docker exec -it web sh -lc 'nginx -v; ls -al /usr/share/nginx/html; ss -ltn'
```

### 9.2 “PID 1이 셸이라 위험해” → ENTRYPOINT 교정
```Dockerfile
# 나쁜 예(쉘 form): 신호 전달/종료 혼선
ENTRYPOINT service nginx start

# 좋은 예(exec form): PID 1 = nginx 포그라운드
ENTRYPOINT ["nginx","-g","daemon off;"]
```

### 9.3 “실행 중 컨테이너에 top/tail 임시 붙이기”
```bash
docker exec -it web top
docker exec -it web tail -f /var/log/nginx/access.log
```

### 9.4 “attach를 꼭 써야 한다면”
```bash
docker attach --sig-proxy=true web    # 기본값, PID 1에게 신호 전달
# 분리: Ctrl+P, Ctrl+Q
```

---

## 10. 케이스 스터디

### 10.1 Nginx 컨테이너
```bash
docker run -d --name web -p 8080:80 nginx:alpine
docker logs -f web                # 로그는 logs로
docker exec -it web sh            # 내부 점검은 exec로
# attach로 붙었다가 Ctrl+C 누르면 PID1에 SIGINT 전달 → 종료 위험
```

### 10.2 PID 1 = bash (학습용 컨테이너)
```bash
docker run -it --name lab ubuntu bash   # PID1= bash
# 다른 터미널:
docker attach lab
# 여기서 'exit' 하면 컨테이너 종료. 안전 분리는 Ctrl+P, Ctrl+Q
```

---

## 11. 고급: nsenter와의 비교(참고)

- `nsenter`는 **호스트에서 컨테이너 네임스페이스**에 진입(별도 런타임 경유 X).  
- Docker 표준 사용성/이식성은 `docker exec`이 우수. `nsenter`는 낮은 레벨의 리눅스 운영/디버깅에 적합.

---

## 12. 치트시트

```bash
# 셸 진입(권장)
docker exec -it <ctr> bash|sh

# 일회성 진단
docker exec -it <ctr> sh -lc 'uname -a; env | sort | head -20'

# 로그는 attach 말고
docker logs -f --tail=200 <ctr>

# attach(주의: PID1 제어, 분리키)
docker attach <ctr>     # Detach = Ctrl+P, Ctrl+Q

# 신호/종료
docker stop -t 20 <ctr>        # SIGTERM → 20초 유예
docker kill --signal=HUP <ctr> # HUP 전달

# 보안 실행 예(컨테이너 띄울 때)
docker run --rm --read-only --tmpfs /tmp --cap-drop ALL \
  --security-opt no-new-privileges --user 65532:65532 myimg
```

---

## 13. 결론 요약

- **운영/디버깅 표준**은 `docker exec` 입니다. 안전하고, 컨테이너 수명에 **영향 최소**.  
- `docker attach`는 **PID 1 IO에 직결**되어 위험(종료·입력 충돌·로그 오염). 꼭 필요할 때만, **detach 키** 숙지.  
- **로그는 `docker logs`**, **제어는 `exec`**, **종료는 stop/kill**, **가시성은 stats/inspect**—역할을 분리하면 사고가 줄어듭니다.
