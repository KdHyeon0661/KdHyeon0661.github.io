---
layout: post
title: Docker - PV와 PVC
date: 2025-04-15 20:20:23 +0900
category: Docker
---
# Kubernetes에서의 볼륨 이해하기: PV와 PVC

## 왜 쿠버네티스에서 영구 볼륨이 필요한가?

쿠버네티스 Pod는 기본적으로 일시적입니다. 컨테이너가 재시작되거나, Pod가 다른 노드로 재스케줄링되면, 컨테이너 내부 파일 시스템에 저장된 모든 데이터는 사라집니다. 그러나 데이터베이스 파일, 사용자 업로드 콘텐츠, 애플리케이션 로그 등 많은 데이터는 **Pod의 생명주기와 무관하게 보존되어야 합니다.**

이러한 문제를 해결하기 위해 쿠버네티스는 **볼륨(Volume)** 시스템을 제공합니다. 특히 **PersistentVolume (PV)** 과 **PersistentVolumeClaim (PVC)** 는 영구적인 데이터 저장소를 안전하게 관리하는 데 핵심적인 역할을 합니다.

---

## 핵심 개념: PV, PVC, StorageClass

### PersistentVolume (PV)
- **클러스터 관리자**가 프로비저닝하는 **실제 저장소 리소스**의 추상화입니다.
- NFS 공유, 클라우드 디스크(예: AWS EBS, Google PD), 로컬 스토리지 등 다양한 백엔드 저장소를 나타냅니다.
- PV는 용량, 접근 모드, 재활용 정책과 같은 속성을 정의합니다.

### PersistentVolumeClaim (PVC)
- **애플리케이션 개발자/사용자**가 PV로부터 요청하는 **스토리지 청구서**입니다.
- PVC는 필요로 하는 스토리지의 용량과 접근 방식을 지정합니다.
- 쿠버네티스는 PVC와 일치하는 PV를 찾아 바인딩하고, Pod는 이 PVC를 마운트하여 사용합니다.

### StorageClass (SC)
- **동적 프로비저닝**을 가능하게 합니다. 사용자가 PVC를 생성하면, StorageClass에 정의된 프로비저너가 자동으로 적절한 PV를 생성합니다.
- 클라우드 환경에서 가장 일반적으로 사용되며, 스토리지 유형(예: SSD, HDD), 성능 계층, 리클레임 정책 등을 정의합니다.

**동작 흐름 요약:**
```
애플리케이션(Pod) -> PVC (스토리지 요청) -> PV (실제 스토리지) -> 외부 저장소(디스크/NFS 등)
```

---

## Ephemeral 볼륨 vs Persistent 볼륨

쿠버네티스 볼륨은 크게 두 가지 범주로 나눌 수 있습니다:

| 항목 | Ephemeral (일시적) 볼륨 | Persistent (영구) 볼륨 |
|------|------------------------|------------------------|
| **데이터 수명** | Pod와 함께 생성되고 삭제됨 | Pod와 독립적이며 삭제 후에도 유지됨 |
| **주요 사용 사례** | 임시 캐시, 빌드 중간 결과, 설정 파일 주입 | 데이터베이스 데이터, 사용자 업로드 파일, 장기 보관 로그 |
| **대표 유형** | `emptyDir`, `configMap`, `secret` | `PersistentVolume` / `PersistentVolumeClaim` |
| **선언 방식** | Pod 명세 내부에 직접 정의 | 별도의 PV/PVC 오브젝트로 관리 |

> **`emptyDir` 주의사항**: 노드의 로컬 디스크(또는 메모리)를 사용하므로, 노드 장애 시 데이터가 유실될 수 있습니다. **`hostPath`는 개발/테스트 환경에서만 제한적으로 사용해야 하며**, 보안 문제와 노드 간 이식성 문제가 있습니다.

---

## PersistentVolume의 핵심 속성 이해하기

### 1. 접근 모드 (Access Modes)
볼륨을 어떻게 마운트할 수 있는지를 정의합니다.
- **`ReadWriteOnce` (RWO)**: **하나의 노드**에서 **하나의 Pod**만 읽기/쓰기 모드로 마운트할 수 있습니다. 대부분의 클라우드 블록 스토리지(예: AWS EBS, Azure Disk)가 이 모드를 지원합니다.
- **`ReadOnlyMany` (ROX)**: **여러 노드**에서 **여러 Pod**가 읽기 전용 모드로 마운트할 수 있습니다.
- **`ReadWriteMany` (RWX)**: **여러 노드**에서 **여러 Pod**가 읽기/쓰기 모드로 마운트할 수 있습니다. NFS, CephFS, 클라우드 파일 스토리지(예: AWS EFS, Azure Files)가 지원합니다.
- **`ReadWriteOncePod` (RWO-Pod)**: **하나의 Pod**만 읽기/쓰기 모드로 마운트할 수 있습니다. (쿠버네티스 1.22+, CSI 드라이버 지원 필요)

### 2. 리클레임 정책 (Reclaim Policy)
PVC가 삭제된 후 PV와 그 아래의 실제 저장소를 어떻게 처리할지 결정합니다.
- **`Retain`**: PVC 삭제 후에도 PV와 실제 데이터를 유지합니다. 관리자가 수동으로 정리해야 합니다. 중요한 데이터를 다루는 환경에 적합합니다.
- **`Delete`**: PVC 삭제 시 PV와 연관된 외부 저장소(예: 클라우드 디스크)도 자동으로 삭제합니다. 개발/테스트 환경에 유용합니다.
- **`Recycle`**: (사용 중지됨) 더 이상 권장되지 않습니다.

### 3. 노드 어피니티 및 바인딩 모드
- **노드 어피니티**: 특정 노드에 연결된 로컬 스토리지를 사용할 때 PV가 특정 노드에 배치되도록 제약을 걸 수 있습니다.
- **바인딩 모드** (StorageClass의 `volumeBindingMode`):
    - **`Immediate`**: PVC 생성 즉시 바인딩을 시도합니다. 사용 가능한 PV가 없으면 실패합니다.
    - **`WaitForFirstConsumer`**: PVC를 사용하는 **첫 번째 Pod가 스케줄된 후**에 해당 노드의 가용 영역(AZ)이나 토폴로지를 고려하여 PV를 프로비저닝하고 바인딩합니다. 멀티 가용 영역 클러스터에서 필수적인 설정입니다.

---

## 실전 예제

### 예제 1: 수동 PV와 PVC 생성 (NFS 백엔드)

**1. PersistentVolume 생성**
```yaml
apiVersion: v1
kind: PersistentVolume
metadata:
  name: manual-nfs-pv
spec:
  capacity:
    storage: 10Gi
  accessModes:
    - ReadWriteMany  # NFS는 여러 Pod에서 공유 가능
  persistentVolumeReclaimPolicy: Retain
  nfs:
    server: 10.0.0.100  # NFS 서버 IP
    path: "/exports/app-data"
```

**2. PersistentVolumeClaim 생성**
```yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: app-data-pvc
spec:
  accessModes:
    - ReadWriteMany
  resources:
    requests:
      storage: 10Gi
  # storageClassName을 지정하지 않으면, accessModes와 용량이 일치하는 PV를 찾음
```

**3. Pod에서 PVC 사용**
```yaml
apiVersion: v1
kind: Pod
metadata:
  name: app-pod
spec:
  containers:
  - name: app
    image: nginx:alpine
    volumeMounts:
    - name: data-store
      mountPath: /usr/share/nginx/html
  volumes:
  - name: data-store
    persistentVolumeClaim:
      claimName: app-data-pvc
```

### 예제 2: 동적 프로비저닝 (StorageClass 사용)

**1. StorageClass 정의 (AWS EBS gp3 예시)**
```yaml
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: gp3-encrypted
provisioner: ebs.csi.aws.com
parameters:
  type: gp3
  encrypted: "true"
reclaimPolicy: Delete
allowVolumeExpansion: true  # 볼륨 확장 허용
volumeBindingMode: WaitForFirstConsumer  # 멀티 AZ 환경에 필수
```

**2. PVC 생성 (StorageClass 참조)**
```yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: dynamic-pvc
spec:
  storageClassName: gp3-encrypted  # 위에서 정의한 StorageClass
  accessModes:
    - ReadWriteOnce
  resources:
    requests:
      storage: 20Gi
```
이 PVC를 생성하는 순간, 지정된 StorageClass의 프로비저너가 AWS에서 20GB gp3 EBS 볼륨을 자동으로 생성하고 PV로 바인딩합니다.

---

## 권한 및 보안 구성

비루트 사용자로 실행되는 컨테이너가 마운트된 볼륨에 쓰기 권한을 가지려면 Pod의 `securityContext`를 올바르게 설정해야 합니다.

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: secured-app
spec:
  securityContext:
    fsGroup: 1000  # 마운트된 볼륨의 파일 그룹 소유권을 1000으로 설정
  containers:
  - name: app
    image: myapp:nonroot
    securityContext:
      runAsUser: 1000  # 컨테이너를 UID 1000 사용자로 실행
    volumeMounts:
    - name: data
      mountPath: /data
  volumes:
  - name: data
    persistentVolumeClaim:
      claimName: dynamic-pvc
```
- `fsGroup`: Pod의 모든 컨테이너에서 추가적인 그룹 ID를 설정하며, 마운트된 볼륨의 파일/디렉토리에 이 그룹 권한을 부여합니다.
- SELinux가 활성화된 환경에서는 `seLinuxOptions` 설정도 필요할 수 있습니다.

---

## StatefulSet과 Volume

데이터베이스나 메시지 큐와 같이 상태를 유지해야 하고, 각 인스턴스가 고유한 스토리지를 가져야 하는 애플리케이션에는 **StatefulSet**을 사용합니다. StatefulSet은 `volumeClaimTemplates`를 사용하여 각 Pod에 대해 고유한 PVC를 자동 생성합니다.

```yaml
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: postgres
spec:
  serviceName: postgres
  replicas: 3
  selector:
    matchLabels:
      app: postgres
  template:
    metadata:
      labels:
        app: postgres
    spec:
      containers:
      - name: postgres
        image: postgres:15
        volumeMounts:
        - name: data
          mountPath: /var/lib/postgresql/data
  volumeClaimTemplates:  # 각 Pod마다 이 템플릿으로 PVC 생성
  - metadata:
      name: data
    spec:
      accessModes: ["ReadWriteOnce"]
      storageClassName: "gp3-encrypted"
      resources:
        requests:
          storage: 10Gi
```
위 설정으로 생성된 Pod(`postgres-0`, `postgres-1`, `postgres-2`)는 각각 `data-postgres-0`, `data-postgres-1`, `data-postgres-2`라는 독립된 PVC와 PV를 갖게 됩니다.

---

## 볼륨 확장 및 스냅샷

### 볼륨 확장
많은 스토리지 클래스와 CSI 드라이버는 동적 볼륨 확장을 지원합니다. PVC의 `spec.resources.requests.storage` 필드를 업데이트하면 확장이 시도됩니다.

```bash
kubectl patch pvc dynamic-pvc -p '{"spec":{"resources":{"requests":{"storage":"30Gi"}}}}'
```
확장 성공 여부는 스토리지 백엔드와 파일시스템(예: ext4, xfs)의 지원에 달려 있습니다. 일부 경우 Pod 재시작이 필요할 수 있습니다.

### 볼륨 스냅샷 (Volume Snapshots)
CSI 스냅샷 기능이 활성화된 클러스터에서는 특정 시점의 볼륨 상태를 보존하고 복원할 수 있습니다.

```yaml
# VolumeSnapshotClass 정의
apiVersion: snapshot.storage.k8s.io/v1
kind: VolumeSnapshotClass
metadata:
  name: csi-aws-snapclass
driver: ebs.csi.aws.com
deletionPolicy: Delete

# 특정 PVC의 스냅샷 생성
apiVersion: snapshot.storage.k8s.io/v1
kind: VolumeSnapshot
metadata:
  name: db-backup-snapshot
spec:
  volumeSnapshotClassName: csi-aws-snapclass
  source:
    persistentVolumeClaimName: dynamic-pvc  # 백업할 PVC 이름
```

---

## 문제 해결 가이드

| 증상 | 확인 사항 | 일반적인 원인 및 해결 방법 |
|------|-----------|---------------------------|
| **PVC가 `Pending` 상태** | `kubectl describe pvc <pvc-name>`의 이벤트 | 일치하는 PV가 없거나, StorageClass가 없거나, 접근 모드가 맞지 않음. StorageClass를 명시하거나 PV를 생성하세요. |
| **Pod가 `ContainerCreating` 상태에서 멈춤** | `kubectl describe pod <pod-name>`의 이벤트 (Mount/Attach 관련) | CSI 드라이버 문제, 네트워크 문제, IAM/권한 오류. 드라이버 Pod 로그를 확인하세요. |
| **파일 시스템 권한 오류** | 컨테이너 로그, `kubectl exec`로 `id` 및 `ls -la` 확인 | `runAsUser`, `fsGroup` 설정 불일치. Pod의 `securityContext`를 조정하세요. |
| **여러 Pod에서 RWX 볼륨 공유 필요** | PVC의 `accessModes` 및 StorageClass/드라이버 지원 확인 | 블록 스토리지(예: EBS)는 RWO만 지원. NFS, EFS, Azure Files 등 파일 스토리지로 전환하세요. |
| **볼륨 확장 실패** | PVC 이벤트, StorageClass의 `allowVolumeExpansion` 설정 | 드라이버가 확장을 지원하지 않거나, `allowVolumeExpansion: true`가 아닐 수 있습니다. |

---

## 결론

쿠버네티스의 영구 스토리지 시스템(PV/PVC/StorageClass)은 상태를 유지하는 애플리케이션을 안정적으로 운영하기 위한 핵심 인프라입니다. 효과적으로 사용하기 위해서는 다음 사항을 명확히 이해하고 설계에 반영해야 합니다:

1.  **애플리케이션 요구사항 분석**: 데이터에 대한 접근 패턴(`ReadWriteOnce` vs `ReadWriteMany`), 필요한 성능(IOPS, 처리량), 그리고 데이터 보존 정책을 먼저 정의하세요.
2.  **적절한 스토리지 솔루션 선택**: 요구사항에 따라 클라우드 블록 스토리지, 파일 스토리지, 또는 로컬 스토리지 중 적합한 백엔드를 선택하세요.
3.  **동적 프로비저닝 활용**: 운영 효율성을 위해 StorageClass를 통한 동적 프로비저닝을 기본으로 사용하세요. 특히 멀티 가용 영역 환경에서는 `volumeBindingMode: WaitForFirstConsumer`를 꼭 설정하세요.
4.  **보안 및 권한 고려**: 컨테이너의 실행 사용자와 볼륨의 파일 권한을 `securityContext`를 통해 조정하여 "Permission denied" 오류를 예방하세요.
5.  **생명주기 관리 전략 수립**: 데이터의 중요도에 따라 `ReclaimPolicy` (`Retain` 또는 `Delete`)를 결정하고, 필요하다면 VolumeSnapshot을 통한 백업/복구 절차를 마련하세요.

영구 스토리지를 올바르게 구성한다면, 쿠버네티스의 탄력성과 자가 치유 기능을 데이터 기반 애플리케이션에도 그대로 적용할 수 있게 되어, 진정한 클라우드 네이티브 아키텍처를 구현하는 데 큰 도움이 될 것입니다.