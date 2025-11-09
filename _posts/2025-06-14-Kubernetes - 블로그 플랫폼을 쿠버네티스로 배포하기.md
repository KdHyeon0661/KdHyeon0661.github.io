---
layout: post
title: Kubernetes - 블로그 플랫폼을 쿠버네티스로 배포하기
date: 2025-06-14 19:20:23 +0900
category: Kubernetes
---
# 실습: 블로그 플랫폼을 쿠버네티스로 배포하기

## 1. 준비

```bash
minikube start --driver=docker
minikube addons enable ingress
```

선택: 기본 스토리지 클래스 확인

```bash
kubectl get storageclass
```

> 고성능이 필요하면 SSD 계열 StorageClass(예: `standard-rwo`, `gp3`, `premium`)를 지정합니다.

---

## 2. 아키텍처

```text
[Client] → [Ingress(nginx)] → [Service/ghost] → [Pod/ghost]
                                  │
                                  └→ DB 연결 (Service/mysql) → [Pod/mysql] ↕ PVC(mysql-pvc)

[PVC/ghost-content]  ←  /var/lib/ghost/content  (이미지/테마)
```

---

## 3. MySQL — Stateful 구성과 보안

### 3.1 MySQL 매니페스트 (PVC/Secret/Deployment/Service)

`mysql-deployment.yaml`

```yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: mysql-pvc
spec:
  accessModes: [ "ReadWriteOnce" ]
  resources:
    requests:
      storage: 5Gi
  # storageClassName: "standard"   # 필요 시 명시
---
apiVersion: v1
kind: Secret
metadata:
  name: mysql-secret
type: Opaque
data:
  password: bXlzcWw= # 'mysql'
  user: bXl1c2Vy     # 'myuser'
  database: Z2hvc3Q= # 'ghost'
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: mysql
spec:
  replicas: 1
  selector:
    matchLabels: { app: mysql }
  template:
    metadata:
      labels: { app: mysql }
    spec:
      containers:
        - name: mysql
          image: mysql:8.0
          ports:
            - containerPort: 3306
          env:
            - name: MYSQL_ROOT_PASSWORD
              valueFrom: { secretKeyRef: { name: mysql-secret, key: password } }
            - name: MYSQL_DATABASE
              valueFrom: { secretKeyRef: { name: mysql-secret, key: database } }
            - name: MYSQL_USER
              valueFrom: { secretKeyRef: { name: mysql-secret, key: user } }
            - name: MYSQL_PASSWORD
              valueFrom: { secretKeyRef: { name: mysql-secret, key: password } }
          args:
            - --character-set-server=utf8mb4
            - --collation-server=utf8mb4_0900_ai_ci
            - --default-authentication-plugin=mysql_native_password
          volumeMounts:
            - name: mysql-storage
              mountPath: /var/lib/mysql
          readinessProbe:
            exec:
              command: ["bash","-lc","mysqladmin ping -h 127.0.0.1 -u$MYSQL_USER -p$MYSQL_PASSWORD || exit 1"]
            initialDelaySeconds: 10
            periodSeconds: 5
          livenessProbe:
            tcpSocket: { port: 3306 }
            initialDelaySeconds: 30
            periodSeconds: 10
          resources:
            requests: { cpu: "250m", memory: "512Mi" }
            limits: { cpu: "1", memory: "1Gi" }
      volumes:
        - name: mysql-storage
          persistentVolumeClaim:
            claimName: mysql-pvc
---
apiVersion: v1
kind: Service
metadata:
  name: mysql
spec:
  selector: { app: mysql }
  ports:
    - port: 3306
      targetPort: 3306
```

적용:

```bash
kubectl apply -f mysql-deployment.yaml
kubectl rollout status deploy/mysql
```

> MySQL 8.0 권장(보안/성능). 로캘, 인코딩을 미리 지정해 문자열 검색/이모지 등 문제를 예방합니다.

---

## 4. Ghost — 콘텐츠 영속화, URL, 헬스 프로브

Ghost는 `/var/lib/ghost/content` 아래에 **이미지/테마/SQLite(미사용)/설정 일부**를 저장합니다. **PVC**를 붙여 영속화합니다.

### 4.1 Ghost ConfigMap/Secret

`ghost-config.yaml`

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: ghost-config
data:
  url: "http://blog.local"  # Ingress 호스트와 일치해야 합니다.
  mail__transport: "Direct" # 필요 시 SMTP로 변경
  # SMTP 예시:
  # mail__transport: "SMTP"
  # mail__options__service: "Mailgun"
  # mail__options__auth__user: "postmaster@domain"
  # mail__options__auth__pass: "********"
---
apiVersion: v1
kind: Secret
metadata:
  name: ghost-secret
type: Opaque
data:
  db_user: bXl1c2Vy     # 'myuser'
  db_password: bXlzcWw= # 'mysql'
  db_name: Z2hvc3Q=     # 'ghost'
```

적용:

```bash
kubectl apply -f ghost-config.yaml
```

### 4.2 Ghost PVC + Deployment + Service

`ghost-deployment.yaml`

```yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: ghost-content-pvc
spec:
  accessModes: [ "ReadWriteOnce" ]
  resources:
    requests:
      storage: 5Gi
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: ghost
spec:
  replicas: 1
  selector:
    matchLabels: { app: ghost }
  template:
    metadata:
      labels: { app: ghost }
    spec:
      containers:
        - name: ghost
          image: ghost:5
          ports:
            - containerPort: 2368
          env:
            - name: url
              valueFrom: { configMapKeyRef: { name: ghost-config, key: url } }
            - name: database__client
              value: mysql
            - name: database__connection__host
              value: mysql
            - name: database__connection__user
              valueFrom: { secretKeyRef: { name: ghost-secret, key: db_user } }
            - name: database__connection__password
              valueFrom: { secretKeyRef: { name: ghost-secret, key: db_password } }
            - name: database__connection__database
              valueFrom: { secretKeyRef: { name: ghost-secret, key: db_name } }
            # 메일/스토리지/S3 연동 등은 ConfigMap/Secret로 추가 가능
          volumeMounts:
            - name: ghost-content
              mountPath: /var/lib/ghost/content
          readinessProbe:
            httpGet:
              path: /
              port: 2368
            initialDelaySeconds: 15
            periodSeconds: 5
            timeoutSeconds: 2
          livenessProbe:
            httpGet:
              path: /
              port: 2368
            initialDelaySeconds: 60
            periodSeconds: 10
            timeoutSeconds: 2
          resources:
            requests: { cpu: "200m", memory: "256Mi" }
            limits: { cpu: "1", memory: "1Gi" }
      volumes:
        - name: ghost-content
          persistentVolumeClaim:
            claimName: ghost-content-pvc
---
apiVersion: v1
kind: Service
metadata:
  name: ghost-service
spec:
  selector: { app: ghost }
  ports:
    - port: 80
      targetPort: 2368
      protocol: TCP
```

적용:

```bash
kubectl apply -f ghost-deployment.yaml
kubectl rollout status deploy/ghost
```

> `url` 환경변수는 Ghost의 **Canonical URL**로 동작합니다. Ingress 호스트와 정확히 일치해야 이미지/링크/세션에 문제 없습니다.

---

## 5. Ingress — 호스트 라우팅 및 헤더

`ghost-ingress.yaml`

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: ghost-ingress
  annotations:
    nginx.ingress.kubernetes.io/rewrite-target: /
    # 보안/성능 헤더 예시
    nginx.ingress.kubernetes.io/proxy-body-size: "20m"
    nginx.ingress.kubernetes.io/server-snippet: |
      add_header X-Frame-Options SAMEORIGIN always;
      add_header X-Content-Type-Options nosniff always;
      add_header Referrer-Policy strict-origin-when-cross-origin always;
spec:
  ingressClassName: nginx
  rules:
    - host: blog.local
      http:
        paths:
          - path: /
            pathType: Prefix
            backend:
              service:
                name: ghost-service
                port:
                  number: 80
```

적용:

```bash
kubectl apply -f ghost-ingress.yaml
```

호스트 등록:

```text
127.0.0.1 blog.local
```

브라우저: `http://blog.local`

---

## 6. 운영 확장: 마이그레이션, 초기 관리자, 이미지 업로드 경로

- **초기 관리자**: Ghost는 첫 접속 시 어드민 비번 생성 페이지가 뜹니다. 동시성 문제를 피하려면 접근 제어(임시 Basic Auth)나 유지보수 윈도우 권장.
- **이미지 업로드**: `/var/lib/ghost/content/images`에 저장. **PVC 크기/백업 주기** 계획 필수.
- **테마 배포**: `content/themes` 디렉터리를 CI에서 빌드 후 동기화하거나, 별도의 사이드카/InitContainer로 테마 아카이브를 주입하는 패턴.

### 테마 InitContainer 예시

```yaml
spec:
  template:
    spec:
      initContainers:
        - name: theme-init
          image: busybox:1.36
          command: ["sh","-c"]
          args:
            - |
              mkdir -p /var/lib/ghost/content/themes/mytheme
              # 예: ConfigMap/emptyDir로 제공한 테마 아카이브를 풀거나, 사내 아티팩트에서 wget
              # tar -xf /extras/mytheme.tar -C /var/lib/ghost/content/themes/mytheme
          volumeMounts:
            - name: ghost-content
              mountPath: /var/lib/ghost/content
      containers:
        - name: ghost
          ...
```

---

## 7. 백업/복구 및 유지보수

### 7.1 DB 백업 Job (mysqldump)

`mysql-backup-cronjob.yaml`

```yaml
apiVersion: v1
kind: Secret
metadata:
  name: mysql-backup-secret
type: Opaque
data:
  password: bXlzcWw=
  user: bXl1c2Vy
  database: Z2hvc3Q=
---
apiVersion: batch/v1
kind: CronJob
metadata:
  name: mysql-backup
spec:
  schedule: "0 3 * * *"  # 매일 03:00
  jobTemplate:
    spec:
      template:
        spec:
          restartPolicy: OnFailure
          containers:
            - name: dump
              image: mysql:8.0
              env:
                - name: MYSQL_USER
                  valueFrom: { secretKeyRef: { name: mysql-backup-secret, key: user } }
                - name: MYSQL_PASSWORD
                  valueFrom: { secretKeyRef: { name: mysql-backup-secret, key: password } }
                - name: MYSQL_DATABASE
                  valueFrom: { secretKeyRef: { name: mysql-backup-secret, key: database } }
              command: ["bash","-lc"]
              args:
                - |
                  set -e
                  TS=$(date +%F-%H%M%S)
                  mysqldump -h mysql -u$MYSQL_USER -p$MYSQL_PASSWORD $MYSQL_DATABASE > /backup/ghost-$TS.sql
                  ls -lh /backup
              volumeMounts:
                - name: backup
                  mountPath: /backup
          volumes:
            - name: backup
              persistentVolumeClaim:
                claimName: mysql-pvc   # 간단히 동일 PVC에 저장(실운영은 별도 백업용 PVC/외부 스토리지 권장)
```

적용/로그:

```bash
kubectl apply -f mysql-backup-cronjob.yaml
kubectl get cronjob
```

복구는 `mysql` 컨테이너 내부에서 `mysql -u... -p... ghost < dump.sql` 로 진행합니다.

### 7.2 Ghost 콘텐츠 백업

- PVC 스냅샷(스토리지 클래스가 지원 시)
- `kubectl cp`로 `/var/lib/ghost/content` 백업
- 외부 오브젝트 스토리지(S3) 동기화(예: rclone sidecar)

---

## 8. 보안: TLS, 인증, 비밀 관리

### 8.1 cert-manager + Let’s Encrypt

`ClusterIssuer`(http01) 생성:

```yaml
apiVersion: cert-manager.io/v1
kind: ClusterIssuer
metadata:
  name: letsencrypt-http01
spec:
  acme:
    email: you@example.com
    server: https://acme-v02.api.letsencrypt.org/directory
    privateKeySecretRef: { name: acme-account-key }
    solvers:
      - http01:
          ingress:
            class: nginx
```

Ingress에 주석 추가:

```yaml
metadata:
  annotations:
    cert-manager.io/cluster-issuer: letsencrypt-http01
spec:
  tls:
    - hosts: [ "blog.local" ] # 실제 도메인 필요
      secretName: ghost-tls
```

> 실도메인을 사용할 것. Minikube의 `blog.local`은 퍼블릭 인증서를 발급받을 수 없습니다(자체 CA 또는 mkcert 사용).

### 8.2 어드민 보호

- NGINX Ingress `auth`(basic auth) 또는 OIDC 리버스 프록시(Keycloak/Gatekeeper, oauth2-proxy)로 `/ghost` 경로 보호.
- 보안 헤더(HSTS, CSP) 추가.

---

## 9. 확장성: HPA, PDB, Anti-Affinity

### 9.1 HPA (CPU 기준)

```bash
kubectl autoscale deployment ghost --cpu-percent=60 --min=1 --max=5
kubectl get hpa
```

> Ghost는 stateful 콘텐츠를 PVC로 공유(단일 RWX가 아닌 RWO)하므로 **복수 레플리카 시 공유 스토리지 고려**가 필요합니다. NFS/RWX 클래스 또는 로드밸런서/외부 저장소 연동으로 정적 파일을 공용화(예: S3 저장 플러그인)하는 패턴이 일반적입니다.

### 9.2 PDB

`pdb-ghost.yaml`

```yaml
apiVersion: policy/v1
kind: PodDisruptionBudget
metadata:
  name: ghost-pdb
spec:
  minAvailable: 1
  selector:
    matchLabels: { app: ghost }
```

### 9.3 Anti-Affinity (멀티노드 시)

```yaml
spec:
  template:
    spec:
      affinity:
        podAntiAffinity:
          preferredDuringSchedulingIgnoredDuringExecution:
            - weight: 50
              podAffinityTerm:
                labelSelector:
                  matchExpressions:
                    - key: app
                      operator: In
                      values: ["ghost"]
                topologyKey: "kubernetes.io/hostname"
```

---

## 10. 관측: 로그/메트릭/대시보드

- **로그**: `kubectl logs deploy/ghost -f`
- **리소스**: `kubectl top pods`, `kubectl top nodes` (metrics-server)
- **애플리케이션 메트릭**: Ghost는 기본 Prometheus 지표를 내지 않습니다. NGINX Ingress, 노드, MySQL(JMX Exporter 또는 mysqld-exporter)로 **간접 지표** 확보.
- **대시보드**: Lens/K9s로 리소스 관측. Prometheus/Grafana 스택을 추가해 인프라 레벨 메트릭을 시각화.

---

## 11. GitOps/Helm로 패키징

- Directory 구조 예:

```text
k8s/
  base/
    mysql-deployment.yaml
    ghost-config.yaml
    ghost-deployment.yaml
    ghost-ingress.yaml
  overlays/prod/...
  overlays/dev/...
```

- Helm Chart로 파라미터화:

  - `values.yaml`에 `ghost.url`, `mysql.password`, `resources`, `persistence.size` 등 정의
  - Argo CD/Flux로 **Git → Cluster 동기화** 적용

---

## 12. 트러블슈팅

| 증상 | 원인 후보 | 조치 |
|---|---|---|
| `Pending` | PVC 바인딩 실패/스토리지 없음 | `kubectl get pvc`, StorageClass 확인 |
| Ghost 502/504 | `url` 불일치, Ingress 설정 누락 | `url`=`http(s)://<host>` 일치, Ingress rule 점검 |
| DB 연결 실패 | Secret 오타, MySQL 준비 전 Ghost 시작 | readinessProbe, `initContainers`로 DB 체크, env 매칭 |
| 로그인 루프 | `url` 스킴/도메인 불일치 | `url`을 실제 접속 URL로 수정 후 재시작 |
| 업로드 실패 | PVC 용량 부족/권한 | PVC 확장, 권한/보안컨텍스트 점검 |
| 느린 응답 | 리소스 부족/이미지 최적화 필요 | requests/limits 상향, 이미지 압축/캐시 |

### DB 준비를 기다리는 initContainers 예시

```yaml
initContainers:
  - name: wait-mysql
    image: busybox:1.36
    command: ["sh","-c"]
    args:
      - |
        until nc -zv mysql 3306; do echo "waiting mysql"; sleep 2; done
```

---

## 13. 삭제

```bash
kubectl delete -f ghost-ingress.yaml
kubectl delete -f ghost-deployment.yaml
kubectl delete -f ghost-config.yaml
kubectl delete -f mysql-deployment.yaml
kubectl delete -f mysql-backup-cronjob.yaml
kubectl delete hpa ghost
kubectl delete pdb ghost-pdb
```

> PVC는 남아 **데이터 보존**됩니다. 완전 제거 시 PVC/PV도 삭제하세요.

---

## 14. 전체 매니페스트 요약

- `mysql-deployment.yaml` — PVC/Secret/Deployment/Service
- `ghost-config.yaml` — ConfigMap/Secret(앱 설정)
- `ghost-deployment.yaml` — Ghost PVC/Deployment/Service(프로브/리소스)
- `ghost-ingress.yaml` — Ingress(보안 헤더/리라이트)
- `mysql-backup-cronjob.yaml` — DB 백업 스케줄(옵션)
- `pdb-ghost.yaml` — Ghost 가용성 정책(옵션)

---

## 정리

| 항목 | 요약 |
|---|---|
| 영속화 | Ghost 콘텐츠 PVC, MySQL PVC |
| 안정성 | readiness/liveness, `initContainers`, PDB |
| 보안 | Secret 분리, TLS(cert-manager), 보안 헤더, 어드민 보호 |
| 확장 | HPA(콘텐츠 공유 전략과 함께), Anti-Affinity |
| 운영 | 백업/복구, 스토리지 클래스/용량 계획, 관측 지표 |
| 지속 배포 | Helm/GitOps로 파라미터화 및 자동 동기화 |

해당 구성을 바탕으로 **프로덕션 수준**의 블로그 서비스를 클러스터 안에서 안정적으로 운영할 수 있습니다. 필요 시 CDN, 오브젝트 스토리지(S3) 플러그인, 외부 이메일/SMS 인증, 이미지 프록시, 캐시 계층(Cloudflare/NGINX 캐시)을 단계적으로 더하면 됩니다.