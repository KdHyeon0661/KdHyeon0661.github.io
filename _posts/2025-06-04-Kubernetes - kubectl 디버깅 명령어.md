---
layout: post
title: Kubernetes - kubectl 디버깅 명령어
date: 2025-06-04 21:20:23 +0900
category: Kubernetes
---
# kubectl 디버깅 명령어 모음 (실전 진단용)

Kubernetes 클러스터에서 문제가 발생했을 때, 가장 먼저 사용하는 도구는 바로 `kubectl`입니다.  
이 글에서는 **Pod, Node, 리소스 상태, 이벤트, 로그 확인 등 디버깅에 꼭 필요한 kubectl 명령어**를 카테고리별로 정리했습니다.

---

## ✅ 기본 정보 조회

### 🔹 모든 리소스 간단히 보기

```bash
kubectl get all
```

- Pod, Service, Deployment, ReplicaSet 등 기본 리소스 확인

### 🔹 특정 리소스 확인

```bash
kubectl get pods -A                # 모든 namespace의 pod 보기
kubectl get deployment -n dev      # 특정 namespace에서만 보기
kubectl get svc                    # 현재 namespace의 service 보기
```

### 🔹 현재 context 및 namespace 확인

```bash
kubectl config current-context
kubectl config view --minify | grep namespace
```

---

## ✅ Pod 상태/이벤트 분석

### 🔹 Pod 상세 정보

```bash
kubectl describe pod <pod-name>
```

- **스케줄 실패, 이벤트 히스토리, Liveness/Readiness Probe 결과 등** 확인 가능

### 🔹 최근 이벤트 보기 (원인 추적 시 유용)

```bash
kubectl get events --sort-by='.lastTimestamp'
```

### 🔹 상태별 Pod 찾기

```bash
kubectl get pods --field-selector=status.phase=Pending
kubectl get pods | grep CrashLoopBackOff
```

---

## ✅ 로그 분석

### 🔹 단일 컨테이너 로그

```bash
kubectl logs <pod-name>
```

### 🔹 여러 컨테이너 중 특정 컨테이너 로그

```bash
kubectl logs <pod-name> -c <container-name>
```

### 🔹 이전 컨테이너 로그 (crash/restart 등 확인)

```bash
kubectl logs <pod-name> -c <container-name> --previous
```

---

## ✅ exec로 내부 디버깅

### 🔹 Pod 내부 쉘 진입 (bash/sh)

```bash
kubectl exec -it <pod-name> -- /bin/bash     # bash가 있는 경우
kubectl exec -it <pod-name> -- /bin/sh       # 간단한 알파인 기반
```

### 🔹 직접 명령 실행

```bash
kubectl exec <pod-name> -- cat /etc/hosts
kubectl exec <pod-name> -c <container-name> -- env
```

---

## ✅ 리소스 상태 확인

### 🔹 Node 상태 확인

```bash
kubectl get nodes
kubectl describe node <node-name>
```

- `NotReady`, `Taints`, `Allocatable`, `Capacity`, `Conditions` 확인 가능

### 🔹 리소스 사용량 확인 (metrics-server 필요)

```bash
kubectl top nodes
kubectl top pods
```

---

## ✅ 네트워크 디버깅

### 🔹 서비스 주소 확인

```bash
kubectl get svc
kubectl describe svc <service-name>
```

### 🔹 DNS 조회 테스트

```bash
kubectl exec -it <pod> -- nslookup <svc-name>.<namespace>.svc.cluster.local
```

### 🔹 포트 포워딩 (로컬 접근)

```bash
kubectl port-forward svc/<service-name> 8080:80
```

---

## ✅ yaml 확인 및 편집

### 🔹 리소스 YAML 출력

```bash
kubectl get pod <pod-name> -o yaml
```

### 🔹 실시간 수정 (편집기 실행)

```bash
kubectl edit deployment <deploy-name>
```

### 🔹 yaml로 저장 후 수정/재적용

```bash
kubectl get deployment myapp -o yaml > myapp.yaml
# 파일 수정 후
kubectl apply -f myapp.yaml
```

---

## ✅ 트러블슈팅 유틸리티

### 🔹 디버깅용 임시 Pod 실행 (alpine/busybox)

```bash
kubectl run debugbox --rm -it --image=busybox -- /bin/sh
```

→ DNS, curl, telnet 등 테스트용

---

## ✅ 특수 상황 진단

### 🔹 CrashLoopBackOff 원인 분석

```bash
kubectl describe pod <pod-name>
kubectl logs <pod-name> --previous
```

### 🔹 Pending 상태 분석

```bash
kubectl describe pod <pod-name>
kubectl get events --sort-by='.lastTimestamp'
```

### 🔹 Volume 마운트 실패 확인

```bash
kubectl describe pod <pod-name>
# PVC 상태도 확인
kubectl get pvc
```

---

## ✅ namespace 분리 확인

```bash
kubectl get pods --all-namespaces
kubectl get resourcequota -n <namespace>
```

---

## ✅ 정리: 상황별 디버깅 핵심 명령어 요약

| 목적 | 명령어 |
|------|--------|
| Pod 상태 확인 | `kubectl describe pod <pod>` |
| 로그 확인 | `kubectl logs <pod>`, `--previous` |
| 내부 명령 실행 | `kubectl exec <pod> -- <cmd>` |
| 최근 이벤트 | `kubectl get events --sort-by='.lastTimestamp'` |
| 노드 상태 | `kubectl get/describe nodes` |
| 리소스 사용량 | `kubectl top nodes/pods` |
| 네트워크 확인 | `nslookup`, `port-forward`, `describe svc` |
| 볼륨 이슈 | `kubectl get pvc`, `describe pod` |

---

## ✅ 추천: 함께 쓰면 좋은 도구

- [`k9s`](https://k9scli.io): 터미널 기반 쿠버네티스 대시보드
- [`stern`](https://github.com/stern/stern): 여러 Pod 로그 동시 확인
- [`kubectx`, `kubens`](https://github.com/ahmetb/kubectx): context, namespace 빠른 전환
- [`lens`](https://k8slens.dev): GUI 기반 클러스터 시각화 도구

---

## ✅ 마무리

kubectl만 잘 써도 대부분의 문제는 진단이 가능합니다.  
자주 쓰는 명령어는 **alias 등록**하거나 **스크립트화**해 두는 것이 좋습니다.