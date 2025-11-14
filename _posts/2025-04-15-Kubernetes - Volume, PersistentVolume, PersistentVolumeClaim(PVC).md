---
layout: post
title: Kubernetes - Volume, PersistentVolume, PersistentVolumeClaim(PVC)
date: 2025-04-11 19:20:23 +0900
category: Kubernetes
---
# Volume, PersistentVolume, PersistentVolumeClaim(PVC)

## 왜 Volume이 필요한가

컨테이너와 Pod의 파일시스템은 기본적으로 일시적이다. 재스케줄링, 롤링 업데이트, 노드 장애가 발생하면 내부 데이터는 사라질 수 있다. 데이터베이스, 사용자 업로드, 장기 로그 등은 Pod의 생명주기와 분리된 **영속성**이 요구된다.

이를 위해 Kubernetes는 다음을 제공한다.

- Ephemeral Volumes: Pod 수명에 종속되는 임시 저장소(예: `emptyDir`, `configMap`, `secret`)
- Persistent Volumes: 클러스터 리소스로서 관리되는 **영속 저장소**(PV/PVC)

---

## Volume의 큰 그림

```
Pod ───> PVC(요청자) ─── 바인딩 ───> PV(공급자) ───> 실제 스토리지(NFS/블록/파일/CSI)
```

- Pod는 PVC를 참조하여 마운트한다.
- PVC는 요청 사양(용량, 접근 모드 등)에 맞는 PV와 바인딩된다.
- PV는 실제 스토리지(클라우드 디스크, NFS, 파일 스토리지, CSI 드라이버)와 연결된다.

---

## Volume 종류 요약

| 종류 | 수명 | 대표 사용처 | 주의점 |
|---|---|---|---|
| `emptyDir` | Pod와 동일 | 임시 캐시, 빌드 중간 산출물 | 노드/Pod 재시작 시 데이터 소멸, `medium: Memory` 가능 |
| `hostPath` | 노드 수명 | 로컬 실습/디버깅 | 개발용 권장, 노드 결합/보안 위험 |
| `configMap`/`secret` | Pod와 동일 | 설정/자격 비밀 주입 | 대용량/빈번 변경에 비적합, 변경 전파 방식 이해 필요 |
| `persistentVolumeClaim` | Pod와 독립 | 영속 데이터(업로드, DB 등) | PV/PVC/SC의 정합성 설계 필요 |

---

## PersistentVolume(PV)와 PersistentVolumeClaim(PVC)

### PV 핵심 필드

- `capacity.storage`: 크기
- `accessModes`: 접근 모드(RWO/ROX/RWX/RWO-Pod)
- `persistentVolumeReclaimPolicy`: 삭제 정책(Retain/Delete)
- 백엔드 스펙: `nfs`, `csi`, `gcePersistentDisk`, `awsElasticBlockStore` 등

### PVC 핵심 필드

- `resources.requests.storage`: 요청 크기
- `accessModes`: 필요 접근 모드
- `storageClassName`: 동적 프로비저닝 시 사용
- `dataSource`: 스냅샷/클론 복원 시 사용

---

## 접근 모드(Access Modes) 설계

| 모드 | 의미 | 전형적 백엔드 | 비고 |
|---|---|---|---|
| RWO (ReadWriteOnce) | 한 노드에서 하나의 Pod만 R/W | 블록(EBS/PD/Azure Disk) | 다수 노드 동시 접근 불가 |
| ROX (ReadOnlyMany) | 여러 노드에서 ReadOnly | 파일(NFS 등) | 공유 읽기 |
| RWX (ReadWriteMany) | 여러 노드에서 R/W | 파일(NFS, EFS, Azure Files, Filestore) | 다중 Pod 공유 쓰기 |
| RWO-Pod (ReadWriteOncePod) | 한 Pod만 R/W | 드라이버 지원 필요 | 격리 강화 |

실무 기준
- 단일 인스턴스 DB: RWO
- 여러 Pod가 공유하는 정적 콘텐츠: RWX
- 멀티존 환경: 토폴로지와 `volumeBindingMode` 고려

---

## 리클레임 정책(Reclaim Policy)

| 정책 | 설명 | 사용 맥락 |
|---|---|---|
| `Retain` | PVC 삭제 후에도 PV와 데이터 유지 | 중요 데이터 보호, 수동 정리 필요 |
| `Delete` | PVC 삭제 시 PV/실제 디스크 제거 | 테스트/임시 워크로드 자동 청소 |
| `Recycle` | 구식(미사용 권장) | 사용하지 않음 |

데이터 보존 요건이 강하면 `Retain`을 우선 검토한다.

---

## 예제 1: 수동 PV/PVC/Pod (hostPath는 학습용)

> 실무에선 `hostPath` 대신 NFS/CSI/클라우드 디스크를 사용한다. 여기선 개념 확인용 최소 예제를 제시한다.

```yaml
apiVersion: v1
kind: PersistentVolume
metadata:
  name: pv-example
spec:
  capacity:
    storage: 1Gi
  accessModes: [ "ReadWriteOnce" ]
  persistentVolumeReclaimPolicy: Retain
  hostPath:
    path: "/mnt/data"    # 데모용, 운영 비권장
---
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: pvc-example
spec:
  accessModes: [ "ReadWriteOnce" ]
  resources:
    requests:
      storage: 1Gi
---
apiVersion: v1
kind: Pod
metadata:
  name: pod-with-pvc
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

적용과 확인:
```bash
kubectl apply -f pv-pvc-pod-demo.yaml
kubectl get pv,pvc,pod
kubectl describe pvc pvc-example
kubectl exec -it pod-with-pvc -- sh -c 'echo ok > /usr/share/nginx/html/health && ls -l /usr/share/nginx/html'
```

---

## 예제 2: NFS 기반 PV/PVC (RWX 공유)

여러 Pod가 동시에 읽기/쓰기를 해야 한다면 RWX가 필요하며, 파일 스토리지(NFS/EFS/Azure Files 등)가 적합하다.

```yaml
apiVersion: v1
kind: PersistentVolume
metadata:
  name: nfs-pv
spec:
  capacity: { storage: 10Gi }
  accessModes: [ "ReadWriteMany" ]
  persistentVolumeReclaimPolicy: Retain
  mountOptions: [ "nfsvers=4.1" ]
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
kind: Deployment
metadata:
  name: web
spec:
  replicas: 3
  selector: { matchLabels: { app: web } }
  template:
    metadata:
      labels: { app: web }
    spec:
      containers:
      - name: nginx
        image: nginx:1.27-alpine
        volumeMounts:
        - name: shared
          mountPath: /usr/share/nginx/html
      volumes:
      - name: shared
        persistentVolumeClaim:
          claimName: nfs-pvc
```

---

## 동적 프로비저닝(StorageClass)와 WaitForFirstConsumer

PVC가 생성되면 StorageClass/CSI가 자동으로 PV를 만들고 바인딩한다. 멀티존 환경에서는 `WaitForFirstConsumer`로 스케줄링된 노드의 토폴로지를 참고해 올바른 존의 볼륨을 생성하도록 한다.

```yaml
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

검증:
```bash
kubectl apply -f sc-pvc.yaml
kubectl get sc
kubectl get pvc pvc-gp3 -w
```

---

## 퍼미션과 보안 컨텍스트(runAsUser, fsGroup, SELinux)

루트가 아닌 사용자로 컨테이너를 실행할 때, 마운트된 볼륨에 쓰기 권한이 없으면 실패한다. 다음을 조정한다.

- `securityContext.runAsUser` / `runAsGroup`
- `securityContext.fsGroup` : 마운트 지점의 그룹 퍼미션 부여
- SELinux 환경에서는 컨텍스트 맞춤 필요

예시:
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
      claimName: pvc-gp3
```

---

## subPath로 단일 파일/하위 디렉터리 마운트

설정 파일 하나만 마운트하고 싶을 때 유용하다. 단, 실시간 변경이 제한될 수 있다.

```yaml
volumeMounts:
- name: conf
  mountPath: /etc/myapp/config.yaml
  subPath: config.yaml
  readOnly: true
```

---

## StatefulSet과 volumeClaimTemplates

상태 저장 워크로드(예: DB, 메시지 큐)는 Pod마다 고유 PVC가 필요하다. StatefulSet은 이를 자동 생성한다.

```yaml
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: pg
spec:
  serviceName: pg
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
      storageClassName: gp3-wffc
      accessModes: [ "ReadWriteOnce" ]
      resources: { requests: { storage: 20Gi } }
```

---

## VolumeSnapshot으로 시점 백업/복구

CSI 스냅샷 CRDs가 설치되어 있어야 한다.

```yaml
apiVersion: snapshot.storage.k8s.io/v1
kind: VolumeSnapshotClass
metadata:
  name: csi-snapclass
driver: ebs.csi.aws.com
deletionPolicy: Delete
---
apiVersion: snapshot.storage.k8s.io/v1
kind: VolumeSnapshot
metadata:
  name: data-snap
spec:
  volumeSnapshotClassName: csi-snapclass
  source:
    persistentVolumeClaimName: pvc-gp3
---
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: pvc-from-snap
spec:
  storageClassName: gp3-wffc
  dataSource:
    name: data-snap
    kind: VolumeSnapshot
    apiGroup: snapshot.storage.k8s.io
  accessModes: [ "ReadWriteOnce" ]
  resources:
    requests: { storage: 20Gi }
```

---

## 볼륨 확장(Resize)

StorageClass에서 `allowVolumeExpansion: true`가 필요하다. 드라이버/파일시스템에 따라 온라인 확장 가능 여부가 다르다.

PVC 확장:
```bash
kubectl patch pvc pvc-gp3 -p '{"spec":{"resources":{"requests":{"storage":"40Gi"}}}}'
kubectl get pvc pvc-gp3
```

컨테이너 안에서 확인:
```bash
kubectl exec -it <pod> -- df -h
```

---

## 관측·운영 명령어

```bash
# 나열

kubectl get sc
kubectl get pv
kubectl get pvc
kubectl get volumesnapshotclass
kubectl get volumesnapshot

# 상세

kubectl describe sc <name>
kubectl describe pv <name>
kubectl describe pvc <name>
kubectl describe volumesnapshot <name>

# 이벤트/로그 중심 진단

kubectl describe pvc <name> | sed -n '/Events/,$p'
kubectl describe pod <name> | sed -n '/Events/,$p'

# 컨테이너 내부 상태

kubectl exec -it <pod> -- ls -al /mountpoint
kubectl exec -it <pod> -- df -h

# 삭제

kubectl delete pvc <name>
kubectl delete pv <name>
```

---

## 트러블슈팅 표

| 증상 | 1차 확인 | 원인 | 해결 |
|---|---|---|---|
| PVC가 Pending | `describe pvc` 이벤트 | 매칭 PV 없음, SC 누락, 접근 모드/크기 불일치 | PVC 스펙/SC 기본값/접근 모드 재설계 |
| Pod가 Scheduling 실패 | `describe pod` 스케줄링 이벤트 | `WaitForFirstConsumer` 토폴로지 불일치, 자원 부족 | 노드풀/존 확인, 리소스 요청 조정 |
| Pod가 ContainerCreating에서 멈춤 | Mount/Attach 이벤트 | CSI 컨트롤러/노드 플러그인 장애, IAM/네트워크 문제 | CSI 로그 점검, 권한/네트워크 복구 |
| EACCES 권한 오류 | 컨테이너 로그/`id`/퍼미션 | runAsUser/fsGroup 미설정, SELinux 컨텍스트 | `fsGroup`/`runAsUser` 설정, 보안 컨텍스트 조정 |
| RWX 필요하나 불가 | PVC/SC 확인 | 블록 스토리지(RWO만) 사용 | 파일 스토리지(NFS/EFS/Filestore/Azure Files)로 전환 |
| 확장 실패 | PVC 이벤트 | 드라이버 미지원, 오프라인 요구 | 드라이버 기능 확인, 재시작/언마운트 후 재시도 |

빠른 점검 스니펫:
```bash
kubectl get pvc <name> -o wide
kubectl get pv | grep <claimRef.name> -n
kubectl get sc <sc> -o yaml | sed -n '1,120p'
kubectl get events --sort-by=.lastTimestamp | tail -n 20
```

---

## 설계 체크리스트

- 접근 모드와 워크로드 패턴 정의
  - 단일 Pod R/W(RWO)인지, 다중 Pod 공유(RWX)인지
- 토폴로지/바인딩 전략
  - 멀티존/가용성 요구 시 `WaitForFirstConsumer`
- 리클레임 정책
  - 보호 필요 시 `Retain`, 임시/테스트는 `Delete`
- 확장/백업
  - `allowVolumeExpansion`, VolumeSnapshotClass 사전 준비
- 권한/보안
  - `runAsUser`/`fsGroup`/SELinux, Secret로 자격증명 분리
- 모니터링/알람
  - Attach/Mount 실패, PVC Pending 지속, IOPS/지연 관측
- 비용/성능
  - IOPS/처리량/지연 요구를 스토리지 타입과 매칭

---

## 통합 예제: Nginx + 동적 PVC + Ingress

실전 스캐폴딩으로 바로 테스트 가능하다.

```yaml
---
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: standard-wffc
provisioner: ebs.csi.aws.com         # 환경에 맞춰 변경
parameters:
  type: gp3
reclaimPolicy: Delete
allowVolumeExpansion: true
volumeBindingMode: WaitForFirstConsumer
---
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: web-pvc
spec:
  storageClassName: standard-wffc
  accessModes: [ "ReadWriteOnce" ]
  resources:
    requests:
      storage: 2Gi
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: web
spec:
  replicas: 2
  selector: { matchLabels: { app: web } }
  template:
    metadata:
      labels: { app: web }
    spec:
      securityContext:
        fsGroup: 2000
      containers:
      - name: nginx
        image: nginx:1.27-alpine
        ports: [{ containerPort: 80 }]
        volumeMounts:
        - name: html
          mountPath: /usr/share/nginx/html
      volumes:
      - name: html
        persistentVolumeClaim:
          claimName: web-pvc
---
apiVersion: v1
kind: Service
metadata:
  name: web-svc
spec:
  type: NodePort
  selector: { app: web }
  ports:
  - port: 80
    targetPort: 80
    nodePort: 30080
```

적용 후 확인:
```bash
kubectl apply -f web-complete.yaml
kubectl rollout status deploy/web
kubectl get pvc web-pvc
kubectl get svc web-svc
```

컨텐츠 쓰기 확인:
```bash
kubectl exec -it deploy/web -- sh -c 'echo "<h1>OK</h1>" > /usr/share/nginx/html/index.html'
curl http://<노드IP>:30080
```

---

## 결론

- Volume은 Pod의 일시성과 데이터의 영속성 사이 간극을 메운다.
- PV/PVC/StorageClass를 통해 공급자-요청자 모델을 확립하고, 접근 모드와 리클레임 정책을 요구사항에 맞춰 설계한다.
- 동적 프로비저닝과 `WaitForFirstConsumer`는 멀티존/운영 환경에서 안전한 바인딩을 돕는다.
- 권한(fsGroup/runAsUser), 스냅샷, 확장, 트러블슈팅 루틴을 초기 설계에 포함하면 장애 대응력이 크게 향상된다.

---

## 참고 명령 요약

```bash
# 상태

kubectl get sc
kubectl get pv
kubectl get pvc
kubectl get events --sort-by=.lastTimestamp | tail

# 상세/디버깅

kubectl describe pvc <name>
kubectl describe pv <name>
kubectl exec -it <pod> -- df -h
kubectl exec -it <pod> -- ls -al /mountpoint
```
