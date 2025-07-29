---
layout: post
title: Kubernetes - Volume, PersistentVolume, PersistentVolumeClaim(PVC)
date: 2025-04-11 19:20:23 +0900
category: Kubernetes
---
# Volume, PersistentVolume, PersistentVolumeClaim(PVC) 완전 정복

Kubernetes에서 컨테이너는 기본적으로 **비휘발성 저장소가 없습니다**.  
즉, Pod가 재시작되면 내부 데이터는 사라집니다.

이 문제를 해결하기 위해 Kubernetes는 다양한 **Volume** 개념을 제공합니다.  
그중 가장 널리 사용되는 방식이 바로 **PersistentVolume(PV)**과 **PersistentVolumeClaim(PVC)**입니다.

---

## ✅ 1. Volume이란?

Kubernetes의 `Volume`은 **Pod 안의 여러 컨테이너 간에 공유 가능한 디스크 공간**입니다.  
일반 Docker의 `volume`과 유사하나, **라이프사이클과 마운트 방식**에서 차이가 있습니다.

### 종류

| 종류 | 설명 |
|------|------|
| emptyDir | Pod 생성 시 빈 디렉토리 생성, Pod 종료 시 사라짐 |
| hostPath | 노드의 특정 경로를 마운트 |
| configMap/secret | 설정 파일을 마운트 |
| persistentVolumeClaim | 영구 저장소(PV)를 PVC로 연결 |

---

## ✅ 2. 문제점: ephemeral volume은 휘발성

예를 들어 `emptyDir`는 Pod가 재시작되면 데이터가 사라집니다.

```yaml
volumes:
- name: cache-volume
  emptyDir: {}
```

→ 따라서 **재시작/재스케줄링에도 살아남는 영구 저장소**가 필요합니다.

---

## ✅ 3. PersistentVolume(PV)란?

`PersistentVolume`은 Kubernetes 클러스터에서 **관리자가 사전에 생성한 스토리지 리소스**입니다.

- 클러스터 전체에 존재
- 다양한 backend 지원 (NFS, AWS EBS, Ceph, CSI 등)

```yaml
apiVersion: v1
kind: PersistentVolume
metadata:
  name: pv-example
spec:
  capacity:
    storage: 1Gi
  accessModes:
    - ReadWriteOnce
  hostPath:
    path: "/mnt/data"
  persistentVolumeReclaimPolicy: Retain
```

---

## ✅ 4. PersistentVolumeClaim(PVC)란?

`PersistentVolumeClaim`은 사용자가 **스토리지 사용을 요청하는 선언적 요청서**입니다.

```yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: pvc-example
spec:
  accessModes:
    - ReadWriteOnce
  resources:
    requests:
      storage: 500Mi
```

→ 이 PVC를 Pod에서 마운트할 수 있습니다.

---

## ✅ 5. PV + PVC + Pod 관계

```yaml
Pod ─────> PVC ─────> PV ─────> 실제 스토리지
```

- Pod는 PVC를 참조
- PVC는 조건에 맞는 PV를 **바인딩**
- PV는 실제 디스크(호스트, NFS, EBS 등)로 연결됨

---

## ✅ 6. Pod에 PVC 사용하기

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: pod-with-pvc
spec:
  containers:
  - name: app
    image: nginx
    volumeMounts:
    - mountPath: "/usr/share/nginx/html"
      name: web-data
  volumes:
  - name: web-data
    persistentVolumeClaim:
      claimName: pvc-example
```

---

## ✅ 7. Access Modes

| 모드 | 설명 |
|------|------|
| ReadWriteOnce (RWO) | 하나의 노드에서 읽기/쓰기 가능 (대부분의 경우 기본값) |
| ReadOnlyMany (ROX) | 여러 노드에서 읽기 전용 가능 |
| ReadWriteMany (RWX) | 여러 노드에서 동시에 읽기/쓰기 가능 (NFS, GlusterFS 등) |

---

## ✅ 8. Reclaim Policy

PV가 **삭제된 PVC에 의해 해제될 때 처리 방식**:

| 정책 | 설명 |
|------|------|
| Retain | PV와 데이터 유지 (수동 삭제 필요) |
| Delete | PVC 삭제 시 PV도 함께 삭제 |
| Recycle (Deprecated) | 간단한 포맷 후 재사용 가능 (비추천) |

---

## ✅ 9. 동적 프로비저닝 (StorageClass)

수동으로 PV를 만드는 대신, PVC를 생성하면 **자동으로 PV를 생성**할 수 있게 하는 기능입니다.

### StorageClass 정의

```yaml
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: fast
provisioner: kubernetes.io/aws-ebs  # EBS 사용 예시
parameters:
  type: gp2
```

### PVC와 연결

```yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: dynamic-pvc
spec:
  storageClassName: fast
  accessModes:
    - ReadWriteOnce
  resources:
    requests:
      storage: 1Gi
```

---

## ✅ 10. 볼륨 상태 확인 명령어

```bash
# PVC 상태 확인
kubectl get pvc

# PV 상태 확인
kubectl get pv

# PVC와 바인딩 상태 보기
kubectl describe pvc pvc-example

# 동적 생성된 PV 확인
kubectl get pv | grep dynamic-pvc
```

---

## ✅ 예제: 간단한 Nginx + PV 배포

```yaml
apiVersion: v1
kind: PersistentVolume
metadata:
  name: my-pv
spec:
  capacity:
    storage: 1Gi
  accessModes:
    - ReadWriteOnce
  hostPath:
    path: "/mnt/nginx-data"
---
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: my-pvc
spec:
  accessModes:
    - ReadWriteOnce
  resources:
    requests:
      storage: 1Gi
---
apiVersion: v1
kind: Pod
metadata:
  name: nginx-pv
spec:
  containers:
  - name: nginx
    image: nginx
    volumeMounts:
    - mountPath: "/usr/share/nginx/html"
      name: html
  volumes:
  - name: html
    persistentVolumeClaim:
      claimName: my-pvc
```

---

## ✅ 결론

| 구성요소 | 설명 |
|----------|------|
| PV (PersistentVolume) | 관리자/시스템이 정의한 저장소 |
| PVC (PersistentVolumeClaim) | 사용자의 요청 |
| Pod | PVC를 통해 스토리지를 사용 |
| StorageClass | 동적 PV 생성을 위한 설정 |
| AccessMode | 읽기/쓰기 동시 가능 여부 |
| ReclaimPolicy | PVC 삭제 시 PV 처리 방법 |

> Kubernetes에서 **지속 가능한 데이터 저장**을 위해 PV/PVC는 필수 개념입니다.  
> 클러스터 운영 안정성을 위해 StorageClass와 ReclaimPolicy도 함께 고려해야 합니다.

---

## ✅ 참고 자료

- [Kubernetes 공식 문서: Volumes](https://kubernetes.io/docs/concepts/storage/volumes/)
- [PersistentVolume](https://kubernetes.io/docs/concepts/storage/persistent-volumes/)
- [StorageClass 동적 프로비저닝](https://kubernetes.io/docs/concepts/storage/storage-classes/)