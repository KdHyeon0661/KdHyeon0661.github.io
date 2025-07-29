---
layout: post
title: Kubernetes - 나만의 Helm 차트 만들기
date: 2025-04-30 22:20:23 +0900
category: Kubernetes
---
# 나만의 Helm 차트 만들기

Helm을 사용하면 오픈소스 패키지를 쉽게 설치할 수 있지만,  
**자신만의 애플리케이션**을 Kubernetes에 배포할 때는  
**커스텀 Helm Chart를 직접 만드는 것이 더 유용**할 수 있습니다.

이 글에서는 `helm create` 명령으로 Chart를 만들고,  
템플릿을 커스터마이징하여 자신만의 차트를 완성하는 과정을 자세히 설명합니다.

---

## ✅ 1. Helm Chart 생성

```bash
helm create mychart
```

이 명령은 기본 구조를 자동으로 생성합니다:

```
mychart/
├── Chart.yaml          # 메타데이터
├── values.yaml         # 기본 설정값
├── templates/          # 템플릿 리소스들
│   ├── deployment.yaml
│   ├── service.yaml
│   ├── ingress.yaml
│   └── _helpers.tpl
└── .helmignore
```

---

## ✅ 2. values.yaml 설정

`values.yaml` 파일은 템플릿에서 사용할 설정값을 정의하는 파일입니다.

예시:

```yaml
replicaCount: 2

image:
  repository: myregistry/myapp
  tag: latest
  pullPolicy: IfNotPresent

service:
  type: ClusterIP
  port: 8080
```

→ 템플릿 내에서 `{{ .Values.image.repository }}` 식으로 사용됩니다.

---

## ✅ 3. 템플릿 수정: deployment.yaml

기본 생성된 `templates/deployment.yaml`을 자신의 앱에 맞게 수정합니다.

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: {{ include "mychart.fullname" . }}
spec:
  replicas: {{ .Values.replicaCount }}
  selector:
    matchLabels:
      app: {{ include "mychart.name" . }}
  template:
    metadata:
      labels:
        app: {{ include "mychart.name" . }}
    spec:
      containers:
        - name: {{ .Chart.Name }}
          image: "{{ .Values.image.repository }}:{{ .Values.image.tag }}"
          imagePullPolicy: {{ .Values.image.pullPolicy }}
          ports:
            - containerPort: {{ .Values.service.port }}
```

→ `include`, `.Chart`, `.Values` 등의 헬퍼 변수를 활용

---

## ✅ 4. 템플릿 수정: service.yaml

```yaml
apiVersion: v1
kind: Service
metadata:
  name: {{ include "mychart.fullname" . }}
spec:
  type: {{ .Values.service.type }}
  ports:
    - port: {{ .Values.service.port }}
      targetPort: {{ .Values.service.port }}
  selector:
    app: {{ include "mychart.name" . }}
```

---

## ✅ 5. _helpers.tpl 작성

`_helpers.tpl` 파일은 반복되는 이름 등을 템플릿 함수로 재사용할 수 있게 도와줍니다.

```yaml
{{- define "mychart.name" -}}
{{ .Chart.Name }}
{{- end }}

{{- define "mychart.fullname" -}}
{{ .Release.Name }}-{{ .Chart.Name }}
{{- end }}
```

---

## ✅ 6. 템플릿 확인 및 렌더링 테스트

실제 Kubernetes 리소스가 어떻게 생성되는지 확인하려면:

```bash
helm template mychart
```

→ 템플릿을 적용한 결과를 출력

---

## ✅ 7. Chart 설치

```bash
helm install myapp ./mychart
```

또는 설정값을 별도 파일로 관리:

```bash
helm install myapp ./mychart -f my-values.yaml
```

→ `kubectl get all`로 리소스 확인 가능

---

## ✅ 8. 업그레이드 및 수정

설정 변경 후:

```bash
helm upgrade myapp ./mychart -f my-values.yaml
```

삭제 시:

```bash
helm uninstall myapp
```

---

## ✅ 9. 실전 예제: values-dev.yaml vs values-prod.yaml

```yaml
# values-dev.yaml
replicaCount: 1
image:
  tag: dev

# values-prod.yaml
replicaCount: 4
image:
  tag: stable
```

환경에 따라 설치:

```bash
helm install myapp ./mychart -f values-dev.yaml
```

---

## ✅ 10. 디버깅 팁

- 템플릿 결과 미리 보기: `helm template`
- 값 확인: `helm get values myapp`
- 값 오타 체크: `helm lint`
- 특정 값 override: `--set key=value` 사용

---

## ✅ 결론

나만의 Helm Chart를 만들면 다음과 같은 이점이 있습니다:

- 복잡한 애플리케이션 배포를 반복 가능하게 관리
- 환경별 설정을 분리하여 안전한 운영 가능
- GitOps 기반으로 CI/CD와 쉽게 통합
- 기존 YAML을 **템플릿화**하여 유지보수가 쉬움