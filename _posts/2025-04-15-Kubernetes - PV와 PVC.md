---
layout: post
title: Docker - PV와 PVC
date: 2025-04-15 20:20:23 +0900
category: Docker
---
# 🧊 Kubernetes에서의 볼륨 이해하기: PV와 PVC

---

## 📦 1. 왜 Kubernetes에서도 Volume이 필요할까?

Kubernetes의 Pod는 컨테이너와 마찬가지로 **일시적(Ephemeral)**입니다.  
Pod가 재시작되면 내부 데이터는 모두 **사라집니다.**

### ✅ 해결 방법
> Kubernetes는 Docker Volume보다 한 단계 발전한 **PersistentVolume(PV)** 및 **PersistentVolumeClaim(PVC)** 개념을 제공합니다.

---

## 🔍 2. Volume이란?

Kubernetes에서의 Volume은 다음 두 종류로 나뉩니다:

| 종류 | 설명 |
|------|------|
| **Ephemeral Volume** | Pod가 삭제되면 함께 삭제됨 (emptyDir, configMap 등) |
| **Persistent Volume (PV)** | 클러스터에 독립적으로 존재하며, Pod와는 별개로 **영속성 유지** |

---

## 🧱 3. PV / PVC 구조 설명

Kubernetes에서는 **영속 볼륨을 추상화**하여 다음 구조로 사용합니다:

```
+----------------+     +---------------------+
|  PersistentVolume(PV) | ← 실제 스토리지 (디스크, NFS 등)
+----------------+     +---------------------+
          ↑
   바인딩(binding)
          ↓
+----------------+
| PersistentVolumeClaim (PVC) | ← Pod에서 요청하는 인터페이스
+----------------+
          ↑
     Pod가 마운트
          ↓
+----------------+
|      Pod       |
+----------------+
```

- **PV (공급자)**: 클러스터 관리자가 정의한 실제 스토리지
- **PVC (요청자)**: 사용자가 스토리지를 필요로 할 때 요청
- **Pod**: PVC를 통해 간접적으로 PV에 접근

---

## 📁 4. Ephemeral vs Persistent 볼륨

| 항목 | Ephemeral (emptyDir 등) | Persistent (PV/PVC) |
|------|-------------------------|----------------------|
| 수명 | Pod 수명과 같음 | Pod 수명과 무관 |
| 데이터 유지 | 재시작 시 삭제됨 | 재시작 후에도 유지 |
| 사용 예시 | 임시 캐시, 세션 | DB, 업로드 파일 등 |
| 선언 위치 | Pod spec 내부 | PV/PVC 객체로 별도 정의 |

---

## 📦 5. 실전 예제: PV + PVC + Pod

### ✅ 1. PersistentVolume 정의 (NFS 예시)

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
  persistentVolumeReclaimPolicy: Retain
  nfs:
    server: 10.0.0.1
    path: "/exports/data"
```

- `capacity`: 스토리지 용량
- `accessModes`:
  - `ReadWriteOnce`: 하나의 Pod에서 읽기/쓰기 가능
  - `ReadOnlyMany`: 여러 Pod에서 읽기 가능
  - `ReadWriteMany`: 여러 Pod에서 읽기/쓰기 가능
- `reclaimPolicy`:
  - `Retain`: PVC 삭제 후에도 PV는 보존
  - `Delete`: PVC 삭제 시 PV도 삭제

---

### ✅ 2. PersistentVolumeClaim 정의

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
      storage: 1Gi
```

- PVC는 `PV를 요청하는 사용자 인터페이스`입니다.
- `spec.accessModes`와 `storage` 요청량이 PV와 매칭되어야 바인딩됩니다.

---

### ✅ 3. Pod에서 PVC 마운트

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: pod-using-pvc
spec:
  containers:
    - name: app
      image: nginx
      volumeMounts:
        - mountPath: "/usr/share/nginx/html"
          name: myvolume
  volumes:
    - name: myvolume
      persistentVolumeClaim:
        claimName: pvc-example
```

- 이 Pod는 PVC를 통해 PV를 마운트하여 `/usr/share/nginx/html`에 접근
- 컨테이너가 삭제되어도 데이터는 유지됨

---

## 🔁 6. 동적 할당 (Dynamic Provisioning)

> PVC가 요청되면 Kubernetes가 자동으로 PV를 생성해주는 기능

```yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: pvc-dynamic
spec:
  storageClassName: standard
  accessModes:
    - ReadWriteOnce
  resources:
    requests:
      storage: 5Gi
```

- `storageClassName`: 동적으로 PV를 생성할 때 사용할 **스토리지 클래스**
- GKE, EKS, AKS 등의 클라우드 환경에서는 이 방식이 일반적

---

## 📋 7. 실전 명령어

```bash
# 볼륨 보기
kubectl get pv
kubectl get pvc

# 볼륨 상세 정보
kubectl describe pv <name>
kubectl describe pvc <name>

# 삭제
kubectl delete pvc <name>
kubectl delete pv <name>
```

---

## 🧩 8. 베스트 프랙티스

| 항목 | 권장 사항 |
|------|-----------|
| ✅ 스토리지 분리 | 코드와 데이터는 철저히 분리 |
| 📦 PVC 사용 | Pod에서 직접 PV 사용 ❌ PVC 통해 사용 ✅ |
| 🔁 Reclaim 정책 설정 | 필요시 Retain, 자동 정리면 Delete |
| ☁ 클라우드 스토리지 | AWS EBS, GCP PD, Azure Disk 등과 연동 |
| 🔒 보안 고려 | 권한 분리된 사용자별 볼륨 사용 고려 |

---

## 📚 참고 자료

- [Kubernetes Volumes 공식 문서](https://kubernetes.io/docs/concepts/storage/volumes/)
- [Persistent Volumes 설명](https://kubernetes.io/docs/concepts/storage/persistent-volumes/)
- [StorageClass 개념](https://kubernetes.io/docs/concepts/storage/storage-classes/)
- [AccessModes](https://kubernetes.io/docs/concepts/storage/persistent-volumes/#access-modes)
