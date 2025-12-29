---
layout: post
title: Kubernetes - Volume, PersistentVolume, PersistentVolumeClaim
date: 2025-05-11 19:20:23 +0900
category: Kubernetes
---
# Kubernetes 스토리지: Volume, PersistentVolume, PersistentVolumeClaim 이해하기

## 저장소 아키텍처 개요

Kubernetes의 스토리지 시스템은 애플리케이션 데이터의 수명주기와 접근성을 관리하는 다층적 아키텍처를 제공합니다.

```
[Pod]
  └─ volumeMounts -> [Volume 스펙]
                        └─ (1) 임시 볼륨(예: emptyDir, configMap, secret, projected)
                        └─ (2) 영구 볼륨 -> [PVC] -> [PV] -> [물리/클라우드 디스크(CSI)]
```

이 구조에서 볼륨은 두 가지 주요 범주로 구분됩니다:
- **임시 볼륨**: Pod의 생명주기와 함께 생성되고 소멸됩니다. 캐시 데이터나 임시 파일 저장에 적합합니다.
- **영구 볼륨**: Pod 생명주기와 독립적으로 존재하며, 데이터베이스나 사용자 업로드 파일과 같은 지속성이 필요한 데이터를 저장합니다.

---

## 볼륨 유형과 활용 시나리오

### 주요 볼륨 타입 비교

| 타입 | 특징 | 일반적인 사용 사례 |
|---|---|---|
| `emptyDir` | Pod 생성 시 생성, 삭제 시 제거. 노드 로컬 저장소 사용 | 컨테이너 간 공유 캐시, 임시 작업 공간 |
| `hostPath` | 호스트 노드의 파일시스템 경로를 직접 마운트 | 시스템 모니터링 도구, 노드 레벨 로그 수집 |
| `configMap`/`secret` | 설정 데이터와 비밀 정보를 파일 또는 환경변수로 주입 | 애플리케이션 설정, TLS 인증서, API 키 |
| `downwardAPI` | Pod 메타데이터를 파일로 노출 | 빌드 정보, 라벨, 어노테이션 참조 |
| `persistentVolumeClaim` | 영구 데이터 저장을 위한 표준 인터페이스 | 데이터베이스 스토리지, 사용자 업로드 파일 |
| `projected` | 여러 볼륨 소스를 단일 디렉토리로 통합 | 복합적인 설정과 비밀 정보 관리 |

**핵심 원칙**: 애플리케이션 데이터는 Pod 재시작이나 스케일링 후에도 유지되어야 하므로 `persistentVolumeClaim`을 통해 관리해야 합니다.

---

## PersistentVolume과 PersistentVolumeClaim 개념

### 개념적 이해
- **PersistentVolume (PV)**: 클러스터 관리자가 프로비저닝하는 실제 스토리지 리소스입니다. AWS EBS, Azure Disk, GCP Persistent Disk, NFS, Ceph 등의 백엔드 스토리지를 추상화합니다.
- **PersistentVolumeClaim (PVC)**: 개발자나 애플리케이션 배포자가 필요한 스토리지의 특성(크기, 접근 모드, 성능)을 정의하는 "요청서"입니다.

이 관계를 도식화하면 다음과 같습니다:
```
Pod ──(volumeMounts)──> PVC ──(바인딩)──> PV ──> 물리 스토리지/CSI 드라이버
```

### 기본 예제

**PV 생성 예시 (수동 프로비저닝)**:
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
  volumeMode: Filesystem
  hostPath:
    path: "/mnt/data"
```

**PVC 생성 예시**:
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

**Pod에서 PVC 사용 예시**:
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
      - name: web-data
        mountPath: /usr/share/nginx/html
  volumes:
  - name: web-data
    persistentVolumeClaim:
      claimName: pvc-example
```

---

## 주요 구성 요소 상세 설명

### 접근 모드 (Access Modes)
접근 모드는 볼륨이 어떻게 마운트되고 사용될 수 있는지를 정의합니다.

| 모드 | 설명 | 일반적인 사용 사례 |
|---|---|---|
| `ReadWriteOnce` (RWO) | 단일 노드에서 읽기/쓰기 가능 | 블록 스토리지(EBS, Azure Disk) |
| `ReadOnlyMany` (ROX) | 여러 노드에서 읽기 전용 접근 | 공유 설정 파일, 참조 데이터 |
| `ReadWriteMany` (RWX) | 여러 노드에서 읽기/쓰기 가능 | 공유 파일 시스템(NFS, EFS) |

**중요 참고사항**: 대부분의 클라우드 블록 스토리지는 RWO만 지원합니다. 여러 Pod가 동일한 데이터에 동시에 쓰기 접근이 필요한 경우 NFS, CephFS, 클라우드 파일 서비스(EFS, Azure Files)와 같은 RWX 지원 스토리지를 고려해야 합니다.

### 볼륨 모드 (VolumeMode)
- **Filesystem**: 표준 파일시스템으로 마운트되며 대부분의 워크로드에 적합합니다.
- **Block**: 원시 블록 장치로 노출되며 고성능 데이터베이스나 특수 스토리지 최적화가 필요한 경우 사용됩니다.

### 재사용 정책 (Reclaim Policy)
볼륨의 재사용 정책은 PVC가 삭제된 후 PV와 백엔드 스토리지를 어떻게 처리할지 결정합니다.

| 정책 | 동작 | 사용 시나리오 |
|---|---|---|
| `Retain` | PVC 삭제 후에도 PV와 데이터 보관 | 프로덕션 데이터, 감사 요구사항이 있는 경우 |
| `Delete` | PVC 삭제 시 PV와 백엔드 스토리지 자동 삭제 | 개발/테스트 환경, 일시적 데이터 |
| `Recycle` | 더 이상 사용되지 않음(폐기됨) | 레거시 시스템 호환성 |

운영 환경에서는 중요한 데이터의 경우 `Retain` 정책을 사용하여 실수로 인한 데이터 손실을 방지하는 것이 좋습니다.

---

## 동적 프로비저닝과 StorageClass

### StorageClass 개념
수동으로 PV를 생성하는 대신 StorageClass를 정의하면 PVC가 생성될 때 자동으로 적절한 PV가 프로비저닝됩니다.

```yaml
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: fast
provisioner: ebs.csi.aws.com  # CSI 드라이버 지정
parameters:
  type: gp3
  fsType: ext4
  iops: "3000"
  throughput: "125"
reclaimPolicy: Delete
allowVolumeExpansion: true
volumeBindingMode: WaitForFirstConsumer
```

**주요 매개변수**:
- **provisioner**: 스토리지 프로비저닝을 담당하는 CSI 드라이버
- **parameters**: 스토리지별 구성 옵션(디스크 유형, 성능 특성 등)
- **allowVolumeExpansion**: PVC 생성 후 용량 확장 허용 여부
- **volumeBindingMode**: `WaitForFirstConsumer`는 Pod가 스케줄된 후 볼륨을 생성하여 가용 영역 불일치 문제를 방지

### 동적 프로비저닝을 사용한 PVC
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
      storage: 10Gi
```

이 PVC가 생성되면 지정된 StorageClass에 따라 자동으로 PV가 생성되고 바인딩됩니다.

---

## Stateful 워크로드를 위한 StatefulSet 패턴

데이터베이스와 같은 상태 유지 애플리케이션의 경우, 각 Pod 인스턴스가 고유한 영구 스토리지를 필요로 합니다. StatefulSet은 이를 위한 이상적인 패턴을 제공합니다.

```yaml
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: web
spec:
  serviceName: "web"
  replicas: 3
  selector:
    matchLabels:
      app: web
  template:
    metadata:
      labels:
        app: web
    spec:
      containers:
      - name: nginx
        image: nginx
        volumeMounts:
        - name: www
          mountPath: /usr/share/nginx/html
  volumeClaimTemplates:
  - metadata:
      name: www
    spec:
      accessModes:
        - ReadWriteOnce
      storageClassName: fast
      resources:
        requests:
          storage: 5Gi
```

이 구성의 결과:
- 각 Pod(`web-0`, `web-1`, `web-2`)에 대해 고유한 PVC(`www-web-0`, `www-web-1`, `www-web-2`)가 자동 생성
- Pod가 재시작되거나 다른 노드로 재스케줄되더라도 동일한 스토리지에 연결
- 삭제 순서는 생성 순서의 역순으로 진행되어 데이터 무결성 보장

---

## 데이터 초기화 및 구성 패턴

### InitContainer를 이용한 초기 데이터 설정
애플리케이션 컨테이너가 시작되기 전에 볼륨을 초기화할 수 있습니다.

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: app-with-init
spec:
  initContainers:
  - name: init-content
    image: busybox
    command: ["sh", "-c", "echo 'Initial content' > /workdir/index.html"]
    volumeMounts:
    - name: data
      mountPath: /workdir
  containers:
  - name: nginx
    image: nginx
    volumeMounts:
    - name: data
      mountPath: /usr/share/nginx/html
  volumes:
  - name: data
    persistentVolumeClaim:
      claimName: my-pvc
```

### 서브패스를 활용한 디렉토리 분리
단일 PVC를 여러 디렉토리로 분할하여 마운트할 수 있습니다.

```yaml
volumeMounts:
- name: shared-data
  mountPath: /app/uploads
  subPath: uploads
```

**주의사항**: `subPath`는 마운트 시점에만 평가되며 동적 경로가 필요한 경우 `subPathExpr`을 사용할 수 있습니다. 일부 스토리지 드라이버와 커널 버전에서는 권한 문제가 발생할 수 있으므로 테스트가 필요합니다.

---

## 보안 및 권한 관리

### 파일 시스템 권한 설정
Pod와 컨테이너 수준에서 보안 컨텍스트를 정의하여 파일 시스템 권한을 관리할 수 있습니다.

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: secure-app
spec:
  securityContext:
    fsGroup: 2000
    fsGroupChangePolicy: "OnRootMismatch"
  containers:
  - name: app
    image: nginx
    securityContext:
      runAsUser: 1000
      runAsGroup: 1000
      readOnlyRootFilesystem: true
    volumeMounts:
    - name: data
      mountPath: /data
  volumes:
  - name: data
    persistentVolumeClaim:
      claimName: secure-pvc
```

**설명**:
- `fsGroup`: 마운트된 볼륨의 파일 그룹 소유권 설정
- `fsGroupChangePolicy`: "OnRootMismatch"는 루트 소유 파일만 변경하여 성능 최적화
- `runAsUser`/`runAsGroup`: 컨테이너 프로세스 실행 사용자/그룹 지정
- `readOnlyRootFilesystem`: 루트 파일시스템을 읽기 전용으로 설정하여 보안 강화

### SELinux 지원
SELinux를 사용하는 배포판의 경우 적절한 컨텍스트를 지정해야 할 수 있습니다.

```yaml
securityContext:
  seLinuxOptions:
    type: "container_file_t"
```

---

## 성능 최적화 기법

### 마운트 옵션 튜닝
StorageClass나 PV에 성능 향상을 위한 마운트 옵션을 지정할 수 있습니다.

```yaml
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: optimized
provisioner: ebs.csi.aws.com
parameters:
  type: gp3
  fsType: xfs
mountOptions:
  - noatime
  - nodiratime
  - nobarrier
```

**효과적인 옵션**:
- **noatime/nodiratime**: 파일 접근 시간 갱신 비활성화로 메타데이터 I/O 감소
- **xfs 파일시스템**: 대용량 파일 및 병렬 I/O 작업에 유리
- **nobarrier**: 저널링 파일시스템에서 배리어 비활성화(전원 장애 시 데이터 손실 위험)

### 볼륨 확장
생성된 PVC의 용량을 확장할 수 있습니다.

```yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: expandable-pvc
spec:
  storageClassName: optimized  # allowVolumeExpansion: true 여야 함
  accessModes:
    - ReadWriteOnce
  resources:
    requests:
      storage: 50Gi  # 초기 용량
```

용량 확장 후:
```bash
# PVC 용량 업데이트
kubectl patch pvc expandable-pvc -p '{"spec":{"resources":{"requests":{"storage":"100Gi"}}}}'

# 파일시스템 확장 확인
kubectl get pvc expandable-pvc
```

---

## 데이터 보호: 스냅샷과 복구

CSI 볼륨 스냅샷을 사용하면 특정 시점의 데이터 상태를 보존하고 복구할 수 있습니다.

### 스냅샷 클래스 정의
```yaml
apiVersion: snapshot.storage.k8s.io/v1
kind: VolumeSnapshotClass
metadata:
  name: csi-snapshot-class
driver: ebs.csi.aws.com
deletionPolicy: Delete
```

### 스냅샷 생성
```yaml
apiVersion: snapshot.storage.k8s.io/v1
kind: VolumeSnapshot
metadata:
  name: data-backup-20240115
spec:
  source:
    persistentVolumeClaimName: production-data
  volumeSnapshotClassName: csi-snapshot-class
```

### 스냅샷으로부터 복구
```yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: restored-data
spec:
  storageClassName: fast
  dataSource:
    name: data-backup-20240115
    kind: VolumeSnapshot
    apiGroup: snapshot.storage.k8s.io
  accessModes:
    - ReadWriteOnce
  resources:
    requests:
      storage: 10Gi
```

**운영 권장사항**: Velero와 같은 백업 도구를 CSI 스냅샷과 통합하여 전체적인 백업/복구 전략을 자동화하는 것이 좋습니다.

---

## 토폴로지 인식 스토리지

클라우드 환경에서 블록 스토리지는 특정 가용 영역에 바인딩됩니다. `WaitForFirstConsumer` 바인딩 모드를 사용하면 Pod가 스케줄된 노드의 가용 영역에 볼륨을 생성하여 스케줄링 실패를 방지할 수 있습니다.

```yaml
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: zone-aware
provisioner: ebs.csi.aws.com
volumeBindingMode: WaitForFirstConsumer
allowedTopologies:
- matchLabelExpressions:
  - key: topology.kubernetes.io/zone
    values:
      - us-east-1a
      - us-east-1b
```

---

## 스토리지 선택 가이드

애플리케이션 요구사항에 따라 적절한 스토리지 솔루션을 선택하는 것이 중요합니다.

| 요구사항 | 추천 솔루션 | 고려사항 |
|---|---|---|
| 단일 Pod 전용 스토리지 | 블록 스토리지(EBS, Azure Disk) | RWO 접근 모드, 높은 성능 |
| 다중 Pod 공유 스토리지 | 파일 스토리지(EFS, Azure Files, NFS) | RWX 접근 모드, 네트워크 지연 |
| 고성능 데이터베이스 | 로컬 SSD 또는 고성능 블록 스토리지 | 높은 IOPS, 낮은 지연 시간 |
| 아카이브/백업 | 오브젝트 스토리지(S3, Blob Storage) | 낮은 비용, 높은 내구성 |
| 컨피그맵/시크릿 | Kubernetes 기본 볼륨 타입 | 보안, 자동 관리 |

**결정 요소**:
- 데이터 접근 패턴: 순차적 vs 임의 접근
- 동시 접근 요구사항: 단일 vs 다중 Pod 접근
- 성능 요구사항: IOPS, 처리량, 지연 시간
- 비용 제약: 스토리지 클래스별 가격 모델
- 데이터 보존 요구사항: 스냅샷, 백업, 복구 SLA

---

## 운영 명령어 참조

### 기본 조회 명령
```bash
# 스토리지 클래스 확인
kubectl get storageclass
kubectl describe storageclass <name>

# PV/PVC 상태 확인
kubectl get pv
kubectl get pvc --all-namespaces
kubectl describe pvc <name> -n <namespace>

# PVC와 연결된 Pod 확인
kubectl get pods --all-namespaces -o=jsonpath='{range .items[*]}{.metadata.namespace}{"\t"}{.metadata.name}{"\t"}{range .spec.volumes[*]}{.persistentVolumeClaim.claimName}{" "}{end}{"\n"}{end}'
```

### 문제 진단
```bash
# 이벤트 로그 확인
kubectl get events --sort-by=.lastTimestamp | tail -n 50

# CSI 드라이버 상태 확인
kubectl get pods -n kube-system | grep csi

# 마운트 상태 확인 (노드에서 실행)
mount | grep kubelet
```

---

## 문제 해결 가이드

### 일반적인 문제와 해결 방안

**PVC가 Pending 상태인 경우**:
- 원인: 사용 가능한 StorageClass가 없거나, 프로비저너가 비정상적이거나, 요청 사양이 지원되지 않음
- 해결: `kubectl describe pvc <name>`으로 이벤트 확인, StorageClass 설정 검증, CSI 드라이버 Pod 로그 확인

**Pod가 VolumeBinding으로 Pending 상태인 경우**:
- 원인: `Immediate` 바인딩 모드로 생성된 PV가 다른 가용 영역에 위치
- 해결: StorageClass의 `volumeBindingMode`를 `WaitForFirstConsumer`로 변경

**마운트 실패**:
- 원인: 권한 문제, 파일시스템 불일치, SELinux 정책 충돌
- 해결: 보안 컨텍스트 검증, 파일시스템 타입 확인, 드라이버 문서 참조

**성능 문제**:
- 원인: 부적절한 마운트 옵션, 디스크 유형 미스매치, 네트워크 지연
- 해결: 마운트 옵션 최적화, 디스크 클래스 업그레이드, I/O 패턴 분석

**볼륨 확장 실패**:
- 원인: StorageClass에서 `allowVolumeExpansion`이 비활성화되었거나 드라이버가 확장을 지원하지 않음
- 해결: StorageClass 설정 수정, CSI 드라이버 버전 확인

---

## 종합 예제 시나리오

### Nginx 웹 서버에 영구 스토리지 연결
```yaml
# PV 정의 (관리자 관점)
apiVersion: v1
kind: PersistentVolume
metadata:
  name: nginx-pv
spec:
  capacity:
    storage: 2Gi
  accessModes:
    - ReadWriteOnce
  volumeMode: Filesystem
  persistentVolumeReclaimPolicy: Retain
  storageClassName: manual
  hostPath:
    path: "/mnt/nginx-data"

# PVC 정의 (개발자 관점)
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: nginx-pvc
spec:
  storageClassName: manual
  accessModes:
    - ReadWriteOnce
  resources:
    requests:
      storage: 1Gi

# Pod 정의
apiVersion: v1
kind: Pod
metadata:
  name: nginx-with-storage
spec:
  containers:
  - name: nginx
    image: nginx:alpine
    ports:
    - containerPort: 80
    volumeMounts:
    - name: web-content
      mountPath: /usr/share/nginx/html
  volumes:
  - name: web-content
    persistentVolumeClaim:
      claimName: nginx-pvc
```

### PostgreSQL 데이터베이스용 StatefulSet 구성
```yaml
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: postgres-sc
provisioner: ebs.csi.aws.com
parameters:
  type: io2
  iops: "10000"
  fsType: xfs
allowVolumeExpansion: true
volumeBindingMode: WaitForFirstConsumer

---
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
        env:
        - name: POSTGRES_PASSWORD
          valueFrom:
            secretKeyRef:
              name: postgres-secret
              key: password
        volumeMounts:
        - name: postgres-data
          mountPath: /var/lib/postgresql/data
        securityContext:
          runAsUser: 999
          runAsGroup: 999
  volumeClaimTemplates:
  - metadata:
      name: postgres-data
    spec:
      accessModes:
        - ReadWriteOnce
      storageClassName: postgres-sc
      resources:
        requests:
          storage: 100Gi
```

---

## 운영 모범 사례

### 설계 시 고려사항
1. **데이터 특성 분석**: 데이터의 접근 패턴, 크기, 성장률, 보존 요구사항을 이해
2. **적절한 스토리지 유형 선택**: RWO vs RWX, 블록 vs 파일, 로컬 vs 네트워크 스토리지 평가
3. **용량 계획**: 현재 요구사항과 예상 성장률을 고려한 용량 계획 수립

### 구현 모범 사례
1. **동적 프로비저닝 활용**: 수동 PV 관리를 피하고 StorageClass 기반 자동 프로비저닝 사용
2. **토폴로지 인식 구성**: `WaitForFirstConsumer` 바인딩 모드로 가용 영역 불일치 문제 방지
3. **확장 가능성 보장**: `allowVolumeExpansion: true` 설정으로 미래 용량 증가 대비
4. **보안 강화**: 적절한 보안 컨텍스트와 파일 권한 설정

### 운영 모범 사례
1. **모니터링 구현**: 스토리지 사용량, 성능 메트릭, CSI 드라이버 상태 모니터링
2. **백업 전략 수립**: 정기적인 스냅샷 생성 및 백업 절차 구현
3. **비용 관리**: 미사용 스토리지 정리, 적절한 스토리지 티어 선택
4. **재해 복구 계획**: 스냅샷 기반 복구 절차 테스트 및 문서화

### 거버넌스
1. **네임스페이스별 할당량 설정**:
```yaml
apiVersion: v1
kind: ResourceQuota
metadata:
  name: storage-quota
  namespace: production
spec:
  hard:
    requests.storage: "1Ti"
    persistentvolumeclaims: "50"
```
2. **정책 기반 접근 제어**: 특정 StorageClass 사용 제한, 라벨 기반 정책 적용
3. **감사 로깅**: 스토리지 생성, 수정, 삭제 이벤트 로깅 및 모니터링

---

## 결론

Kubernetes의 스토리지 시스템은 PV, PVC, StorageClass, CSI의 조합을 통해 강력하면서도 유연한 데이터 관리 체계를 제공합니다. 올바른 설계와 구현을 통해 다음과 같은 이점을 얻을 수 있습니다:

1. **애플리케이션과 인프라의 분리**: 개발자는 스토리지 세부사항보다는 용량과 성능 요구사항에 집중할 수 있습니다.
2. **다양한 스토리지 백엔드 지원**: 단일 인터페이스로 로컬 디스크부터 클라우드 스토리지까지 다양한 백엔드를 통합 관리할 수 있습니다.
3. **동적 프로비저닝**: 요청 시 자동으로 스토리지를 프로비저닝하여 운영 부담을 줄일 수 있습니다.
4. **상태 유지 애플리케이션 지원**: StatefulSet과 영구 스토리지의 조합으로 데이터베이스와 같은 상태 유지 워크로드를 효과적으로 운영할 수 있습니다.

성공적인 스토리지 운영을 위해서는 애플리케이션 요구사항을 철저히 분석하고, 적절한 스토리지 솔루션을 선택하며, 지속적인 모니터링과 최적화를 수행해야 합니다. 또한 보안, 백업, 재해 복구와 같은 운영 측면을 설계 단계부터 고려하는 것이 장기적인 성공을 보장합니다.

Kubernetes 스토리지는 단순히 데이터를 저장하는 기능을 넘어, 현대적이고 복잡한 애플리케이션을 지원하는 핵심 인프라 구성 요소로 자리 잡고 있습니다. 이 가이드의 개념과 모범 사례를 적용하여 안정적이고 확장 가능한 스토리지 환경을 구축하시기 바랍니다.