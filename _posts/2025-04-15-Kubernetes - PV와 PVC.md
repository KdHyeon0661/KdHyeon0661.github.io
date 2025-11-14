---
layout: post
title: Docker - PV와 PVC
date: 2025-04-15 20:20:23 +0900
category: Docker
---
# Kubernetes에서의 볼륨 이해하기: PV와 PVC

- Pod의 일시성 한계를 이해하고, 영속 스토리지를 안전하게 다루기
- PersistentVolume(PV) / PersistentVolumeClaim(PVC)의 동작과 바인딩 모델 숙지
- StorageClass, 접근 모드, 리클레임 정책, 확장과 권한, 보안 컨텍스트 설계
- 동적 프로비저닝, StatefulSet, VolumeSnapshot 등 실전 패턴 정리
- 운영 명령과 흔한 장애 원인, 재현 가능한 해결 루틴 제공

---

## 왜 Kubernetes에서도 Volume이 필요한가

Pod는 일시적이다. 재스케줄링이나 롤링 업데이트, 장애 복구가 발생하면 컨테이너 파일시스템의 내용은 사라진다.
데이터베이스, 업로드 파일, 캐시 중 일부, 로그 보관 등은 **Pod 수명과 무관하게 보존**되어야 한다.

해결을 위해 Kubernetes는 다음을 제공한다.

- Ephemeral Volume: Pod와 함께 사라지는 임시 저장소(예: `emptyDir`, `configMap`, `secret`)
- Persistent Volume: 클러스터 리소스로 관리되는 **지속 볼륨**(PV/PVC)

---

## 용어와 큰 그림

```
+--------------------------+     +--------------------------+
| PersistentVolume (PV)    |<--->| 실제 스토리지(디스크/NFS/ |
| - 용량/접근모드/정책 정의 |     | 클라우드 블록/파일 등)    |
+--------------------------+     +--------------------------+
              ^
           바인딩
              v
+--------------------------+
| PersistentVolumeClaim    |
| (PVC)                    |  ← 애플리케이션(사용자)의 스토리지 요청
+--------------------------+
              ^
           마운트
              v
+--------------------------+
|           Pod            |
+--------------------------+
```

- PV: 클러스터 관리자가 제공하는 실제 스토리지에 대한 **추상화 리소스**
- PVC: 애플리케이션이 필요 용량/접근모드를 **요청**하는 객체
- Pod: PVC를 **마운트**하여 파일시스템로 사용

---

## Ephemeral vs Persistent

| 항목 | Ephemeral (예: emptyDir, configMap, secret) | Persistent (PV/PVC) |
|---|---|---|
| 수명 | Pod 수명과 동일 | Pod와 독립, 재시작 후에도 유지 |
| 데이터 유지 | 삭제됨 | 유지 |
| 선언 위치 | Pod spec 내부 | PV/PVC 별도 객체 |
| 예시 | 임시 캐시, 빌드 아티팩트, 설정 주입 | DB 데이터, 업로드, 장기 로그 |

추가 팁
- `emptyDir`는 노드 로컬 디스크나 메모리(`medium: Memory`)를 사용한다. 장애 시 데이터 손실 가능.
- `hostPath`는 개발용으로만 권장. 노드 결합과 보안 위험이 크다.

---

## PV 스펙의 핵심: 용량, 접근 모드, 리클레임 정책, 소유권

### 접근 모드(accessModes)

- `ReadWriteOnce` (RWO): 한 노드에서 하나의 Pod가 읽기/쓰기 마운트
- `ReadOnlyMany` (ROX): 여러 노드에서 읽기 전용
- `ReadWriteMany` (RWX): 여러 노드에서 읽기/쓰기 공유
- `ReadWriteOncePod` (RWO-Pod): 단 하나의 Pod만 R/W 마운트 (드라이버 지원 필요)

클라우드 블록 스토리지(예: AWS EBS, GCE PD, Azure Disk)는 일반적으로 RWO.
RWX가 필요하면 NFS, CephFS, EFS(AWS), Filestore(GCP), Azure Files 등 **파일 스토리지**를 고려한다.

### 리클레임 정책(persistentVolumeReclaimPolicy)

- `Retain`: PVC 삭제 후에도 PV와 실제 데이터 유지. 수동 정리 필요.
- `Delete`: PVC 삭제 시 PV와 스토리지(클라우드 디스크)도 삭제.
- `Recycle`: 구식. 사용하지 않는다.

운영 관점
- 데이터 보존이 필요한 워크로드는 `Retain`을 검토.
- 테스트 환경이나 임시 워크로드는 `Delete`로 자동 정리.

### 노드 어피니티와 바인딩 시점

- `nodeAffinity` (PV): 특정 노드/존에 귀속된 로컬/블록 볼륨 매핑에 사용.
- StorageClass의 `volumeBindingMode`
  - `Immediate`: PVC 생성 즉시 바인딩 시도
  - `WaitForFirstConsumer`: **Pod가 스케줄될 노드 정보**를 보고 해당 토폴로지에 맞는 볼륨을 프로비저닝. 멀티존에서 권장.

---

## 예제: 수동 PV + PVC + Pod (NFS)

### PersistentVolume (PV)

```yaml
apiVersion: v1
kind: PersistentVolume
metadata:
  name: pv-example
spec:
  capacity:
    storage: 1Gi
  accessModes:
    - ReadWriteMany
  persistentVolumeReclaimPolicy: Retain
  mountOptions:
    - nfsvers=4.1
  nfs:
    server: 10.0.0.1
    path: "/exports/data"
```

포인트
- NFS는 RWX를 지원하므로 여러 Pod에서 공유가 가능하다.
- `mountOptions`로 프로토콜/성능 옵션을 제어할 수 있다.

### PersistentVolumeClaim (PVC)

```yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: pvc-example
spec:
  accessModes:
    - ReadWriteMany
  resources:
    requests:
      storage: 1Gi
```

PV와 PVC의 `accessModes`/`storage`가 매칭되어야 바인딩된다.

### Pod에서 PVC 마운트

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: pod-using-pvc
spec:
  containers:
  - name: nginx
    image: nginx:1.27-alpine
    volumeMounts:
    - name: web
      mountPath: /usr/share/nginx/html
  volumes:
  - name: web
    persistentVolumeClaim:
      claimName: pvc-example
```

---

## 동적 프로비저닝(StorageClass)

PVC가 생성될 때 자동으로 PV를 만들어 붙여 준다. 클라우드/CSI 드라이버 환경에서 가장 일반적이다.

### StorageClass 예시 (블록, RWO)

```yaml
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: standard
provisioner: kubernetes.io/aws-ebs               # 예시: legacy in-tree. 현대엔 csi.driver 형태 사용
parameters:
  type: gp3
reclaimPolicy: Delete
allowVolumeExpansion: true
volumeBindingMode: WaitForFirstConsumer
```

### PVC (동적)

```yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: pvc-dynamic
spec:
  storageClassName: standard
  accessModes: [ "ReadWriteOnce" ]
  resources:
    requests:
      storage: 5Gi
```

### Pod

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: app
spec:
  containers:
  - name: app
    image: alpine:3.20
    command: ["sh","-c","echo ok && sleep 3600"]
    volumeMounts:
    - name: data
      mountPath: /data
  volumes:
  - name: data
    persistentVolumeClaim:
      claimName: pvc-dynamic
```

운영 팁
- `WaitForFirstConsumer`를 사용하면 멀티존에서 잘못된 존의 디스크가 먼저 생성되는 문제를 피할 수 있다.
- `allowVolumeExpansion: true`로 PVC 크기 증가를 허용한다.

---

## 볼륨 확장(Resize)과 파일시스템

- PVC 스펙의 `resources.requests.storage`를 늘리면 확장이 시도된다.
- 드라이버/파일시스템에 따라 온라인 확장 지원 여부가 다르다.
- 파일시스템 종류와 마운트 옵션(ext4, xfs 등)을 StorageClass `parameters`나 CSI 매개변수로 지정 가능.

예시: PVC 크기 확장

```bash
kubectl patch pvc pvc-dynamic -p '{"spec":{"resources":{"requests":{"storage":"10Gi"}}}}'
kubectl get pvc pvc-dynamic
```

---

## 권한과 보안: 퍼미션, fsGroup, 보안 컨텍스트

컨테이너 실행 사용자가 루트가 아니라면, 마운트된 볼륨에 쓰기 권한이 없을 수 있다. 대표적으로 다음을 조정한다.

- Pod/컨테이너 `securityContext.runAsUser`, `runAsGroup`
- Pod `securityContext.fsGroup`: 마운트된 볼륨의 파일/디렉터리에 그룹 권한을 부여
- SELinux 환경에서는 `seLinuxOptions` 또는 NFS `context` 등 보안 컨텍스트 고려

예시

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: perm-demo
spec:
  securityContext:
    fsGroup: 2000
  containers:
  - name: app
    image: alpine:3.20
    command: ["sh","-c","id && touch /data/test && ls -l /data; sleep 3600"]
    volumeMounts:
    - name: data
      mountPath: /data
  volumes:
  - name: data
    persistentVolumeClaim:
      claimName: pvc-dynamic
```

---

## subPath와 읽기 전용 마운트

특정 파일 또는 하위 디렉터리만 마운트하려면 `subPath`를 사용한다.

```yaml
volumeMounts:
- name: data
  mountPath: /app/config.yaml
  subPath: config.yaml
  readOnly: true
```

주의
- `subPath`는 실시간 변경 전파가 제한될 수 있다. 자주 바뀌는 설정은 디렉터리 전체 마운트와 핫리로드 전략을 고려한다.

---

## StatefulSet과 volumeClaimTemplates

StatefulSet은 각 Pod에 고유한 PVC를 자동 생성한다. 상태를 가진 워크로드(DB, 큐 등)에 적합하다.

```yaml
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: db
spec:
  serviceName: db
  replicas: 3
  selector:
    matchLabels: { app: db }
  template:
    metadata:
      labels: { app: db }
    spec:
      containers:
      - name: pg
        image: postgres:16
        env:
        - name: POSTGRES_PASSWORD
          valueFrom:
            secretKeyRef: { name: pg-secret, key: password }
        volumeMounts:
        - name: data
          mountPath: /var/lib/postgresql/data
  volumeClaimTemplates:
  - metadata:
      name: data
    spec:
      accessModes: [ "ReadWriteOnce" ]
      storageClassName: standard
      resources:
        requests:
          storage: 20Gi
```

각 Pod는 `data-db-0`, `data-db-1` 같은 **고유 PVC**를 가진다.

---

## VolumeSnapshot으로 백업/복구

CSI 스냅샷 CRD가 설치된 환경에서 스냅샷을 만들어 시점 복구를 구현할 수 있다.

### VolumeSnapshotClass

```yaml
apiVersion: snapshot.storage.k8s.io/v1
kind: VolumeSnapshotClass
metadata:
  name: csi-snapclass
driver: ebs.csi.aws.com           # 드라이버에 맞게 수정
deletionPolicy: Delete
```

### 스냅샷 생성

```yaml
apiVersion: snapshot.storage.k8s.io/v1
kind: VolumeSnapshot
metadata:
  name: data-snap
spec:
  volumeSnapshotClassName: csi-snapclass
  source:
    persistentVolumeClaimName: pvc-dynamic
```

### 스냅샷으로 PVC 복원

```yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: pvc-from-snap
spec:
  storageClassName: standard
  dataSource:
    name: data-snap
    kind: VolumeSnapshot
    apiGroup: snapshot.storage.k8s.io
  accessModes: [ "ReadWriteOnce" ]
  resources:
    requests:
      storage: 10Gi
```

---

## 운영 명령어

```bash
# 리소스 나열

kubectl get pv
kubectl get pvc
kubectl get sc
kubectl get volumesnapshotclass
kubectl get volumesnapshot

# 상세 확인

kubectl describe pv <pv>
kubectl describe pvc <pvc>
kubectl describe sc <sc>
kubectl describe volumesnapshot <snap>

# 스토리지 사용 확인(컨테이너 내부에서)

kubectl exec -it <pod> -- df -h
kubectl exec -it <pod> -- ls -l /mountpoint

# 삭제

kubectl delete pvc <pvc>
kubectl delete pv <pv>
```

---

## 트러블슈팅 가이드

| 증상 | 주요 확인 포인트 | 흔한 원인 | 해결책 |
|---|---|---|---|
| PVC가 Pending에서 멈춤 | `kubectl describe pvc` 이벤트 | 매칭되는 PV 없음, StorageClass 미지정, 접근모드 불일치 | PV 스펙/SC/접근모드 정합성 점검, SC 기본값 확인 |
| Pod가 Scheduling 안 됨 | `describe pod`에서 스케줄링 이벤트 | `WaitForFirstConsumer`에서 토폴로지 미만족, 노드 자원 부족 | 노드풀/존 확인, 리소스 요청 조정 |
| Pod가 ContainerCreating에서 멈춤 | `describe pod` Mount/Attach 이벤트 | CSI 컨트롤러/노드 플러그인 문제, 권한/네트워크 문제 | CSI 로그 확인, 드라이버/CRD 재배포, IAM/네트워크 점검 |
| 권한 오류(EACCES) | 컨테이너 로그, `id`, 디렉터리 퍼미션 | runAsUser 불일치, fsGroup 미설정, SELinux 컨텍스트 | Pod 보안 컨텍스트 조정, fsGroup/SELinux 설정 |
| RWX가 필요한데 불가 | PVC/StorageClass/드라이버 스펙 | 블록 스토리지 사용(RWO만) | 파일 스토리지(NFS/EFS/Azure Files/Filestore)로 전환 |
| 확장 실패 | PVC 이벤트 | 드라이버 미지원, 오프라인 확장 요구 | 드라이버 기능 확인, Pod 재시작/언마운트 후 재시도 |

이벤트 확인 예시

```bash
kubectl describe pvc <name> | sed -n '/Events/,$p'
kubectl describe pod <name> | sed -n '/Events/,$p'
```

---

## 체크리스트

- 스토리지 특성 파악
  - 지연/처리량/IOPS, 가용성 영역(존), 스냅샷/암호화 지원
- 접근 모드 선택
  - RWO vs RWX, 필요 시 파일 스토리지로 설계
- 바인딩 전략
  - `WaitForFirstConsumer`로 토폴로지와 일치시키기
- 리클레임 정책
  - 데이터 보존 필요 시 `Retain`, 그렇지 않으면 `Delete`
- 확장/스냅샷
  - `allowVolumeExpansion`, VolumeSnapshotClass 준비
- 권한/보안
  - `runAsUser`/`fsGroup`/SELinux, Secret로 자격증명 관리
- 운영 자동화
  - 백업/복구 절차 문서화, 관측/알람(Attach/Mount 실패, PVC Pending)

---

## 부록: 빈번히 쓰는 에피hemeral 볼륨들 개요

| 타입 | 요약 | 주의점 |
|---|---|---|
| `emptyDir` | Pod 생명주기와 동일한 임시 저장소 | 노드 장애 시 손실, `medium: Memory` 설정 가능 |
| `configMap` / `secret` | 설정/비밀 파일 주입 | 변경 전파 방식 이해, 대용량에 부적합 |
| `downwardAPI` | 리소스/라벨/필드 주입 | 환경변수/파일로 노출 |
| `projected` | 여러 소스를 단일 볼륨으로 | 각 소스의 제약을 함께 고려 |

---

## 참고 예제 묶음

### NFS PV/PVC/Pod 일괄

```yaml
---
apiVersion: v1
kind: PersistentVolume
metadata:
  name: nfs-pv
spec:
  capacity: { storage: 10Gi }
  accessModes: [ "ReadWriteMany" ]
  persistentVolumeReclaimPolicy: Retain
  nfs:
    server: 10.0.0.1
    path: /exports/app
---
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: nfs-pvc
spec:
  accessModes: [ "ReadWriteMany" ]
  resources:
    requests:
      storage: 10Gi
---
apiVersion: v1
kind: Pod
metadata:
  name: nfs-test
spec:
  containers:
  - name: app
    image: alpine:3.20
    command: ["sh","-c","echo hello > /data/hello.txt && sleep 3600"]
    volumeMounts:
    - name: data
      mountPath: /data
  volumes:
  - name: data
    persistentVolumeClaim:
      claimName: nfs-pvc
```

### StorageClass 기반 동적 PVC

```yaml
---
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: gp3-wffc
provisioner: ebs.csi.aws.com
parameters:
  type: gp3
reclaimPolicy: Delete
allowVolumeExpansion: true
volumeBindingMode: WaitForFirstConsumer
---
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: pvc-gp3
spec:
  storageClassName: gp3-wffc
  accessModes: [ "ReadWriteOnce" ]
  resources:
    requests:
      storage: 20Gi
```

---

## 결론

Kubernetes 스토리지는 **애플리케이션의 일시성**과 **데이터의 영속성** 사이의 간극을 메운다.
핵심은 다음과 같다.

- 워크로드의 **접근 모드**와 **성능 특성**을 먼저 정하고, 그에 맞는 드라이버/StorageClass를 선택한다.
- 멀티존/멀티노드에서는 `WaitForFirstConsumer`로 올바른 토폴로지에 바인딩한다.
- 권한/보안 컨텍스트와 리클레임/스냅샷/확장 전략을 **처음부터** 설계에 포함한다.
