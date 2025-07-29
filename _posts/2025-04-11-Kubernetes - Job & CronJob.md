---
layout: post
title: Kubernetes - Job & CronJob
date: 2025-04-11 19:20:23 +0900
category: Kubernetes
---
# Kubernetes 핵심 오브젝트 이해하기  
## Job & CronJob

Kubernetes는 기본적으로 **지속적으로 실행되는 서비스(Pod)**를 운영하기 위한 플랫폼입니다.  
하지만, 백업, 데이터 마이그레이션, 배치 작업 등 **일회성 또는 정해진 시간에 실행해야 하는 작업**이 필요할 때도 있습니다.

이럴 때 사용하는 리소스가 바로:

- `Job` : **한 번만 실행되는 작업**
- `CronJob` : **정해진 시간마다 반복되는 작업**

---

## ✅ 1. Job이란?

Job은 **일회성 작업을 보장하는 쿠버네티스 리소스**입니다.  
성공할 때까지 Pod를 재시작하고, 지정한 개수만큼 성공하면 종료됩니다.

> 예: DB 초기화 스크립트 실행, 백업 스냅샷, 마이그레이션 스크립트

### 🔧 Job YAML 예제

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
        image: busybox
        command: ["echo", "Hello Kubernetes"]
      restartPolicy: Never
```

### 🧠 핵심 설정

| 필드 | 설명 |
|------|------|
| `completions` | 총 실행 횟수 (성공 기준) |
| `parallelism` | 동시에 실행할 작업 수 |
| `backoffLimit` | 실패 후 재시도 횟수 |
| `restartPolicy` | 항상 `Never` or `OnFailure` 사용 |

```yaml
spec:
  completions: 3
  parallelism: 2
  backoffLimit: 4
```

---

## ✅ 2. CronJob이란?

CronJob은 **리눅스의 crontab처럼 일정한 시간마다 Job을 실행**하는 오브젝트입니다.

> 예: 매일 자정에 로그 압축, 매 5분마다 DB 백업

### 🔧 CronJob YAML 예제

```yaml
apiVersion: batch/v1
kind: CronJob
metadata:
  name: hello-cron
spec:
  schedule: "*/1 * * * *" # 매 1분
  jobTemplate:
    spec:
      template:
        spec:
          containers:
          - name: hello
            image: busybox
            command: ["echo", "Hello from CronJob"]
          restartPolicy: OnFailure
```

### 🧠 주요 필드 설명

| 필드 | 설명 |
|------|------|
| `schedule` | cron 형식 스케줄 (`"0 * * * *"`, `"*/5 * * * *"` 등) |
| `jobTemplate` | 실행할 Job의 템플릿 |
| `successfulJobsHistoryLimit` | 성공 기록 보존 개수 |
| `failedJobsHistoryLimit` | 실패 기록 보존 개수 |
| `startingDeadlineSeconds` | 늦게 시작될 수 있는 최대 시간 |
| `concurrencyPolicy` | `Allow`, `Forbid`, `Replace` (동시 실행 정책) |

---

## ✅ 3. Job vs CronJob 요약 비교

| 항목 | Job | CronJob |
|------|-----|---------|
| 용도 | 단일 작업 실행 | 반복 작업 예약 실행 |
| 실행 시점 | 즉시 | 스케줄에 따라 |
| 실행 횟수 | 1회 또는 명시 횟수 | 반복 |
| YAML 구조 | 단순 | `schedule` + `jobTemplate` 포함 |
| 활용 예 | DB 초기화, 일회성 마이그레이션 | 로그 백업, 배치 처리 |

---

## ✅ 4. 실습 명령어

### Job 실행

```bash
kubectl apply -f hello-job.yaml
kubectl get jobs
kubectl get pods
kubectl logs job/hello-job
```

### CronJob 실행 및 로그 확인

```bash
kubectl apply -f hello-cronjob.yaml
kubectl get cronjobs
kubectl get jobs --watch
```

---

## ✅ 5. Job 관련 팁

- `restartPolicy`는 반드시 `Never` 또는 `OnFailure`
- Job의 Pod가 실패하면 자동 재시작 → `backoffLimit` 횟수만큼
- 실행 완료된 Job의 Pod는 **삭제되지 않고 남아 있음**
  → 필요 시 `ttlSecondsAfterFinished`를 설정하여 자동 정리 가능

```yaml
spec:
  ttlSecondsAfterFinished: 300  # 완료 후 5분 뒤 삭제
```

---

## ✅ 6. CronJob 관련 팁

- `schedule`은 **UTC 기준**이므로 시간대 주의
- 동시 실행 제어 필요 시 `concurrencyPolicy` 사용

```yaml
spec:
  concurrencyPolicy: Forbid  # 동시에 실행 방지
```

- 너무 자주 실행하면 Job이 누적되어 리소스 고갈 가능 → `historyLimit` 설정 필요

```yaml
successfulJobsHistoryLimit: 3
failedJobsHistoryLimit: 1
```

---

## ✅ 결론

| 상황 | 사용 리소스 |
|------|--------------|
| 단발성 작업 실행 | ✅ Job |
| 주기적 작업 예약 | ✅ CronJob |

Kubernetes에서 반복적이지 않은 작업, 배치성 작업을 처리할 때는 `Job`,  
시간 기반으로 반복 실행이 필요한 작업에는 `CronJob`을 사용하는 것이 정석입니다.