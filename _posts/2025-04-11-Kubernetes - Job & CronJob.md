---
layout: post
title: Kubernetes - Job & CronJob
date: 2025-04-11 19:20:23 +0900
category: Kubernetes
---
# Kubernetes 핵심 오브젝트: Job과 CronJob 이해하기

## 개요

쿠버네티스는 단순히 웹 서버나 API와 같은 장시간 실행되는 서비스뿐만 아니라, 데이터 처리, 리포트 생성, 백업과 같은 **일회성 또는 주기적 배치 작업**도 실행할 수 있습니다. 이를 위해 설계된 두 가지 핵심 오브젝트가 바로 **Job**과 **CronJob**입니다. Job은 한 번 실행되고 종료되는 작업을 관리하며, CronJob은 Job을 정해진 스케줄에 따라 반복적으로 생성합니다.

---

## Job: 일회성 작업 관리

### 기본 개념

Job은 하나 이상의 Pod을 생성하고, 지정된 수의 Pod이 **성공적으로 완료될 때까지** 실행을 보장합니다. 실행 중 Pod이 실패하면, Job 컨트롤러는 설정된 재시도 한도 내에서 새로운 Pod을 생성하여 작업을 계속합니다. 작업이 완료되면 Job은 성공 또는 실패 상태로 남아 있으며, 완료된 Pod도 기록으로 남습니다.

### 간단한 Job 예시

다음 YAML은 간단한 메시지를 출력하고 종료하는 Job을 정의합니다.

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
        command: ["sh", "-c", "echo 'Hello from a Kubernetes Job'"]
      restartPolicy: Never  # Job에서는 'Never' 또는 'OnFailure'만 사용 가능
```

### Job의 주요 구성 파라미터

Job의 동작을 세밀하게 제어하기 위해 다음과 같은 스펙 필드들을 사용할 수 있습니다.

```yaml
apiVersion: batch/v1
kind: Job
metadata:
  name: example-job
spec:
  completions: 5          # 총 성공해야 하는 Pod의 수
  parallelism: 2          # 동시에 실행할 Pod의 수 (병렬 처리)
  backoffLimit: 4         # 실패 시 새로운 Pod으로 재시도할 최대 횟수
  activeDeadlineSeconds: 300  # 전체 작업의 최대 실행 시간(초). 초과하면 실패
  ttlSecondsAfterFinished: 3600 # 작업 완료 후 Job과 Pod 리소스를 자동 삭제하기까지의 대기 시간(초)
  template:
    spec:
      containers:
      - name: worker
        image: my-worker:latest
      restartPolicy: OnFailure
```

- **`completions`**: 이 작업이 총 몇 번 성공해야 완료되는지 정의합니다. `1`이 기본값입니다.
- **`parallelism`**: 한 번에 몇 개의 Pod을 병렬로 실행할지 정의합니다. `completions`보다 작거나 같아야 합니다.
- **`backoffLimit`**: Pod이 실패할 때마다 재시도하는 것이 아니라, **새로운 Pod을 생성하여 재시도**합니다. 이 필드는 실패 가능성을 고려하여 충분히 설정해야 합니다.
- **`activeDeadlineSeconds`**: 작업이 너무 오래 실행되는 것을 방지합니다. 이 시간을 초과하면 전체 Job이 실패로 표시됩니다.
- **`ttlSecondsAfterFinished`**: 완료된 Job과 그 Pod을 자동으로 정리하여 클러스터 리소스를 관리하는 데 유용합니다.

### 병렬 Job: 배치 처리 활용

여러 개의 독립적인 작업 항목을 병렬로 처리해야 할 때 Job을 사용할 수 있습니다. 예를 들어, 10개의 데이터 파일을 처리해야 하고, 최대 3개씩 병렬로 처리하려면 다음과 같이 설정합니다.

```yaml
apiVersion: batch/v1
kind: Job
metadata:
  name: batch-processor
spec:
  completions: 10  # 총 10개의 작업 항목
  parallelism: 3   # 최대 3개를 동시에 처리
  template:
    spec:
      containers:
      - name: processor
        image: data-processor:latest
        command: ["process-file.sh"]
      restartPolicy: OnFailure
```

### 인덱싱된 Job (Indexed Job) - 작업 샤딩

작업 항목에 순서나 고유 식별자가 필요한 경우 `Indexed` 완료 모드를 사용할 수 있습니다. 이 모드에서는 각 Pod에 0부터 시작하는 고유 인덱스(`JOB_COMPLETION_INDEX`)가 할당됩니다. 이 인덱스를 활용하여 입력 데이터를 분할(샤딩) 처리할 수 있습니다.

```yaml
apiVersion: batch/v1
kind: Job
metadata:
  name: indexed-job
spec:
  completions: 5
  parallelism: 2
  completionMode: Indexed  # 인덱싱 모드 활성화
  template:
    spec:
      containers:
      - name: worker
        image: alpine:3.20
        command: ["sh", "-c"]
        args:
          - |
            echo "Processing shard number $JOB_COMPLETION_INDEX"
            # 예: input-file-part-$JOB_COMPLETION_INDEX.csv 파일을 처리
        env:
        - name: JOB_COMPLETION_INDEX
          valueFrom:
            fieldRef:
              fieldPath: metadata.annotations['batch.kubernetes.io/job-completion-index']
      restartPolicy: OnFailure
```

### 워크 큐(Work Queue) 패턴: 동적 작업 할당

작업 항목의 총 수를 미리 알 수 없는 경우, `parallelism`만 설정하고 Pod들이 공통 작업 큐(예: Redis, RabbitMQ, Amazon SQS)에서 작업을 가져가도록 구성하는 **워크 큐 패턴**을 사용할 수 있습니다. Job은 지정된 수의 작업자 Pod을 유지하며, 큐가 빌 때까지 작업을 처리합니다.

```yaml
apiVersion: batch/v1
kind: Job
metadata:
  name: work-queue-consumer
spec:
  parallelism: 5  # 5개의 작업자 Pod을 유지
  template:
    spec:
      containers:
      - name: worker
        image: queue-worker:latest
        command: ["python", "/app/worker.py"] # 큐에서 작업을 가져와 처리하는 스크립트
      restartPolicy: OnFailure
```

---

## CronJob: 주기적 작업 예약

### 기본 개념

CronJob은 리눅스의 `cron` 데몬처럼, Cron 형식의 스케줄 표현식을 기반으로 Job을 주기적으로 생성합니다. 이를 통해 데이터 백업, 리포트 생성, 정기적인 데이터 정리와 같은 반복 작업을 자동화할 수 있습니다.

### 간단한 CronJob 예시

매분마다 현재 시간을 출력하는 CronJob을 정의해 보겠습니다.

```yaml
apiVersion: batch/v1
kind: CronJob
metadata:
  name: hello-cron
spec:
  schedule: "*/1 * * * *"  # 매분 실행 (Cron 표현식)
  jobTemplate:             # 이 템플릿으로 Job을 생성
    spec:
      template:
        spec:
          containers:
          - name: hello
            image: busybox:1.36
            command: ["sh", "-c", "date; echo 'Hello from CronJob'"]
          restartPolicy: OnFailure
```

### CronJob의 주요 구성 파라미터

CronJob의 동작을 제어하는 몇 가지 중요한 필드가 있습니다.

```yaml
apiVersion: batch/v1
kind: CronJob
metadata:
  name: nightly-backup
spec:
  schedule: "0 2 * * *"          # 매일 새벽 2시 실행
  concurrencyPolicy: Forbid       # 동시 실행 정책
  successfulJobsHistoryLimit: 3   # 성공한 Job 이력을 3개까지 보관
  failedJobsHistoryLimit: 1       # 실패한 Job 이력을 1개까지 보관
  startingDeadlineSeconds: 600    # 지연 시작 허용 시간(초)
  suspend: false                  # true로 설정하면 스케줄 일시 중지
  timeZone: "Asia/Seoul"          # 스케줄 해석 기준 시간대 (쿠버네티스 1.27+)
  jobTemplate:
    spec:
      backoffLimit: 2
      template:
        spec:
          restartPolicy: OnFailure
          containers:
          - name: backup
            image: backup-tool:latest
```

- **`concurrencyPolicy`**:
    - **`Allow`** (기본값): 이전 실행이 아직 진행 중이더라도 새로운 Job을 생성합니다.
    - **`Forbid`**: 이전 실행이 진행 중이면 새로운 실행을 스킵합니다.
    - **`Replace`**: 이전 실행을 취소하고 새로운 실행으로 대체합니다.
- **`startingDeadlineSeconds`**: CronJob 컨트롤러가 다운되었거나 클러스터에 부하가 걸려 스케줄을 놓쳤을 때, 이 시간 내의 실행은 지연되어 시작될 수 있습니다. 이를 초과하면 실행이 실패로 기록됩니다.
- **`timeZone`**: 쿠버네티스 1.27 버전부터 지원됩니다. 명시하지 않으면 CronJob 컨트롤러의 로컬 시간대(대부분 UTC)를 기준으로 스케줄이 해석됩니다.

---

## Job과 CronJob 비교

| 항목 | Job | CronJob |
|---|---|---|
| **주 목적** | 일회성 또는 유한 횟수의 작업 실행 보장 | 주기적 반복 작업의 예약 및 실행 |
| **실행 트리거** | 사용자 또는 시스템이 생성하는 즉시 실행 | Cron 표현식에 따른 자동 생성 |
| **병렬 처리** | `parallelism`, `completions`, `Indexed` 모드 지원 | `jobTemplate` 내의 Job 설정을 상속받음 |
| **실패 관리** | `backoffLimit`, `activeDeadlineSeconds`, Pod Failure Policy | 상위 Job 설정을 상속받음. 추가로 `startingDeadlineSeconds`로 지연 실행 관리 |
| **리소스 정리** | `ttlSecondsAfterFinished`로 완료 후 자동 삭제 | `successfulJobsHistoryLimit`, `failedJobsHistoryLimit`로 이력 관리. 생성된 Job도 TTL 적용 가능 |

---

## 운영 실습: 명령어 활용

### Job 관련 명령
```bash
# Job 생성 및 확인
kubectl apply -f my-job.yaml
kubectl get jobs
kubectl describe job my-job

# Job의 Pod 확인 및 로그 보기
kubectl get pods -l job-name=my-job
kubectl logs job/my-job  # 가장 최근의 완료된 Pod 로그

# Job 강제 삭제
kubectl delete job my-job
```

### CronJob 관련 명령
```bash
# CronJob 생성 및 확인
kubectl apply -f my-cronjob.yaml
kubectl get cronjobs
kubectl describe cronjob my-cronjob

# CronJob이 생성한 Job 목록 확인
kubectl get jobs --watch

# CronJob 일시 중지/재개
kubectl patch cronjob my-cronjob -p '{"spec":{"suspend":true}}'
kubectl patch cronjob my-cronjob -p '{"spec":{"suspend":false}}'
```

---

## 모범 사례와 고려사항

### 1. 리소스 및 스케줄링 구성
배치 작업이 상시 서비스에 영향을 주지 않도록 전용 노드풀을 구성하고, `nodeSelector`, `affinity`, `tolerations`를 활용하는 것이 좋습니다. 또한 리소스 요청(`requests`)과 상한(`limits`)을 명시하여 스케줄링과 노드 안정성을 보장해야 합니다.

```yaml
spec:
  template:
    spec:
      nodeSelector:
        workload-type: batch
      tolerations:
      - key: "workload"
        operator: "Equal"
        value: "batch"
        effect: "NoSchedule"
      containers:
      - name: worker
        image: my-worker:latest
        resources:
          requests:
            cpu: "500m"
            memory: "1Gi"
          limits:
            cpu: "2"
            memory: "4Gi"
```

### 2. 설정 및 비밀 정보 관리
환경변수보다는 파일 마운트 방식을 통해 ConfigMap과 Secret을 주입하는 것이 보안상 더 안전합니다. 환경변수는 로그나 프로세스 목록에 노출될 위험이 있습니다.

```yaml
envFrom:
- configMapRef:
    name: job-config
- secretRef:
    name: job-secrets
volumeMounts:
- name: secret-volume
  mountPath: /etc/secrets
  readOnly: true
```

### 3. 종료 및 정리 전략
- `activeDeadlineSeconds`를 설정하여 무한 루프나 장기 대기를 방지하세요.
- `ttlSecondsAfterFinished`를 사용하여 완료된 Job 리소스를 자동 정리하세요.
- CronJob의 경우 `successfulJobsHistoryLimit`과 `failedJobsHistoryLimit`을 적절히 설정하여 불필요한 이력이 누적되지 않도록 관리하세요.

---

## 실전 패턴 예제

### 1. 매일 데이터베이스 백업 (CronJob)
```yaml
apiVersion: batch/v1
kind: CronJob
metadata:
  name: daily-db-backup
spec:
  schedule: "0 3 * * *"  # 매일 새벽 3시
  concurrencyPolicy: Forbid  # 중복 실행 방지
  successfulJobsHistoryLimit: 7  # 일주일치 성공 이력 보관
  jobTemplate:
    spec:
      template:
        spec:
          containers:
          - name: backup
            image: postgres:15-alpine
            envFrom:
            - secretRef:
                name: db-credentials
            command: ["pg_dump"]
            args: ["-h", "$(DB_HOST)", "-U", "$(DB_USER)", "-d", "$(DB_NAME)", "-f", "/backup/backup-$(date +%Y%m%d).sql"]
            volumeMounts:
            - name: backup-volume
              mountPath: /backup
          restartPolicy: OnFailure
          volumes:
          - name: backup-volume
            persistentVolumeClaim:
              claimName: backup-pvc
```

### 2. 대용량 데이터 ETL 처리 (인덱싱된 Job)
```yaml
apiVersion: batch/v1
kind: Job
metadata:
  name: etl-process
spec:
  completions: 50  # 총 50개의 샤드 처리
  parallelism: 10  # 최대 10개 샤드 동시 처리
  completionMode: Indexed
  template:
    spec:
      containers:
      - name: etl-worker
        image: etl-pipeline:latest
        command: ["python", "/app/process.py"]
        args: ["--shard-index", "$(JOB_COMPLETION_INDEX)"]
        env:
        - name: JOB_COMPLETION_INDEX
          valueFrom:
            fieldRef:
              fieldPath: metadata.annotations['batch.kubernetes.io/job-completion-index']
      restartPolicy: OnFailure
```

---

## 문제 해결 가이드

| 증상 | 확인 사항 | 가능한 원인 및 해결 방안 |
|---|---|---|
| **Job이 계속 `Pending` 상태** | `kubectl describe pod <pod-name>`의 이벤트 확인 | 리소스 부족, 노드 선택기(NodeSelector) 불일치, 테인트(Taint) 허용 불가. 리소스 요청을 낮추거나 노드풀을 확장하세요. |
| **Job이 반복적으로 실패** | `kubectl logs <pod-name>` 로그 확인. `backoffLimit` 설정 확인. | 애플리케이션 오류, 외부 의존성(DB, API) 접근 실패, 잘못된 자격 증명. 로그를 분석하고 `backoffLimit`을 적절히 조정하세요. |
| **CronJob이 예정된 시간에 실행되지 않음** | CronJob 컨트롤러 로그 확인. `startingDeadlineSeconds` 설정 확인. | 스케줄 표현식 오류, 컨트롤러 중단, 시간대 차이. `timeZone`을 명시하고(지원되는 경우), `startingDeadlineSeconds`를 늘려보세요. |
| **CronJob이 중복 실행됨** | `concurrencyPolicy` 설정 확인. | `Allow`(기본값)로 설정되어 있을 수 있습니다. 중복을 원하지 않으면 `Forbid`로 변경하세요. |
| **완료된 Job/Pod이 계속 쌓임** | `ttlSecondsAfterFinished`, `historyLimit` 설정 확인. | 자동 정리 설정이 없습니다. `ttlSecondsAfterFinished`(Job) 또는 `*HistoryLimit`(CronJob)을 설정하세요. |

---

## 결론

Job과 CronJob은 쿠버네티스를 데이터 처리 플랫폼으로서의 가능성을 넓혀주는 강력한 오브젝트입니다. **Job**은 "성공적인 완료"를 보장하는 일회성 작업 실행에, **CronJob**은 정기적인 반복 작업의 자동화에 적합합니다.

이들을 효과적으로 사용하기 위한 핵심은 명확한 목표 설정입니다: **몇 번 성공해야 하는가(`completions`)?**, **얼마나 동시에 실행할 것인가(`parallelism`)?**, **실패 시 어떻게 대응할 것인가(`backoffLimit`, `activeDeadlineSeconds`)?**

또한 운영의 안정성을 위해 리소스 요청/제한 설정, 전용 노드풀 활용, 보안을 위한 Secret 관리, 그리고 `ttlSecondsAfterFinished`나 히스토리 제한을 통한 자원 정리 전략을 함께 설계하는 것이 중요합니다. 이러한 요소들을 종합적으로 고려하여 Job과 CronJob을 구성한다면, 예측 가능하고 신뢰할 수 있는 배치 작업 운영 체계를 구축할 수 있을 것입니다.