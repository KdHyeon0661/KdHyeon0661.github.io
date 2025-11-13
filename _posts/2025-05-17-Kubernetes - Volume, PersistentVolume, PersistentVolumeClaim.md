---
layout: post
title: Kubernetes - Volume, PersistentVolume, PersistentVolumeClaim
date: 2025-05-11 19:20:23 +0900
category: Kubernetes
---
# Volume, PersistentVolume, PersistentVolumeClaim(PVC)

## 0. 큰 그림: 저장소 계층도

```
[Pod]
  └─ volumeMounts -> [Volume 스펙]
                        └─ (1) ephemeral(e.g., emptyDir, configMap, secret, projected)
                        └─ (2) persistentVolumeClaim -> [PVC] -> [PV] -> [물리/클라우드 디스크(CSI)]
```

- **ephemeral**: Pod 라이프사이클과 함께 소멸(예: `emptyDir`)
- **persistent**: Pod 라이프사이클과 분리, **PV/PVC**로 관리(예: EBS, AzureDisk, NFS, Ceph, EFS 등)

---

## 1. Volume 기초와 유형

### 1.1 주요 볼륨 타입(요약)

| 타입 | 특성 | 사용 예 |
|---|---|---|
| `emptyDir` | Pod 생명주기와 동일, 노드 디스크(메모리 옵션 가능) | 캐시/임시 파일 |
| `hostPath` | 노드 경로를 Pod에 마운트 | 데몬형 에이전트, 로그 수집(주의) |
| `configMap`/`secret` | 설정/비밀 값 투입, 파일/환경변수 | 설정 주입, TLS 키 |
| `downwardAPI` | Pod 메타데이터 파일/ENV 주입 | 빌드정보/라벨 참조 |
| `persistentVolumeClaim` | **지속형**. PVC에 바인딩된 PV를 마운트 | DB, 미디어, 업로드 |
| `projected` | configMap/secret/downwardAPI/ServiceAccountToken 통합 | 복합 설정 |

> **중요**: 애플리케이션 데이터는 `persistentVolumeClaim`로 관리해야 **Pod 교체/스케일**에도 데이터 일관성을 보장한다.

---

## 2. PersistentVolume(PV)와 PersistentVolumeClaim(PVC)

### 2.1 개념
- **PV**: 클러스터 관점의 **실제 스토리지 단위**(관리자/프로비저너가 생성)
- **PVC**: 개발자/배포자가 요구하는 **스토리지 요청서** (크기/모드/클래스)

```
Pod ──(volumeMounts)──> PVC ──(바인딩)──> PV ──> 물리 스토리지/CSI 드라이버
```

### 2.2 PV 예시(수동 프로비저닝, 데모용 hostPath)
```yaml
apiVersion: v1
kind: PersistentVolume
metadata:
  name: pv-example
spec:
  capacity:
    storage: 1Gi
  accessModes: [ReadWriteOnce]
  persistentVolumeReclaimPolicy: Retain
  volumeMode: Filesystem
  hostPath:
    path: "/mnt/data"
```

### 2.3 PVC 예시
```yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: pvc-example
spec:
  accessModes: [ReadWriteOnce]
  resources:
    requests:
      storage: 500Mi
```

### 2.4 Pod에서 PVC 사용
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

## 3. AccessMode, VolumeMode, ReclaimPolicy

### 3.1 AccessMode
| 모드 | 의미 | 예 |
|---|---|---|
| `RWO` | 단일 **노드**에서 Read/Write | EBS/AzureDisk/GCE PD 기본 |
| `ROX` | 멀티 노드 ReadOnly | 일부 공유 스토리지 |
| `RWX` | 멀티 노드 Read/Write | NFS, CephFS, EFS, Azure Files |

> 멀티 Pod가 **동일 데이터를 동시에 RW**해야 하면 **RWX**가 필요. 클라우드 블록디스크는 보통 **RWO** 제한이므로 **공유 파일시스템(CSI)**이 대안.

### 3.2 VolumeMode
- `Filesystem`: 일반 마운트(대부분 기본)
- `Block`: **원시 블록 디바이스**로 노출(데이터베이스/특수 워크로드 튜닝용)

```yaml
spec:
  volumeMode: Block
```

### 3.3 ReclaimPolicy
| 값 | 동작 |
|---|---|
| `Retain` | PVC 삭제해도 PV/데이터 유지(수동 정리) |
| `Delete` | PVC 삭제 시 PV/백엔드 볼륨 삭제 |
| `Recycle` | 폐지됨(Deprecated) |

> 운영에서는 **실수 방지**를 위해 민감 데이터는 `Retain`을 고려. 자동 생성 리소스는 `Delete`가 편의.

---

## 4. StorageClass와 동적 프로비저닝(Dynamic Provisioning)

### 4.1 StorageClass 정의
```yaml
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: fast
provisioner: kubernetes.io/aws-ebs           # 예: 레거시 in-tree (권장: CSI 프로비저너)
parameters:
  type: gp3
  fsType: ext4
reclaimPolicy: Delete
allowVolumeExpansion: true
volumeBindingMode: WaitForFirstConsumer
```

핵심 필드:
- `provisioner`: **CSI 드라이버**(예: `ebs.csi.aws.com`, `disk.csi.azure.com`, `pd.csi.storage.gke.io`, `efs.csi.aws.com`, `file.csi.azure.com`, `filestore.csi.storage.gke.io` 등)
- `parameters`: 스토리지 클래스별 파라미터(디스크 타입, IOPS, Throughput 등)
- `allowVolumeExpansion`: PVC **확장** 허용
- `volumeBindingMode`: `Immediate` vs `WaitForFirstConsumer`(**토폴로지** 결정 후 디스크 생성 → 스케줄 실패 예방)

### 4.2 PVC에서 StorageClass 사용
```yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: dynamic-pvc
spec:
  storageClassName: fast
  accessModes: [ReadWriteOnce]
  resources:
    requests:
      storage: 10Gi
```
→ 생성 즉시 **PV가 자동 생성**되어 바인딩.

---

## 5. StatefulSet + volumeClaimTemplates 패턴

**Pod N개 각각** 고유 디스크를 가지려면 StatefulSet을 사용한다.

```yaml
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: web
spec:
  serviceName: "web"
  replicas: 3
  selector:
    matchLabels: { app: web }
  template:
    metadata: { labels: { app: web } }
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
      accessModes: [ReadWriteOnce]
      storageClassName: fast
      resources:
        requests: { storage: 5Gi }
```

- 생성 결과: `www-web-0`, `www-web-1`, `www-web-2` PVC/PV 자동 생성
- **Pod 교체/재스케줄**에도 **각 Pod의 디스크**가 유지

---

## 6. 데이터 초기화/마이그레이션: initContainer + subPath

### 6.1 초기 데이터 투입
```yaml
apiVersion: v1
kind: Pod
metadata:
  name: app-init
spec:
  initContainers:
  - name: init-content
    image: busybox
    command: ["sh", "-c", "echo 'Hello' > /workdir/index.html"]
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

### 6.2 디렉터리 매핑: `subPath`
```yaml
volumeMounts:
- name: data
  mountPath: /app/uploads
  subPath: uploads
```
> **주의**: `subPath`는 마운트 시점에만 평가되며, 동적 경로 문자열은 `subPathExpr`(환경변수 확장) 사용. 일부 커널/드라이버에서 **권한 이슈**가 있을 수 있으니 테스트 필요.

---

## 7. 보안·권한: fsGroup, SELinux, readOnlyRootFilesystem

### 7.1 Pod 보안 컨텍스트
```yaml
spec:
  securityContext:
    fsGroup: 2000
    fsGroupChangePolicy: "OnRootMismatch"  # Kubernetes 1.20+
```
- 마운트된 볼륨의 파일 소유권을 `fsGroup`으로 조정
- `OnRootMismatch`: 루트 소유만 변경해 **지연 최소화**

### 7.2 컨테이너 보안 컨텍스트
```yaml
containers:
- name: app
  securityContext:
    runAsUser: 1000
    runAsGroup: 1000
    readOnlyRootFilesystem: true
```

### 7.3 SELinux(사용 배포판 한정)
```yaml
securityContext:
  seLinuxOptions:
    type: "spc_t"  # 예시. 실제 정책/타입은 환경에 맞게
```

> 클라우드/배포판별 CSI 드라이버 문서에 **권한/마운트 옵션** 가이드를 반드시 확인할 것.

---

## 8. 성능 튜닝: mountOptions, fsType, atime, 블록 모드

### 8.1 StorageClass 또는 PV에 마운트 옵션
```yaml
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata: { name: perf }
provisioner: ebs.csi.aws.com
parameters:
  type: gp3
  fsType: xfs
mountOptions:
  - noatime
  - nodiratime
```

- `noatime`: 읽기 접근시 atime 미갱신 → 메타데이터 IO 감소
- `xfs`는 대용량/병렬 IO에 유리한 경우가 많다(워크로드/벤치마크 필수)

### 8.2 PVC 확장(online resize)
```yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata: { name: db }
spec:
  storageClassName: perf
  accessModes: [ReadWriteOnce]
  resources: { requests: { storage: 200Gi } }
```
- SC가 `allowVolumeExpansion: true`여야 함
- 파일시스템 온라인 확장 지원 여부 확인(`resize2fs`, `xfs_growfs`)

---

## 9. 스냅샷 & 복구(CSI VolumeSnapshot)

### 9.1 CRD 설치(클러스터당 1회, 배포별 번들 사용 권장)
- `VolumeSnapshotClass`, `VolumeSnapshot`, `VolumeSnapshotContent`

### 9.2 스냅샷 클래스
```yaml
apiVersion: snapshot.storage.k8s.io/v1
kind: VolumeSnapshotClass
metadata: { name: csi-snapclass }
driver: ebs.csi.aws.com
deletionPolicy: Delete
```

### 9.3 스냅샷 생성
```yaml
apiVersion: snapshot.storage.k8s.io/v1
kind: VolumeSnapshot
metadata: { name: data-snap }
spec:
  source:
    persistentVolumeClaimName: my-pvc
  volumeSnapshotClassName: csi-snapclass
```

### 9.4 스냅샷으로 PVC 복원
```yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata: { name: restore-pvc }
spec:
  storageClassName: fast
  dataSource:
    name: data-snap
    kind: VolumeSnapshot
    apiGroup: snapshot.storage.k8s.io
  accessModes: [ReadWriteOnce]
  resources:
    requests: { storage: 10Gi }
```

> 백업/복구는 **Velero**(+ CSI snapshot)로 자동화하는 케이스가 많다.

---

## 10. 토폴로지/스케줄링: `WaitForFirstConsumer`와 가용영역

- 블록 디스크는 **특정 AZ/Zone에 귀속**된다.
- `volumeBindingMode: WaitForFirstConsumer`를 사용하면 **Pod가 스케줄될 노드의 AZ**를 보고 그곳에 볼륨을 생성 → **스케줄 실패** 줄임.
- `allowedTopologies`(SC) 또는 CSI 별 파라미터로 영역 제한 가능.

```yaml
allowedTopologies:
- matchLabelExpressions:
  - key: topology.kubernetes.io/zone
    values: ["ap-northeast-2a", "ap-northeast-2c"]
```

---

## 11. RWX(공유 쓰기) 패턴 옵션

| 선택지 | 특성 | 주의 |
|---|---|---|
| NFS 서버(자체) | 간단/저비용 | HA/성능/백업 직접 구성 |
| Managed File(예: AWS EFS, Azure Files, GCP Filestore) | 관리형, RWX | 지연/성능·비용 모델 검토 |
| CephFS(Rook) | 온프레/자체 클러스터 | 운영 복잡성 |

**트래픽 패턴**(작은 파일 다수/메타데이터 작업 vs 순차 쓰기)과 **지연 민감도**를 고려해 선택.

---

## 12. 용량 계획(간단)

초기 용량 $$C_0$$, 연평균 증가율 $$g$$, 기간 $$t$$(년)일 때 최소 용량:
$$
C_{\text{min}} = C_0 \cdot (1 + g)^t
$$
- 저장 계층별(핫/웜/콜드) 분리와 **수명주기 정책**(오브젝트 스토리지 아카이빙)을 병행하면 비용 최적화에 유리.

---

## 13. 운영 명령 치트시트

```bash
# PV/PVC 현황
kubectl get pv
kubectl get pvc -A

# 상세 상태
kubectl describe pvc <name>
kubectl describe pv <name>

# 스토리지 클래스
kubectl get sc
kubectl describe sc <name>

# PVC ↔ Pod 매핑
kubectl get pod -A -o=jsonpath='{range .items[*]}{.metadata.namespace}{"\t"}{.metadata.name}{"\t"}{range .spec.volumes[*]}{.persistentVolumeClaim.claimName}{" "}{end}{"\n"}{end}' | column -t

# 파일시스템 타입 확인(노드 내부)
mount | grep kubelet
```

---

## 14. 장애·트러블슈팅 가이드

| 증상 | 원인 | 조치 |
|---|---|---|
| PVC Pending | SC 없음/프로비저너 비정상/요건 불일치 | `describe pvc`, 프로비저너 Pod/CRD 확인, SC 파라미터 수정 |
| Pod Pending(`VolumeBinding`) | `Immediate`로 생성된 PV가 다른 AZ | SC를 `WaitForFirstConsumer`로 전환 |
| 마운트 실패 | 권한/SELinux/fsType 불일치 | `securityContext.fsGroup`, fsType 일치, 드라이버 문서 확인 |
| IO 느림 | 마운트옵션/디스크 타입/네트워크 지연 | `noatime`, 디스크 클래스 상향, RW 패턴별 스토리지 전환 |
| 확장 실패 | `allowVolumeExpansion=false` | SC 옵션 변경, 드라이버 resize 지원 확인 |
| 데이터 소실 | `Delete` 정책, 테스트 클러스터 | 민감 볼륨은 `Retain`, 백업/스냅샷 정책 수립 |

**로그/이벤트 확인 포인트**
- CSI 컨트롤러/노드 플러그인 Pod 로그
- `kubectl describe pvc/pv/pod`의 Events 섹션

---

## 15. 예제 모음

### 15.1 Nginx + PV/PVC(수동)
```yaml
apiVersion: v1
kind: PersistentVolume
metadata: { name: my-pv }
spec:
  capacity: { storage: 1Gi }
  accessModes: [ReadWriteOnce]
  volumeMode: Filesystem
  persistentVolumeReclaimPolicy: Retain
  hostPath: { path: "/mnt/nginx-data" }
---
apiVersion: v1
kind: PersistentVolumeClaim
metadata: { name: my-pvc }
spec:
  accessModes: [ReadWriteOnce]
  resources: { requests: { storage: 1Gi } }
---
apiVersion: v1
kind: Pod
metadata: { name: nginx-pv }
spec:
  containers:
  - name: nginx
    image: nginx
    volumeMounts:
    - name: html
      mountPath: /usr/share/nginx/html
  volumes:
  - name: html
    persistentVolumeClaim: { claimName: my-pvc }
```

### 15.2 동적 프로비저닝 + RWX(EFS 예시, 개념용)
```yaml
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata: { name: efs-rwx }
provisioner: efs.csi.aws.com
parameters:
  provisioningMode: efs-ap
  fileSystemId: fs-12345678
  directoryPerms: "700"
mountOptions:
  - noatime
reclaimPolicy: Retain
allowVolumeExpansion: true
volumeBindingMode: WaitForFirstConsumer
---
apiVersion: v1
kind: PersistentVolumeClaim
metadata: { name: web-shared }
spec:
  accessModes: [ReadWriteMany]
  storageClassName: efs-rwx
  resources: { requests: { storage: 20Gi } }
```

### 15.3 StatefulSet DB(블록 모드, XFS 예시)
```yaml
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata: { name: db-sc }
provisioner: ebs.csi.aws.com
parameters:
  type: io2
  fsType: xfs
allowVolumeExpansion: true
volumeBindingMode: WaitForFirstConsumer
---
apiVersion: apps/v1
kind: StatefulSet
metadata: { name: pg }
spec:
  serviceName: "pg"
  replicas: 3
  selector: { matchLabels: { app: pg } }
  template:
    metadata: { labels: { app: pg } }
    spec:
      containers:
      - name: postgres
        image: postgres:16
        env:
          - name: POSTGRES_PASSWORD
            valueFrom:
              secretKeyRef: { name: pg-secret, key: password }
        volumeMounts:
        - name: data
          mountPath: /var/lib/postgresql/data
  volumeClaimTemplates:
  - metadata: { name: data }
    spec:
      accessModes: [ReadWriteOnce]
      storageClassName: db-sc
      resources: { requests: { storage: 200Gi } }
```

---

## 16. 멀티테넌시/거버넌스

- **네임스페이스별 ResourceQuota**로 **스토리지 총량 제한**:
```yaml
apiVersion: v1
kind: ResourceQuota
metadata:
  name: ns-storage-quota
  namespace: team-a
spec:
  hard:
    requests.storage: "2Ti"
    persistentvolumeclaims: "100"
```
- **StorageClass 노출 제한**: 어드미션 컨트롤러/레이블 기반
- **백업/스냅샷 정책**: 팀별 스냅샷 클래스/보존기간 표준화

---

## 17. 베스트 프랙티스 요약

1. **데이터 특성→스토리지 매핑**(RWO vs RWX, 블록 vs 파일, IOPS/지연)
2. **WaitForFirstConsumer**로 AZ 스케줄 실패 방지
3. **allowVolumeExpansion** 켜고, 온라인 리사이즈 경로 검증
4. **fsGroup/fsGroupChangePolicy**로 권한 이슈 최소화
5. 성능: `noatime`, 파일시스템 선택(ext4/xfs), 디스크 클래스 적정화
6. **스냅샷/백업** 자동화(Velero/CSI)
7. 민감 데이터는 **ReclaimPolicy=Retain**, 삭제 워크플로 따로
8. **RWX**가 필요하면 관리형 파일 서비스/분산 FS 고려
9. **로그/메트릭**: CSI 컨트롤러·노드 플러그인 상태 대시보드화
10. **DR/복구 연습**: 스냅샷 복원·리전 DR 리허설

---

## 18. 현업 체크리스트

- [ ] 워크로드 I/O 패턴(랜덤/순차, R/W 비율, 파일 크기) 파악
- [ ] AccessMode 요구사항 정리(RWO/RWX/ROX)
- [ ] SC 파라미터/토폴로지/확장 여부 설정
- [ ] 보안컨텍스트/SELinux/마운트옵션 테스트
- [ ] 스냅샷/백업/복구 절차 문서화 및 자동화
- [ ] 대시보드(프로비저너, PVC 상태, 용량, 리사이즈 이벤트)
- [ ] 비용 모니터링(IOPS/Throughput/GB-month, 네트워크 egress)

---

## 19. 결론

- **PV/PVC/StorageClass/CSI**는 Kubernetes 저장소의 **표준 추상화**이며, 올바른 설계로 **데이터 일관성과 민첩성**을 모두 얻을 수 있다.
- **AccessMode/VolumeMode/ReclaimPolicy/BindingMode/Topology**를 이해하면 **스케줄 실패·권한·성능** 문제를 사전에 방지할 수 있다.
- **StatefulSet + volumeClaimTemplates**는 스테이트풀 워크로드의 필수 패턴이고, **스냅샷·확장**으로 운영 편의성이 크게 올라간다.
- 마지막으로, **보안·백업·관측** 없이는 저장소 운영이 없다. 설계 초기부터 포함하라.

---
```bash
# 빠른 실습(요약)
kubectl get sc,pv,pvc
kubectl describe pvc <pvc>
kubectl apply -f your-pv-pvc-and-pod.yaml
kubectl get events --sort-by=.lastTimestamp | tail -n 50
```

**참고**
- Kubernetes Volumes: https://kubernetes.io/docs/concepts/storage/volumes/
- PersistentVolumes: https://kubernetes.io/docs/concepts/storage/persistent-volumes/
- StorageClass: https://kubernetes.io/docs/concepts/storage/storage-classes/
- CSI 스냅샷: https://kubernetes.io/docs/concepts/storage/volume-snapshots/
```
