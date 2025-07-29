---
layout: post
title: Kubernetes - kubectl describe
date: 2025-05-05 20:20:23 +0900
category: Kubernetes
---
# Pod 상태 및 이벤트 모니터링: `kubectl describe` 완전 정리

Kubernetes에서 애플리케이션을 배포했는데…

- Pod가 Pending 상태에서 멈춰 있거나  
- CrashLoopBackOff 상태로 계속 재시작하거나  
- Service가 연결되지 않을 때

이럴 땐 단순한 로그(`kubectl logs`)만으로는 부족합니다.  
이럴 때 사용하는 명령어가 바로 **`kubectl describe`**입니다.

`kubectl describe`는 **Pod를 포함한 모든 Kubernetes 리소스의 상태와 이벤트를 상세하게 확인**할 수 있게 해줍니다.

---

## ✅ 1. 기본 명령어

```bash
kubectl describe pod <pod-name>
```

예:

```bash
kubectl describe pod myapp-79c6dbcf7c-sntbz
```

→ 해당 Pod의 전체 상태, 이벤트, 컨테이너 상태까지 상세하게 출력

---

## ✅ 2. 출력 구조 이해

출력 예시:

```
Name:           myapp-79c6dbcf7c-sntbz
Namespace:      default
Node:           node-1/192.168.1.10
Start Time:     Thu, 25 Jul 2025 12:00:00 +0900
Labels:         app=myapp
Annotations:    ...
Status:         Running
IP:             10.1.2.5
...

Containers:
  myapp:
    Container ID:   docker://123456...
    Image:          myapp:latest
    Image ID:       docker-pullable://myapp@sha256:...
    Port:           8080/TCP
    State:          Running
    Ready:          True
    Restart Count:  0
...

Conditions:
  Type              Status
  Initialized       True
  Ready             True
  ContainersReady   True
  PodScheduled      True

Events:
  Type     Reason     Age               From               Message
  ----     ------     ----              ----               -------
  Normal   Scheduled  1m                default-scheduler  Successfully assigned default/myapp... to node-1
  Normal   Pulling    1m                kubelet            Pulling image "myapp:latest"
  Normal   Pulled     58s               kubelet            Successfully pulled image
  Normal   Created    57s               kubelet            Created container myapp
  Normal   Started    57s               kubelet            Started container myapp
```

---

## ✅ 3. 주요 정보 필드 설명

| 필드 | 설명 |
|------|------|
| **Name / Namespace** | 리소스 이름과 네임스페이스 |
| **Node** | 이 Pod가 스케줄된 노드 이름 |
| **Status** | Pod 전체 상태 (Pending / Running / Succeeded / Failed / Unknown) |
| **IP** | Pod의 내부 IP 주소 |
| **Containers** | 각각의 컨테이너 상태, 이미지, 포트 등 |
| **State** | Running / Waiting / Terminated |
| **Ready** | 컨테이너가 정상적으로 준비되었는지 |
| **Restart Count** | 컨테이너 재시작 횟수 |
| **Conditions** | Pod 전체의 조건 상태 (Ready, PodScheduled 등) |
| **Events** | 스케줄링, 이미지 pull, 시작/종료 등 이벤트 로그 |

---

## ✅ 4. 유용한 사용 예제

### ❗️ Pod이 Pending 상태로 멈춘 경우

```bash
kubectl describe pod myapp
```

🔍 확인할 것:

- **Events 항목**에서 `"FailedScheduling"` 메시지 확인  
  → CPU/Memory 부족, PVC 문제 등  
- `"NodeAffinity"`나 `"Toleration"`이 문제일 수 있음

---

### ❗️ CrashLoopBackOff 발생 시

```bash
kubectl describe pod <pod-name>
```

🔍 확인할 것:

- **State**: Terminated with exit code
- **Restart Count**: 2 이상이면 Crash 반복 중
- **Events**: `"Back-off restarting failed container"`

이 경우, 로그도 함께 확인:

```bash
kubectl logs <pod-name> --previous
```

---

### ❗️ 이미지 Pull 실패

```bash
kubectl describe pod <pod-name>
```

🔍 Events 예시:

```
Failed to pull image "myapp:v1": image not found
```

→ 이미지 이름 오타, Private registry 인증 실패 등이 원인

---

## ✅ 5. 리소스별 describe 사용

`kubectl describe`는 Pod뿐 아니라 모든 Kubernetes 리소스에 사용 가능합니다.

| 리소스 | 명령어 |
|--------|--------|
| Pod | `kubectl describe pod <pod-name>` |
| Node | `kubectl describe node <node-name>` |
| Service | `kubectl describe svc <svc-name>` |
| Deployment | `kubectl describe deploy <deploy-name>` |
| PVC | `kubectl describe pvc <pvc-name>` |
| ConfigMap | `kubectl describe configmap <name>` |

---

## ✅ 6. 특정 네임스페이스 지정

```bash
kubectl describe pod <pod-name> -n my-namespace
```

---

## ✅ 7. 복수 Pod 중 가장 최근 생성된 Pod describe

```bash
kubectl get pod -l app=myapp -o name | tail -n 1 | xargs kubectl describe
```

---

## ✅ 8. describe 결과 필터링 (grep 조합)

```bash
kubectl describe pod <pod-name> | grep -A 5 "Events"
```

→ 이벤트 항목만 빠르게 확인

---

## ✅ 결론

`kubectl describe`는 다음과 같은 문제를 진단할 때 매우 유용합니다:

- Pod이 Pending 상태에서 멈춤
- 이미지 Pull 오류
- CrashLoopBackOff 원인 확인
- PVC 마운트 실패
- 스케줄링 문제