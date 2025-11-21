---
layout: post
title: Kubernetes - Job & CronJob
date: 2025-04-11 19:20:23 +0900
category: Kubernetes
---
# Kubernetes 핵심 오브젝트 이해하기

## Job — 일회성 작업을 “성공 개수”로 보장

### 기본 개념(정확히 이해하기)

- **Job = “성공한 Pod 수를 목표치까지 만들면 종료”**
  실패한 Pod는 `backoffLimit` 한도 내에서 **새 Pod로 재시도**한다.
- 완료된 Job은 **상태(완료/실패)와 Pod 이력**이 남는다(자동 정리 옵션: `ttlSecondsAfterFinished`).

### 최소 예제(Hello)

```yaml
apiVersion: batch/v1
kind: Job
metadata:
  name: hello-job
spec:
  template:
    spec:
      containers:
      - name: hello
        image: busybox:1.36
        command: ["sh","-c","echo 'Hello Kubernetes'"]
      restartPolicy: Never
```

### 핵심 파라미터

```yaml
spec:
  completions: 3          # 총 "성공"해야 하는 Pod 수
  parallelism: 2          # 동시에 띄울 Pod 수(병렬도)
  backoffLimit: 4         # 실패 시 새 Pod로 재시도 가능한 횟수
  activeDeadlineSeconds: 600   # 전체 작업 타임아웃(초)
  ttlSecondsAfterFinished: 300 # 완료 후 5분 뒤 리소스 자동 정리
```

- `restartPolicy`: `Never` 또는 `OnFailure`만 허용.
- `backoffLimit`은 **Job 레벨 재시도 횟수**(실패 Pod를 새 Pod로 다시 실행).
- `activeDeadlineSeconds`: **전체 벽시계 제한**(무한 루프/장기 대기 방지).

### 병렬 Job(배치 처리의 기본기)

```yaml
apiVersion: batch/v1
kind: Job
metadata:
  name: batch-parallel
spec:
  completions: 10
  parallelism: 3
  template:
    spec:
      containers:
      - name: worker
        image: busybox:1.36
        command: ["sh","-c","echo do work; sleep 5"]
      restartPolicy: OnFailure
```
- 최대 3개씩 동시에 처리, 총 10회 **성공**하면 완료.

### 컴플리션 — 샤딩 처리

각 작업에 고유 인덱스를 부여하여 **입력 데이터 샤딩**에 활용한다.

```yaml
apiVersion: batch/v1
kind: Job
metadata:
  name: shard-indexed
spec:
  completions: 5
  parallelism: 2
  completionMode: Indexed   # 중요!
  template:
    spec:
      containers:
      - name: worker
        image: alpine:3.20
        command: ["sh","-c","echo index=$JOB_COMPLETION_INDEX; sleep 2"]
        env:
        - name: JOB_COMPLETION_INDEX
          valueFrom:
            fieldRef: { fieldPath: metadata.annotations['batch.kubernetes.io/job-completion-index'] }
      restartPolicy: OnFailure
```

> 인덱스 값은 각 Pod에 **고유**하게 주어지므로, 예컨대 S3 파티션 `shard-<index>`를 병렬 처리하기 쉽다.

### 패턴 — 동적 분배

입력 수를 모를 때는 `parallelism`만 설정하고, **공유 큐(예: Redis/SQS/Kafka)** 에서 일을 가져가며 끝낼 때까지 루프. Job은 **작업자 수**만 보장한다.

```yaml
apiVersion: batch/v1
kind: Job
metadata:
  name: workqueue
spec:
  parallelism: 5
  template:
    spec:
      containers:
      - name: worker
        image: ghcr.io/example/worker:1.0.0
        command: ["sh","-c","python /app/worker.py"]   # 큐에서 POP → 처리 → ACK
      restartPolicy: OnFailure
```

### — Pod Failure Policy(버전 의존)

클러스터 버전에 따라 **특정 종료 코드/상태에 대해 즉시 실패/재시도**를 세밀 제어하는 `podFailurePolicy`를 쓸 수 있다(지원 버전에서).

```yaml
spec:
  podFailurePolicy:
    rules:
    - action: FailJob
      onExitCodes:
        containerName: worker
        operator: In
        values: [137]   # 예: OOMKilled 신호로 실패 처리
```

---

## CronJob — 크론 스케줄에 따라 Job 생성

### 개념

- **스케줄러**가 Cron 표현식(`.spec.schedule`)을 기준으로 새로운 Job 오브젝트를 **주기 생성**한다.
- 생성된 Job의 성공/실패 이력을 **보존 개수**로 관리한다.

### 최소 예제(매분 실행)

```yaml
apiVersion: batch/v1
kind: CronJob
metadata:
  name: hello-cron
spec:
  schedule: "*/1 * * * *"
  jobTemplate:
    spec:
      template:
        spec:
          containers:
          - name: hello
            image: busybox:1.36
            command: ["sh","-c","date; echo hello from CronJob"]
          restartPolicy: OnFailure
```

### 중요한 필드들

```yaml
spec:
  schedule: "0 0 * * *"        # 매일 00:00
  concurrencyPolicy: Forbid     # Allow | Forbid | Replace
  successfulJobsHistoryLimit: 3
  failedJobsHistoryLimit: 1
  startingDeadlineSeconds: 300  # 스케줄 놓쳤을 때 허용 지연(초)
  suspend: false                # true면 스케줄 일시정지
  timeZone: "Asia/Seoul"        # 지원 버전에서 사용 가능(미지원이면 컨트롤러 시간대 기준)
  jobTemplate:
    spec:
      backoffLimit: 2
      activeDeadlineSeconds: 900
      template:
        spec:
          restartPolicy: OnFailure
          containers:
          - name: backup
            image: ghcr.io/example/backup:1.0.0
            args: ["--target=s3://bucket/daily/"]
```

- `concurrencyPolicy`
  - **Allow**: 겹쳐서 실행 가능(기본).
  - **Forbid**: 이전 실행이 남아 있으면 **새 실행 취소**.
  - **Replace**: 이전 실행 **중지** 후 새 실행.
- `startingDeadlineSeconds`: 컨트롤러 중단/지연 시 **“놓친 실행”을 얼마나까지 보상할지**.

> **시간대 주의**: 클러스터/배포판에 따라 CronJob 스케줄 **해석 시간대**가 다를 수 있다. `timeZone` 필드가 **지원되는 버전**이라면 명시하는 것을 권장한다. 미지원이면 일반적으로 **컨트롤러의 시스템 시간대**(관리형 환경에선 대개 UTC)로 스케줄된다.

---

## Job vs CronJob 요약

| 항목 | Job | CronJob |
|---|---|---|
| 목적 | 단일/유한 작업 완료 | 반복 작업 예약 |
| 트리거 | 즉시 생성 | Cron 스케줄에 따라 Job 생성 |
| 병렬 처리 | `parallelism`/`completions`/Indexed | jobTemplate 내부 Job 설정을 그대로 계승 |
| 실패/재시도 | `backoffLimit`/`activeDeadlineSeconds`/Failure Policy | 위와 동일(템플릿 내), 추가로 `concurrencyPolicy`/`startingDeadlineSeconds` |
| 정리 | `ttlSecondsAfterFinished` | 이력 보존 개수 + Job 자체 TTL |

---

## 운영 실습 흐름(명령 모음)

### Job

```bash
kubectl apply -f hello-job.yaml
kubectl get jobs
kubectl get pods -l job-name=hello-job
kubectl logs job/hello-job
kubectl describe job hello-job
```

### CronJob

```bash
kubectl apply -f hello-cron.yaml
kubectl get cronjobs
kubectl get jobs --watch   # 스케줄 시 새 Job 생성되는지 관찰
kubectl logs -l job-name=hello-cron-... --tail=100
kubectl describe cronjob hello-cron
```

---

## 리소스/스케줄링/보안 베스트 프랙티스

### 리소스/스케줄링

```yaml
spec:
  template:
    spec:
      nodeSelector: { "kubernetes.io/os": linux }
      affinity:
        nodeAffinity:
          requiredDuringSchedulingIgnoredDuringExecution:
            nodeSelectorTerms:
            - matchExpressions:
              - key: batch-node
                operator: In
                values: ["true"]
      tolerations:
      - key: "workload"
        operator: "Equal"
        value: "batch"
        effect: "NoSchedule"
      containers:
      - name: job
        image: ghcr.io/example/worker:1.0.0
        resources:
          requests: { cpu: "500m", memory: "512Mi" }
          limits:   { cpu: "2",    memory: "2Gi" }
```
- **전용 노드풀**(taints/tolerations)로 상시 서비스와 배치를 **격리**하라.
- 요청/상한을 명시해 **스케줄 실패/노드 과부하**를 예방.

### 설정/비밀 주입

```yaml
envFrom:
- configMapRef: { name: job-config }
- secretRef:    { name: job-secret }
```
- 민감정보는 **Secret + 파일 마운트** 권장(환경변수는 덤프/로그로 새기 쉬움).

### 종료/중단/타임아웃

- **활동 타임아웃**: `activeDeadlineSeconds`.
- **우아한 종료**: 컨테이너 **TERM 신호 처리**, `terminationGracePeriodSeconds` 고려.
- **일시 정지**: CronJob `suspend: true`.

---

## 패턴 모음(실전)

### 데이터 백업(매일 자정, 동시 실행 금지)

```yaml
apiVersion: batch/v1
kind: CronJob
metadata: { name: db-backup }
spec:
  schedule: "0 0 * * *"
  timeZone: "Asia/Seoul"     # 지원 버전일 때
  concurrencyPolicy: Forbid
  successfulJobsHistoryLimit: 7
  failedJobsHistoryLimit: 3
  jobTemplate:
    spec:
      backoffLimit: 2
      activeDeadlineSeconds: 1800
      template:
        spec:
          restartPolicy: OnFailure
          containers:
          - name: backup
            image: ghcr.io/example/pgdump:1.0.0
            envFrom:
            - secretRef: { name: pg-cred }
            args: ["--out=s3://bucket/daily/$(date +%F).sql.gz"]
```

### 대용량 ETL — 인덱스드 샤딩

```yaml
apiVersion: batch/v1
kind: Job
metadata: { name: etl-sharded }
spec:
  completions: 32
  parallelism: 8
  completionMode: Indexed
  template:
    spec:
      restartPolicy: OnFailure
      containers:
      - name: etl
        image: ghcr.io/example/etl:2.1.0
        env:
        - name: SHARD
          valueFrom:
            fieldRef: { fieldPath: metadata.annotations['batch.kubernetes.io/job-completion-index'] }
        args: ["--input", "s3://raw/2025-11/", "--shard", "$(SHARD)"]
```

### 워크 큐(동적 길이 작업)

```yaml
apiVersion: batch/v1
kind: Job
metadata: { name: etl-queue }
spec:
  parallelism: 10
  template:
    spec:
      restartPolicy: OnFailure
      containers:
      - name: worker
        image: ghcr.io/example/worker:latest
        envFrom:
        - secretRef: { name: mq-cred }
        args: ["--queue=redis://mq:6379/0", "--concurrency=4"]
```

---

## 비용/성능 감 잡기(간단 산식)

평균 처리시간 \(t\)초, 초당 유입 작업량 \(R\)일 때 필요한 **병렬도** \(P\)의 1차 추정:
$$
P \approx \lceil R \cdot t \rceil \cdot \text{버퍼}
$$
- **버퍼**(1.2~2.0)를 곱해 스파이크/락경합/노드 여유를 고려.
- Job: `parallelism ≈ P` (completions은 총량에 맞춰), CronJob: Job 템플릿에 동일 적용.

---

## 트러블슈팅 표

| 증상 | 1차 확인 | 원인 후보 | 해결 |
|---|---|---|---|
| Job `Pending` 지속 | `describe pod` 이벤트 | 리소스 부족/taint/affinity 과도 | 요청 축소, 노드풀 확장, tolerations 추가 |
| 반복 실패(Backoff) | `kubectl logs`, `backoffLimit` | 외부 의존 실패/자격증명/네트워크 | 재시도 로직/지수 백오프, Secret/네트워크 점검 |
| CronJob이 안 돈다 | `startingDeadlineSeconds`, 컨트롤러 로그 | 스케줄 지연/시간대 오해 | timeZone 명시(지원 시), deadline 조정 |
| 동시 실행 중첩 | `concurrencyPolicy` | 기본 Allow로 중복 실행 | Forbid/Replace 사용 |
| 완료 Job 누적 | 이력/TTL 설정 부재 | 기본 보존으로 리소스 증가 | `ttlSecondsAfterFinished`/historyLimit 설정 |
| 인덱스 혼선 | completionMode 미설정 | Indexed 필요 불가 | `completionMode: Indexed`와 인덱스 참조 확인 |

---

## 보안·거버넌스 체크리스트

- **RBAC 최소권한**: Job/CronJob가 읽는 Secret/ConfigMap만 허용.
- **이미지 서명/취약점 스캔**: 배치 컨테이너도 동일 기준.
- **리소스 쿼터**: 네임스페이스에 **동시 실행 제한**(무분별한 Cron 폭주 방지).
- **감사/관측**: 실행당 로그 라우팅, 메트릭(성공률/지연/실패코드), 알람.

---

## 명령 치트시트

```bash
# 생성/조회

kubectl apply -f job.yaml
kubectl apply -f cronjob.yaml
kubectl get jobs,cronjobs
kubectl get pods -l job-name=<job-name>
kubectl describe job/<name>
kubectl describe cronjob/<name>

# 로그

kubectl logs job/<name>
kubectl logs -l job-name=<name> --tail=100 -f

# 강제 삭제/정리

kubectl delete job/<name> --force --grace-period=0
kubectl delete cronjob/<name>

# 이력/TTL(적용)

kubectl patch job/<name> -p '{"spec":{"ttlSecondsAfterFinished":300}}'
```

---

## 결론 — 선택 기준

| 상황 | 리소스 |
|---|---|
| **한 번만** 또는 유한 횟수로 끝내야 하는 배치 | **Job** |
| **주기적**으로 실행해야 하는 배치 | **CronJob** |

핵심은 **성공 개수/시간 제한/재시도/동시 실행**을 명확히 정의하는 것.
Job/CronJob에 **리소스 요청·스케줄링 제약·보안 정책·정리 전략(TTL/이력)** 을 함께 설계하면 **예측 가능한 배치 운영**이 가능해진다.
