---
layout: post
title: Kubernetes - kubectl logs
date: 2025-05-05 19:20:23 +0900
category: Kubernetes
---
# Kubernetes 로그 확인: `kubectl logs` 완전 정리

Kubernetes 클러스터에서 애플리케이션을 배포한 후,  
문제가 발생하거나 상태를 확인하고 싶을 때 **가장 먼저 확인하는 것**은 바로 **로그(Log)**입니다.

`kubectl logs` 명령어는 **Pod 안에서 실행 중인 컨테이너의 로그를 확인**하는 가장 기본적인 도구입니다.

---

## ✅ 1. 기본 사용법

```bash
kubectl logs <pod-name>
```

Pod의 기본(첫 번째) 컨테이너 로그를 출력합니다.

예:

```bash
kubectl logs my-nginx-74b46bcf5b-wbp5d
```

---

## ✅ 2. 멀티 컨테이너 Pod 로그 확인

Pod에 컨테이너가 **2개 이상**일 경우에는 `-c` 옵션으로 컨테이너를 지정해야 합니다.

```bash
kubectl logs <pod-name> -c <container-name>
```

예:

```bash
kubectl logs my-app-pod -c main-container
```

컨테이너 이름은 다음 명령어로 확인할 수 있습니다:

```bash
kubectl get pod my-app-pod -o jsonpath='{.spec.containers[*].name}'
```

---

## ✅ 3. 이전 컨테이너 로그 확인 (CrashLoopBackOff 등)

Pod가 재시작되었거나 Crash가 발생했을 때,  
이전 컨테이너의 로그를 보려면 `--previous` 옵션을 사용합니다.

```bash
kubectl logs <pod-name> --previous
```

예:

```bash
kubectl logs my-api-pod --previous
```

→ "CrashLoopBackOff" 상태일 때 매우 유용합니다.

---

## ✅ 4. 실시간 로그 스트리밍

로그를 **실시간으로 모니터링**하려면 `-f` 또는 `--follow` 옵션을 사용합니다.

```bash
kubectl logs -f <pod-name>
```

또는 멀티 컨테이너일 경우:

```bash
kubectl logs -f <pod-name> -c <container-name>
```

→ tail -f 와 유사하게 로그가 출력되는 즉시 확인 가능

---

## ✅ 5. 여러 Pod의 로그 보기 (예: ReplicaSet)

`kubectl logs`는 한 번에 하나의 Pod만 지원합니다.  
하지만 `kubectl get pods`와 `xargs`, `grep`, `tail` 등을 조합하면 여러 Pod의 로그도 확인할 수 있습니다.

예: `app=my-api` 라벨을 가진 Pod들의 로그

```bash
kubectl get pods -l app=my-api -o name | \
xargs -I {} kubectl logs {} > all-logs.txt
```

---

## ✅ 6. 특정 시간 로그 추출 (tail, since)

Helm, CI/CD 등 자동화 환경에서는 **최근 몇 줄 또는 시간 단위로 로그 추출**이 필요합니다.

### ✅ 최근 N줄만 보기

```bash
kubectl logs <pod-name> --tail=100
```

→ 마지막 100줄만 출력

### ✅ 최근 X분 이내 로그만 보기

```bash
kubectl logs <pod-name> --since=5m
```

→ 5분 이내 로그만 출력 (m=minutes, s=seconds, h=hours)

---

## ✅ 7. 로그를 파일로 저장하기

```bash
kubectl logs <pod-name> > app-log.txt
```

→ 로컬 파일로 저장해서 grep, 분석, 첨부 가능

---

## ✅ 8. 로그 필터링 (grep, less 등 활용)

```bash
kubectl logs <pod-name> | grep ERROR
kubectl logs <pod-name> | less
```

→ 쉘 명령어와 조합해서 에러만 추출하거나 검색 가능

---

## ✅ 9. 로그 문제 해결 팁

| 현상 | 원인 및 해결 방법 |
|------|------------------|
| `no logs available` | 컨테이너가 실행되지 않음. `kubectl describe pod`로 상태 확인 |
| 로그가 안 보임 | 앱이 stdout/stderr이 아닌 파일로만 로그 출력하는 경우 |
| 로그가 짧게 끊김 | 컨테이너가 너무 빨리 종료됨. `--previous` 사용 |

---

## ✅ 10. 로그 확인 + Pod 상태 함께 보기

```bash
kubectl describe pod <pod-name>
```

- 로그 외에도 이벤트(Event), 상태(Status), 재시작 횟수 등을 종합적으로 볼 수 있음
- Pod의 문제점 추적에 매우 유용

---

## ✅ 예제 요약

```bash
# 단일 로그
kubectl logs mypod

# 멀티 컨테이너 로그
kubectl logs mypod -c app-container

# 이전 로그 (CrashLoopBackOff)
kubectl logs mypod --previous

# 실시간 로그
kubectl logs -f mypod

# 최근 50줄
kubectl logs mypod --tail=50

# 10분 이내 로그
kubectl logs mypod --since=10m

# 로그 저장
kubectl logs mypod > pod.log
```

---

## ✅ 결론

`kubectl logs`는 Kubernetes 환경에서 가장 많이 쓰이는 **문제 해결 도구 중 하나**입니다.  
Pod나 컨테이너의 상태를 빠르게 파악하고, 실시간으로 감시하거나,  
자동화 도구와 연계하여 로그를 추출할 때 매우 유용합니다.