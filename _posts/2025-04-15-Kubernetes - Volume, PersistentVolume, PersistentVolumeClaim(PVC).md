---
layout: post
title: Kubernetes - Volume, PersistentVolume, PersistentVolumeClaim(PVC)
date: 2025-04-11 19:20:23 +0900
category: Kubernetes
---
# Kubernetes Volume, PersistentVolume, PersistentVolumeClaim 완전 가이드

## Volume의 필요성: 데이터 지속성 문제 해결

컨테이너와 Pod의 기본 파일 시스템은 일시적입니다. 재스케줄링, 롤링 업데이트, 노드 장애와 같은 이벤트가 발생하면 컨테이너 내부의 데이터는 손실될 수 있습니다. 데이터베이스, 파일 업로드, 장기 로그 저장과 같은 사용 사례에서는 Pod의 생명주기와 분리된 영속적인 저장소가 필수적입니다.

쿠버네티스는 이러한 요구사항을 충족시키기 위해 다양한 볼륨 유형을 제공합니다:

- **임시 볼륨(Ephemeral Volumes)**: Pod 수명에 종속되는 임시 저장소 (예: `emptyDir`, `configMap`, `secret`)
- **영구 볼륨(Persistent Volumes)**: 클러스터 리소스로서 관리되는 영속적 저장소 (PV/PVC)

---

## Kubernetes 볼륨 아키텍처의 전체적 이해

```
Pod ───> PVC(요청자) ─── 바인딩 ───> PV(공급자) ───> 실제 스토리지(NFS/블록/파일/CSI)
```

이 아키텍처에서 각 구성 요소의 역할은 다음과 같습니다:

1. **Pod**: PVC를 참조하여 볼륨을 마운트하고 사용합니다.
2. **PersistentVolumeClaim (PVC)**: 스토리지 요구사항(용량, 접근 모드 등)을 정의합니다.
3. **PersistentVolume (PV)**: 실제 물리적 또는 네트워크 스토리지를 추상화합니다.
4. **실제 스토리지**: 클라우드 디스크, NFS 서버, 파일 시스템, CSI 드라이버 등

---

## 주요 볼륨 유형 비교

| 볼륨 유형 | 수명 주기 | 일반적인 사용 사례 | 고려사항 |
|---|---|---|---|
| `emptyDir` | Pod와 동일 | 임시 캐시, 빌드 중간 파일, 프로세스 간 공유 데이터 | 노드 또는 Pod 재시작 시 데이터 소멸, `medium: Memory` 옵션으로 메모리 기반 사용 가능 |
| `hostPath` | 노드 수명 | 로컬 개발 및 디버깅, 노드 특정 데이터 접근 | 운영 환경에서 권장되지 않음, 노드 간 이식성 없음, 보안 위험 가능성 |
| `configMap` / `secret` | Pod와 동일 | 애플리케이션 설정 파일, 자격 증명 및 비밀 정보 | 대용량 파일 또는 빈번한 변경에 적합하지 않음 |
| `persistentVolumeClaim` | Pod 독립적 | 데이터베이스, 파일 저장소, 로그 아카이브 등 영구 데이터 | PV/PVC/StorageClass 설계 및 정합성 필요 |

---

## PersistentVolume(PV)와 PersistentVolumeClaim(PVC) 심층 분석

### PersistentVolume 주요 구성 요소

- **`capacity.storage`**: 스토리지 용량 (예: 10Gi, 100Gi)
- **`accessModes`**: 접근 모드 (RWO, ROX, RWX, RWO-Pod)
- **`persistentVolumeReclaimPolicy`**: 삭제 정책 (Retain, Delete)
- **백엔드 스펙**: `nfs`, `csi`, `gcePersistentDisk`, `awsElasticBlockStore` 등 실제 스토리지 연결 정보

### PersistentVolumeClaim 주요 구성 요소

- **`resources.requests.storage`**: 요청 스토리지 용량
- **`accessModes`**: 필요한 접근 모드
- **`storageClassName`**: 동적 프로비저닝 시 사용할 StorageClass 지정
- **`dataSource`**: 볼륨 스냅샷 또는 클론 복원 시 사용

---

## 접근 모드(Access Modes) 설계 가이드

쿠버네티스는 다양한 접근 모드를 제공하여 다양한 워크로드 요구사항을 충족시킵니다:

| 접근 모드 | 의미 | 일반적인 백엔드 스토리지 | 사용 사례 |
|---|---|---|---|
| **RWO (ReadWriteOnce)** | 단일 노드에서 하나의 Pod만 읽기/쓰기 가능 | 블록 스토리지 (EBS, PD, Azure Disk) | 단일 인스턴스 데이터베이스 |
| **ROX (ReadOnlyMany)** | 여러 노드에서 동시 읽기 전용 접근 가능 | 파일 스토리지 (NFS 등) | 공유 정적 콘텐츠 |
| **RWX (ReadWriteMany)** | 여러 노드에서 동시 읽기/쓰기 접근 가능 | 파일 스토리지 (NFS, EFS, Azure Files) | 공유 파일 시스템, 협업 작업 |
| **RWO-Pod (ReadWriteOncePod)** | 단일 Pod만 읽기/쓰기 접근 가능 | 드라이버 지원 필요 | Pod 수준 격리 강화 |

**실무 설계 가이드라인**:
- 단일 인스턴스 데이터베이스: RWO 모드
- 여러 Pod가 공유하는 정적 콘텐츠: RWX 모드
- 다중 가용 영역 환경: 토폴로지 인식 및 `volumeBindingMode` 고려

---

## 리클레임 정책(Reclaim Policy) 이해

리클레임 정책은 PVC가 삭제된 후 PV와 해당 데이터를 어떻게 처리할지 결정합니다:

| 정책 | 설명 | 사용 시나리오 |
|---|---|---|
| **`Retain`** | PVC 삭제 후에도 PV와 데이터 보존 | 중요 데이터 보호, 수동 정리 필요 시 |
| **`Delete`** | PVC 삭제 시 PV와 실제 스토리지 자동 삭제 | 테스트 환경, 임시 데이터, 자동 청소 필요 시 |
| **`Recycle`** | 레거시 방식 (사용 권장되지 않음) | 더 이상 사용되지 않음 |

**데이터 보존 요구사항**이 높은 환경에서는 `Retain` 정책을 우선적으로 고려해야 합니다.

---

## 실전 예제 1: 수동 PV/PVC/Pod 구성

이 예제는 개념 이해를 위한 학습용으로, 운영 환경에서는 `hostPath` 대신 NFS, CSI, 또는 클라우드 디스크를 사용해야 합니다:

```yaml
# PersistentVolume 정의
apiVersion: v1
kind: PersistentVolume
metadata:
  name: manual-pv-example
spec:
  capacity:
    storage: 1Gi
  accessModes:
    - ReadWriteOnce
  persistentVolumeReclaimPolicy: Retain
  hostPath:
    path: "/mnt/data"  # 데모용으로만 사용, 운영 환경에서는 권장되지 않음

---
# PersistentVolumeClaim 정의
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: manual-pvc-example
spec:
  accessModes:
    - ReadWriteOnce
  resources:
    requests:
      storage: 1Gi

---
# Pod 정의 (PVC 사용)
apiVersion: v1
kind: Pod
metadata:
  name: pod-with-manual-pvc
spec:
  containers:
  - name: nginx-container
    image: nginx:1.27-alpine
    volumeMounts:
    - name: web-content
      mountPath: /usr/share/nginx/html
  volumes:
  - name: web-content
    persistentVolumeClaim:
      claimName: manual-pvc-example
```

**적용 및 검증**:
```bash
# 리소스 생성
kubectl apply -f manual-pv-pvc-pod.yaml

# 상태 확인
kubectl get pv,pvc,pod

# 상세 정보 확인
kubectl describe pvc manual-pvc-example

# 컨테이너 내부에서 테스트
kubectl exec -it pod-with-manual-pvc -- sh -c 'echo "Test Content" > /usr/share/nginx/html/test.txt && ls -la /usr/share/nginx/html'
```

---

## 실전 예제 2: NFS 기반 공유 스토리지 (RWX)

여러 Pod가 동시에 읽기와 쓰기를 수행해야 하는 경우 RWX 접근 모드가 필요하며, 파일 기반 스토리지가 적합합니다:

```yaml
# NFS PersistentVolume 정의
apiVersion: v1
kind: PersistentVolume
metadata:
  name: nfs-shared-pv
spec:
  capacity:
    storage: 10Gi
  accessModes:
    - ReadWriteMany
  persistentVolumeReclaimPolicy: Retain
  mountOptions:
    - nfsvers=4.1
  nfs:
    server: 10.0.0.1  # NFS 서버 IP 주소
    path: /exports/app-data

---
# PersistentVolumeClaim 정의
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: nfs-shared-pvc
spec:
  accessModes:
    - ReadWriteMany
  resources:
    requests:
      storage: 10Gi

---
# 다중 Pod 디플로이먼트 정의
apiVersion: apps/v1
kind: Deployment
metadata:
  name: web-application
spec:
  replicas: 3
  selector:
    matchLabels:
      app: web-app
  template:
    metadata:
      labels:
        app: web-app
    spec:
      containers:
      - name: nginx
        image: nginx:1.27-alpine
        volumeMounts:
        - name: shared-storage
          mountPath: /usr/share/nginx/html
      volumes:
      - name: shared-storage
        persistentVolumeClaim:
          claimName: nfs-shared-pvc
```

---

## 동적 프로비저닝과 StorageClass 활용

동적 프로비저닝은 PVC 생성 시 StorageClass와 CSI 드라이버를 통해 자동으로 PV를 생성하고 바인딩합니다. 다중 가용 영역 환경에서는 `WaitForFirstConsumer` 바인딩 모드를 사용하여 스케줄링된 노드의 토폴로지에 맞는 볼륨을 생성할 수 있습니다:

```yaml
# StorageClass 정의
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: gp3-wait-for-consumer
provisioner: ebs.csi.aws.com
parameters:
  type: gp3
reclaimPolicy: Delete
allowVolumeExpansion: true
volumeBindingMode: WaitForFirstConsumer

---
# PersistentVolumeClaim 정의
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: dynamic-pvc-example
spec:
  storageClassName: gp3-wait-for-consumer
  accessModes:
    - ReadWriteOnce
  resources:
    requests:
      storage: 20Gi
```

**검증 절차**:
```bash
# 리소스 생성
kubectl apply -f storageclass-pvc.yaml

# StorageClass 확인
kubectl get storageclass

# PVC 상태 모니터링
kubectl get pvc dynamic-pvc-example -w
```

---

## 권한 및 보안 컨텍스트 구성

비루트(non-root) 사용자로 컨테이너를 실행할 때 마운트된 볼륨에 쓰기 권한이 없는 경우, 다음과 같은 보안 컨텍스트 설정을 통해 문제를 해결할 수 있습니다:

- **`securityContext.runAsUser` / `runAsGroup`**: 컨테이너 실행 사용자/그룹 ID 지정
- **`securityContext.fsGroup`**: 마운트 지점의 파일 시스템 그룹 권한 설정
- **SELinux 컨텍스트**: SELinux 환경에서 적절한 컨텍스트 설정

**예시 구성**:
```yaml
apiVersion: v1
kind: Pod
metadata:
  name: security-context-demo
spec:
  securityContext:
    fsGroup: 2000
  containers:
  - name: demo-app
    image: alpine:3.20
    command: ["sh", "-c", "id && touch /data/test-file && ls -la /data; sleep 3600"]
    volumeMounts:
    - name: app-data
      mountPath: /data
  volumes:
  - name: app-data
    persistentVolumeClaim:
      claimName: dynamic-pvc-example
```

---

## 하위 경로(SubPath) 마운트 활용

단일 파일이나 특정 하위 디렉터리만 마운트해야 할 경우 `subPath`를 사용할 수 있습니다:

```yaml
volumeMounts:
- name: application-config
  mountPath: /etc/myapp/config.yaml
  subPath: config.yaml
  readOnly: true
```

**주의사항**: `subPath`를 사용하는 경우 실시간 변경 사항이 제한될 수 있습니다.

---

## StatefulSet과 볼륨 클레임 템플릿

상태 저장 워크로드(데이터베이스, 메시지 큐 등)의 경우 각 Pod마다 고유한 영구 스토리지가 필요합니다. StatefulSet은 `volumeClaimTemplates`를 통해 이를 자동으로 관리합니다:

```yaml
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: postgres-cluster
spec:
  serviceName: postgres-service
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
        image: postgres:16
        env:
        - name: POSTGRES_PASSWORD
          valueFrom:
            secretKeyRef:
              name: postgres-credentials
              key: password
        volumeMounts:
        - name: database-storage
          mountPath: /var/lib/postgresql/data
  volumeClaimTemplates:
  - metadata:
      name: database-storage
    spec:
      storageClassName: gp3-wait-for-consumer
      accessModes:
        - ReadWriteOnce
      resources:
        requests:
          storage: 20Gi
```

---

## 볼륨 스냅샷을 통한 백업 및 복구

CSI 스냅샷 기능을 활용하면 특정 시점의 볼륨 상태를 백업하고 복구할 수 있습니다:

```yaml
# VolumeSnapshotClass 정의
apiVersion: snapshot.storage.k8s.io/v1
kind: VolumeSnapshotClass
metadata:
  name: csi-snapshot-class
driver: ebs.csi.aws.com
deletionPolicy: Delete

---
# VolumeSnapshot 생성
apiVersion: snapshot.storage.k8s.io/v1
kind: VolumeSnapshot
metadata:
  name: database-snapshot
spec:
  volumeSnapshotClassName: csi-snapshot-class
  source:
    persistentVolumeClaimName: dynamic-pvc-example

---
# 스냅샷으로부터 PVC 생성
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: restored-pvc
spec:
  storageClassName: gp3-wait-for-consumer
  dataSource:
    name: database-snapshot
    kind: VolumeSnapshot
    apiGroup: snapshot.storage.k8s.io
  accessModes:
    - ReadWriteOnce
  resources:
    requests:
      storage: 20Gi
```

---

## 볼륨 확장(Resize) 작업

StorageClass에서 `allowVolumeExpansion: true`를 설정하면 PVC의 스토리지 용량을 확장할 수 있습니다:

```bash
# PVC 용량 확장
kubectl patch pvc dynamic-pvc-example -p '{"spec":{"resources":{"requests":{"storage":"40Gi"}}}}'

# 확장 상태 확인
kubectl get pvc dynamic-pvc-example

# 컨테이너 내부에서 파일 시스템 확인
kubectl exec -it <pod-name> -- df -h
```

**참고사항**: 드라이버와 파일 시스템에 따라 온라인 확장 지원 여부가 다릅니다.

---

## 운영 및 모니터링 명령어 모음

```bash
# 기본 상태 확인
kubectl get storageclass
kubectl get persistentvolume
kubectl get persistentvolumeclaim
kubectl get volumesnapshotclass
kubectl get volumesnapshot

# 상세 정보 확인
kubectl describe storageclass <storageclass-name>
kubectl describe persistentvolume <pv-name>
kubectl describe persistentvolumeclaim <pvc-name>
kubectl describe volumesnapshot <snapshot-name>

# 이벤트 기반 문제 진단
kubectl describe persistentvolumeclaim <pvc-name> | sed -n '/Events/,$p'
kubectl describe pod <pod-name> | sed -n '/Events/,$p'

# 컨테이너 내부 스토리지 상태 확인
kubectl exec -it <pod-name> -- ls -la /mount-point
kubectl exec -it <pod-name> -- df -h

# 리소스 삭제
kubectl delete persistentvolumeclaim <pvc-name>
kubectl delete persistentvolume <pv-name>
```

---

## 일반적인 문제 해결 가이드

| 증상 | 1차 확인 방법 | 가능한 원인 | 해결 방안 |
|---|---|---|---|
| PVC가 Pending 상태 | `kubectl describe pvc` 이벤트 확인 | 매칭되는 PV 없음, StorageClass 누락, 접근 모드 불일치 | PVC 사양 조정, 적절한 StorageClass 지정, 접근 모드 재검토 |
| Pod 스케줄링 실패 | `kubectl describe pod` 이벤트 확인 | `WaitForFirstConsumer` 토폴로지 불일치, 리소스 부족 | 노드 풀 및 가용 영역 확인, 리소스 요청량 조정 |
| Pod가 ContainerCreating 상태에서 정지 | 마운트/연결 이벤트 확인 | CSI 드라이버 장애, IAM/네트워크 문제 | CSI 드라이버 로그 확인, 권한 및 네트워크 구성 검증 |
| 권한 오류(EACCES) | 컨테이너 로그 및 권한 확인 | runAsUser/fsGroup 미설정, SELinux 컨텍스트 문제 | fsGroup 및 runAsUser 설정 조정, 보안 컨텍스트 검토 |
| RWX 접근 모드 필요하지만 지원되지 않음 | PVC 및 StorageClass 확인 | 블록 스토리지 사용(RWO만 지원) | 파일 스토리지(NFS, EFS, Azure Files 등)로 전환 |
| 볼륨 확장 실패 | PVC 이벤트 확인 | 드라이버 미지원, 오프라인 확장 필요 | 드라이버 기능 확인, 재시작 또는 언마운트 후 재시도 |

**빠른 진단 스크립트**:
```bash
# PVC 상태 확인
kubectl get pvc <pvc-name> -o wide

# 관련 PV 찾기
kubectl get pv | grep -n <claim-name>

# StorageClass 상세 정보
kubectl get sc <storageclass-name> -o yaml | head -n 120

# 최근 이벤트 확인
kubectl get events --sort-by=.lastTimestamp | tail -n 20
```

---

## 통합 예제: 완전한 웹 애플리케이션 구성

이 예제는 동적 프로비저닝, PVC, 디플로이먼트, 서비스를 통합한 실전용 구성입니다:

```yaml
# StorageClass 정의
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: standard-wait-for-consumer
provisioner: ebs.csi.aws.com  # 환경에 맞게 수정 필요
parameters:
  type: gp3
reclaimPolicy: Delete
allowVolumeExpansion: true
volumeBindingMode: WaitForFirstConsumer

---
# PersistentVolumeClaim 정의
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: web-application-pvc
spec:
  storageClassName: standard-wait-for-consumer
  accessModes:
    - ReadWriteOnce
  resources:
    requests:
      storage: 2Gi

---
# 디플로이먼트 정의
apiVersion: apps/v1
kind: Deployment
metadata:
  name: web-application
spec:
  replicas: 2
  selector:
    matchLabels:
      app: web-app
  template:
    metadata:
      labels:
        app: web-app
    spec:
      securityContext:
        fsGroup: 2000
      containers:
      - name: nginx
        image: nginx:1.27-alpine
        ports:
        - containerPort: 80
        volumeMounts:
        - name: web-content
          mountPath: /usr/share/nginx/html
      volumes:
      - name: web-content
        persistentVolumeClaim:
          claimName: web-application-pvc

---
# 서비스 정의
apiVersion: v1
kind: Service
metadata:
  name: web-application-service
spec:
  type: NodePort
  selector:
    app: web-app
  ports:
  - port: 80
    targetPort: 80
    nodePort: 30080
```

**배포 및 검증**:
```bash
# 모든 리소스 생성
kubectl apply -f complete-web-application.yaml

# 배포 상태 확인
kubectl rollout status deployment/web-application

# PVC 상태 확인
kubectl get pvc web-application-pvc

# 서비스 정보 확인
kubectl get service web-application-service

# 컨텐츠 생성 및 테스트
kubectl exec -it deploy/web-application -- sh -c 'echo "<h1>Application Running</h1>" > /usr/share/nginx/html/index.html'
curl http://<node-ip>:30080
```

---

## 결론: 효과적인 Kubernetes 스토리지 관리 전략

쿠버네티스 볼륨 시스템은 컨테이너의 일시성과 데이터의 영속성 사이의 간극을 효과적으로 해결합니다. 성공적인 스토리지 관리를 위해서는 다음 원칙들을 준수해야 합니다:

### 핵심 설계 원칙

1. **요구사항 분석 및 적절한 접근 모드 선택**: 워크로드의 특성에 맞는 접근 모드(RWO, ROX, RWX)를 선택하세요. 단일 인스턴스 애플리케이션은 RWO를, 공유 파일 시스템은 RWX를 고려하세요.

2. **동적 프로비저닝과 StorageClass 활용**: 수동 PV 관리는 복잡성과 운영 부담을 증가시킵니다. 가능한 경우 동적 프로비저닝과 StorageClass를 활용하여 자동화된 스토리지 관리를 구현하세요.

3. **다중 가용 영역 환경 고려**: 클라우드 환경에서는 `WaitForFirstConsumer` 바인딩 모드를 사용하여 Pod가 스케줄링된 노드의 가용 영역에 맞는 스토리지를 자동으로 프로비저닝하도록 구성하세요.

4. **데이터 보호 및 리클레임 정책 설정**: 중요 데이터는 `Retain` 정책을 사용하여 실수로 인한 데이터 손실을 방지하세요. 테스트 환경에서는 `Delete` 정책을 사용하여 자동 정리 기능을 활용하세요.

5. **보안 및 권한 관리**: 비루트 컨테이너 실행 시 `fsGroup`과 `runAsUser`를 적절히 설정하여 스토리지 접근 권한 문제를 사전에 예방하세요.

6. **백업 및 복구 전략 수립**: VolumeSnapshot 기능을 활용한 정기적인 백업 전략을 수립하고, 재해 복구 절차를 문서화하세요.

### 운영 모범 사례

- **모니터링 및 알림 설정**: PVC Pending 상태, 마운트 실패, 스토리지 용량 임계치 초과 등을 모니터링하고 적시에 대응할 수 있는 알림 체계를 구축하세요.

- **비용 및 성능 최적화**: 워크로드의 IOPS, 처리량, 지연 시간 요구사항에 맞는 스토리지 유형을 선택하고, 사용하지 않는 볼륨은 정기적으로 정리하세요.

- **문서화 및 지식 공유**: 팀 내 스토리지 설계 표준, 문제 해결 절차, 모범 사례를 문서화하고 정기적으로 검토하세요.

- **테스트 및 검증**: 새로운 스토리지 구성을 프로덕션에 적용하기 전에 스테이징 환경에서 충분히 테스트하고 검증하세요.

### 지속적인 개선

쿠버네티스 스토리지 생태계는 지속적으로 발전하고 있습니다. CSI(Container Storage Interface) 표준의 도입으로 다양한 스토리지 백엔드와의 통합이 더욱 용이해졌습니다. 조직의 요구사항 변화와 기술 발전에 맞춰 스토리지 전략을 지속적으로 개선하고 최적화하세요.

효과적인 스토리지 관리는 안정적이고 확장 가능한 쿠버네티스 환경의 핵심 요소입니다. 이 가이드에서 제시한 개념과 모범 사례를 기반으로 조직에 적합한 스토리지 전략을 수립하고 구현한다면, 데이터 지속성 문제를 효과적으로 해결하면서도 운영 효율성을 극대화할 수 있을 것입니다.

---

## 참고 명령어 요약

```bash
# 기본 상태 확인
kubectl get storageclass
kubectl get persistentvolume
kubectl get persistentvolumeclaim
kubectl get events --sort-by=.lastTimestamp | tail -n 10

# 상세 진단
kubectl describe persistentvolumeclaim <pvc-name>
kubectl describe persistentvolume <pv-name>
kubectl exec -it <pod-name> -- df -h
kubectl exec -it <pod-name> -- ls -la /mount-point

# 문제 해결을 위한 이벤트 확인
kubectl get events --field-selector involvedObject.kind=PersistentVolumeClaim,involvedObject.name=<pvc-name>
```