---
layout: post
title: Kubernetes - Helm Chart
date: 2025-04-30 20:20:23 +0900
category: Kubernetes
---
# Helm Chart 구조 설명

Helm Chart는 Kubernetes 애플리케이션을 배포하기 위한 **패키지 단위**입니다.  
하나의 Chart는 여러 리소스(Deployment, Service, ConfigMap 등)를 포함하고 있으며,  
`템플릿 + 변수(values.yaml)` 구조로 되어 있습니다.

> Helm Chart = "템플릿화된 Kubernetes 애플리케이션 패키지"

---

## ✅ 기본 구조

```bash
mychart/
├── Chart.yaml          # Chart 메타데이터 (이름, 버전 등)
├── values.yaml         # 사용자 정의 설정값
├── templates/          # 실제 YAML 템플릿 파일들
│   ├── deployment.yaml
│   ├── service.yaml
│   ├── _helpers.tpl    # 템플릿 함수(변수) 정의
├── charts/             # 서브 차트 (Chart dependency)
└── .helmignore         # 무시할 파일 목록 (.gitignore처럼)
```

---

## ✅ 1. Chart.yaml (필수)

Chart에 대한 **기본 정보**를 담는 메타데이터 파일입니다.

```yaml
apiVersion: v2
name: mychart
description: A sample Helm chart for Kubernetes
type: application
version: 1.0.0            # Chart 자체 버전
appVersion: "1.16.0"      # 배포 대상 앱의 버전
```

- `version`: Chart의 버전 (Helm이 이걸 보고 업데이트함)
- `appVersion`: 실제 앱(ex. nginx)의 버전. UI에 표시만 됨.

---

## ✅ 2. values.yaml

템플릿에서 사용하는 **기본 설정값(key-value)**을 정의합니다.

```yaml
replicaCount: 3

image:
  repository: nginx
  tag: 1.25
  pullPolicy: IfNotPresent

service:
  type: ClusterIP
  port: 80

env:
  - name: LOG_LEVEL
    value: debug
```

→ 템플릿 파일 안에서 `{% raw %}{{ .Values.image.repository }}{% endraw %}` 형태로 사용됩니다.

---

## ✅ 3. templates/ (핵심)

`Kubernetes 리소스들을 정의하는 템플릿 파일들`입니다.

Helm은 Go 템플릿 엔진을 사용하며, 여기에 `values.yaml`의 값을 주입합니다.

### 예: `deployment.yaml`

{% raw %}
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: {{ .Release.Name }}-app
spec:
  replicas: {{ .Values.replicaCount }}
  selector:
    matchLabels:
      app: {{ .Release.Name }}
  template:
    metadata:
      labels:
        app: {{ .Release.Name }}
    spec:
      containers:
      - name: app
        image: "{{ .Values.image.repository }}:{{ .Values.image.tag }}"
        imagePullPolicy: {{ .Values.image.pullPolicy }}
        env:
        {{- range .Values.env }}
        - name: {{ .name }}
          value: {{ .value }}
        {{- end }}
```
{% endraw %}

> `.Release.Name`, `.Values`, `.Chart`, `.Capabilities` 등 내장 변수 사용 가능

---

## ✅ 4. _helpers.tpl (선택)

템플릿 내에서 사용하는 **공통 함수, 이름 규칙 등을 정의**하는 파일입니다.

예:

{% raw %}
```yaml
{{- define "mychart.fullname" -}}
{{ .Release.Name }}-{{ .Chart.Name }}
{{- end }}
```
{% endraw %}

템플릿에서 호출:

{% raw %}
```yaml
name: {{ include "mychart.fullname" . }}
```
{% endraw %}

→ 유지보수성 향상, 중복 제거에 유용

---

## ✅ 5. charts/ (subcharts)

이 디렉토리는 다른 Helm Chart를 의존성으로 포함할 때 사용됩니다.

- `charts/redis-*.tgz` 와 같이 압축된 서브차트를 포함
- `dependencies:`는 `Chart.yaml`에서 정의

예:

```yaml
dependencies:
- name: redis
  version: 17.3.11
  repository: https://charts.bitnami.com/bitnami
```

→ `helm dependency update` 명령으로 charts 디렉토리에 다운로드됨

---

## ✅ 6. .helmignore

Helm 패키징 시 무시할 파일을 지정합니다. `.gitignore`와 유사합니다.

```txt
.git/
*.md
*.DS_Store
```

→ `helm package` 명령으로 chart를 .tgz로 만들 때 제외됨

---

## ✅ 예시: 커스텀 values 파일 사용하기

```bash
# 기본 values.yaml 사용
helm install myapp ./mychart

# 커스텀 values 파일 적용
helm install myapp ./mychart -f values-prod.yaml
```

---

## ✅ Helm Chart 흐름 요약

1. `Chart.yaml`에 메타데이터 정의
2. `values.yaml`에 설정값 정의
3. `templates/`에 Kubernetes 리소스 템플릿 작성
4. `helm install`로 템플릿 + 값 => 실제 리소스로 배포

---

## ✅ 결론

Helm Chart는 단순히 파일 모음이 아니라,  
**템플릿화된 애플리케이션 배포 시스템**입니다.

- `values.yaml`을 통해 환경별 설정 분리
- `Go Template` 문법으로 유연한 리소스 정의
- `_helpers.tpl`로 네이밍 전략 통일
- `charts/`를 통해 여러 컴포넌트를 모듈화