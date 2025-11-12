---
layout: post
title: Kubernetes - Calico로 L7 정책 구성
date: 2025-05-11 21:20:23 +0900
category: Kubernetes
---
# Calico로 L7 정책 구성하기

Kubernetes 기본 `NetworkPolicy`는 **L3/L4**(ipBlock/Pod/Namespace + TCP/UDP:port)만 다룬다.  
운영에서는 다음 같은 **L7(애플리케이션 계층) 제어**가 필요해진다.

- HTTP **메서드** 허용/차단 (GET/POST/PUT/DELETE…)  
- HTTP **경로**(접두/정규식) 기반 제어 (`/api/v1/*`만 허용 등)  
- **호스트명**(Host 헤더, FQDN) 기반 구분  
- 특정 **헤더**(예: `X-Auth-Token`) 조건 충족 시 허용

Calico는 **Application Layer Policy**(HTTP 인식)로 이를 제공한다.  
다만 **중요한 라이선스 차이**가 있다:

> - **Calico OSS(오픈소스)**: L3/L4 중심이다. L7(HTTP) 세부 정책은 **기능 제한**이 있으며, 대부분의 **HTTP/L7 풍부한 매칭은 Calico Enterprise**에서 제공된다.  
> - **Calico Enterprise**: `HTTPPolicy`/`GlobalNetworkPolicy`의 `http:` 섹션으로 **method/path/host/header** 매칭, 관측(Flow logs), 정책 시뮬레이션 등 **고급 L7 기능**을 지원한다.

이 글은
1) **Calico Enterprise** 환경에서의 정석 L7 정책,  
2) **Calico OSS**에서 사용할 수 있는 **우회/대체 패턴**(Ingress/Service Mesh/WAF/애플리케이션 게이트웨이 연계)을 **둘 다** 제시한다.

---

## 1. 사전 준비와 판단 가이드

### 1.1 요구사항-기능 매트릭스

| 요구 | Calico OSS | Calico Enterprise |
|---|---|---|
| L3/L4 정책(Pod/NS/IP/포트) | 가능 | 가능(강화) |
| HTTP 메서드/경로/호스트/헤더 | 제한적/사실상 불가 | **가능** |
| DNS/FQDN 기반 아웃바운드 제어 | 일부 사례에서 제한·우회 | **가능(풍부)** |
| 정책 시각화/시뮬레이션/감사 | 수동/외부 도구 | **내장 기능** |

> 엔터프라이즈 없이 “정밀 L7”이 꼭 필요하면, **CNI-레벨 대신 L7 프록시/인그레스/서비스 메시**로 해결하는 설계를 권장한다(아래 7장 참고).

### 1.2 실습 클러스터 가정

- Kubernetes ≥ 1.24
- CNI: Calico (기본 또는 eBPF dataplane)
- 테스트 네임스페이스: `demo`
- HTTP 서비스: `httpbin`(요청을 반사하는 테스트 서버) 또는 `hashicorp/http-echo`

---

## 2. 데모 워크로드 배포

```bash
kubectl create ns demo

# 백엔드: httpbin (GET/POST/…를 쉽게 검증 가능)
kubectl -n demo run backend --image=kennethreitz/httpbin --port=80 --labels="app=backend"
kubectl -n demo expose pod backend --port=80 --target-port=80

# 프런트엔드: 테스트 클라이언트 Pod (셸로 들어가 curl/wget 수행)
kubectl -n demo run frontend --image=busybox:1.36 --restart=Never -it -- sh
```

프런트엔드 컨테이너 셸에서 접속 확인:

```sh
# 허용 정책 설정 전엔 대체로 통과
wget -qO- http://backend.demo.svc.cluster.local/get
```

---

## 3. Calico Enterprise: 정석 L7(HTTP) 정책

Calico Enterprise에서는 **HTTP 인식 정책**을 `GlobalNetworkPolicy`의 `http:` 섹션 혹은 **전용** `HTTPPolicy`(정책 객체 형태)로 선언한다. 아래는 **두 방식 중 하나**를 택해도 된다. 대부분은 `GlobalNetworkPolicy` 한 파일에 L3/L4/L7을 함께 선언하는 패턴이 운영에 편하다.

### 3.1 `GlobalNetworkPolicy`로 L7 제어(대표 패턴)

```yaml
# gnp-l7-allow-get-path.yaml
apiVersion: projectcalico.org/v3
kind: GlobalNetworkPolicy
metadata:
  name: demo-l7-allow-get-path
spec:
  # backend 파드로 들어오는 트래픽 제어
  selector: app == "backend"
  order: 100  # 우선순위(낮을수록 먼저 평가). 운영에서 필수
  types:
  - Ingress
  ingress:
  - action: Allow
    protocol: TCP
    source:
      namespaceSelector: projectcalico.org/name == "demo"   # demo NS에서만
      selector: app == "frontend"                            # frontend만
    destination:
      ports: [80]
    http:
      methods: ["GET"]
      paths:
        - exact: "/get"             # 정확히 /get
        - prefix: "/api/v1/"        # /api/v1/* 접두 허용
      hosts:
        - "backend.demo.svc.cluster.local"  # Host 매칭(선택)
      headers:
        # 정적 헤더 매칭 예시(필요 시)
        - name: "X-Env"
          exact: "test"
  - action: Deny   # 나머지는 차단(기본 Deny가 아니라면 명시 권장)
```

적용:

```bash
kubectl apply -f gnp-l7-allow-get-path.yaml
```

테스트:

```sh
# frontend 셸에서
# 1. 허용: GET /get
wget -S -qO- --header="X-Env: test" http://backend.demo.svc.cluster.local/get

# 2. 허용: GET /api/v1/users (prefix 규칙)
wget -S -qO- --header="X-Env: test" http://backend.demo.svc.cluster.local/api/v1/users

# 3. 차단: POST /post
wget -S -qO- --post-data="x=1" http://backend.demo.svc.cluster.local/post

# 4. 차단: 헤더 미지정
wget -S -qO- http://backend.demo.svc.cluster.local/get
```

> 응답이 **끊기거나 403/권한거부**로 보일 수 있다. 실제 동작은 Calico 버전/모드(Conntrack/프록시)와 파드 측 응답 모드에 따라 다소 차이가 있다.

### 3.2 `HTTPPolicy` + 네트워크 정책 연동(대안 패턴)

HTTP 매칭 룰을 별도 객체로 나누면 **여러 정책에서 재사용** 가능하다.

```yaml
# http-policy.yaml
apiVersion: projectcalico.org/v3
kind: HTTPPolicy
metadata:
  name: demo-http-allow-list
spec:
  selector: app == "backend"
  rules:
    - methods: ["GET"]
      paths:
        - exact: "/get"
        - prefix: "/api/"
      headers:
        - name: "X-Env"
          exact: "test"
```

`HTTPPolicy`를 참조하는 `GlobalNetworkPolicy`(또는 동등한 연결)가 필요한 구성은 배포 모델에 따라 다르다(엔터프라이즈 UI/CRD 연동). 운영에서는 **한 파일에 L3/L4 + L7을 합치는 패턴**이 간결하다(3.1 참고).

### 3.3 헤더·호스트 고급 매칭 예시

```yaml
apiVersion: projectcalico.org/v3
kind: GlobalNetworkPolicy
metadata:
  name: demo-l7-headers-hosts
spec:
  selector: app == "backend"
  order: 90
  types: [Ingress]
  ingress:
  - action: Allow
    protocol: TCP
    source:
      selector: app == "frontend"
    destination:
      ports: [80]
    http:
      methods: ["GET"]
      hosts:
        - "api.example.com"
        - "backend.demo.svc.cluster.local"
      headers:
        - name: "Authorization"
          prefix: "Bearer "     # Bearer 토큰 접두 매칭
        - name: "X-Role"
          regex: "^admin|ops$"  # 정규식 매칭(버전 지원 확인)
  - action: Deny
```

> 헤더/정규식 지원 범위는 배포 버전·라이선스에 따라 달라질 수 있다. 정책 릴리즈 노트를 확인할 것.

---

## 4. 정책 검증 시나리오

1. **통과 케이스**  
   - `frontend` → `backend` : `GET /get` + `X-Env: test` 헤더 → **Allow**
2. **거부 케이스**  
   - `frontend` → `backend` : `POST /post` → **Deny**  
   - `frontend` → `backend` : `GET /get` (헤더 누락) → **Deny**  
   - `demo` 외 네임스페이스 → `backend` : 어떤 요청이든 → **Deny**  
3. **옵저버빌리티**  
   - Calico 엔터프라이즈의 플로우 로그/정책 히트 카운터/시뮬레이터로 룰 매치 확인  
   - OSS라면 kube-proxy/앱 로그로 간접 확인

---

## 5. 우선순위와 기본 거부(Deny) 전략

Calico는 정책 평가에 **order**(숫자가 **낮을수록 우선**)가 관여한다.  
운영 권장 패턴:

1. **기본 차단** 정책(큰 `order` 값, 예: 1000)으로 바닥 깔기
2. **세부 허용** 정책(작은 `order`)을 앞쪽에 배치
3. 마지막에 남는 트래픽 = 의도치 않은 접근 → 기본 Deny로 흡수

예:

```yaml
apiVersion: projectcalico.org/v3
kind: GlobalNetworkPolicy
metadata:
  name: demo-deny-all
spec:
  order: 1000
  selector: app == "backend"
  types: [Ingress]
  ingress:
  - action: Deny
```

> 네임스페이스 단위 `NetworkPolicy`와 글로벌 정책의 상호작용을 항상 염두에 둘 것. 테스트 네임스페이스별로 **기본 거부**를 둔 뒤, 필요한 경로만 허용하는 **제로 트러스트** 스타일이 안전하다.

---

## 6. 성능·설계 고려사항

- **프록시/앱 계층 인식**이 개입되면 L3/L4 순수 패킷 필터보다 **오버헤드가 증가**한다.  
  - 고 QPS 경로에는 경로 캐싱·CDN·게이트웨이 레이어에서 coarse-grained 제어 후 Calico L7로 정밀 제어를 덧입히는 **계층형 방어**를 고려.
- **eBPF dataplane** 사용 시 L3/L4 성능 이점이 있으나, L7 인식은 별도 경로/프록시가 관여할 수 있다. 실제 스택에서 측정 필수.
- **TLS 종단**: L7 매칭은 평문 HTTP 헤더·경로에 의존한다.  
  - End-to-end TLS(파드 내부까지 암호화)라면, L7 인식을 위해 **중간(인그레스/사이드카)에서 TLS 종료/재암호화**가 필요하다.
- **정규식/헤더 다중 매칭**은 룰 비용 증가. 가능하면 `prefix/exact` 중심으로 설계.

---

## 7. Calico OSS만 사용할 때의 우회/대체 패턴

Calico OSS는 사실상 L7 풍부한 매칭이 어렵다. 대안은 다음 **L7 계층**에서 해결하는 것이다.

1. **Ingress Controller(NGINX/HAProxy/Contour)**  
   - Ingress `path`/`host`/`method` 제어  
   - 인증/헤더 검사/RateLimit/미들웨어 연동  
   - 장점: 운영 친화, 선언적, 널리 사용
2. **Service Mesh(Istio/Linkerd)**  
   - `AuthorizationPolicy`로 **method/path/host/헤더** 제어  
   - mTLS/정책/관측 일체 제공  
   - 대규모 마이크로서비스에 적합
3. **API Gateway(Kong/Traefik/NGINX Plus)**  
   - 라우팅/인증/쿼터/키 관리/서킷브레이커 등  
   - 북-남(North-South)·동-서(East-West) 트래픽 모두 다룸
4. **애플리케이션 레벨 미들웨어**  
   - 앱 프레임워크 필터/미들웨어로 `method/path/header` 제한  
   - 단점: 분산된 정책, 일관성 관리 부담

> OSS Calico는 이들 L7 계층 앞뒤에서 **L3/L4 최소 권한**을 강제(예: 인그레스/메시 엔드포인트만 열어두기)하는 용도로 탁월하다.

### 7.1 OSS + Ingress로 L7 제어 예시

```yaml
# Ingress: /api/*만 백엔드로 전달, 메서드·헤더 검증은 NGINX 어노테이션/정책으로
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: demo-ing
  namespace: demo
  annotations:
    nginx.ingress.kubernetes.io/use-regex: "true"
    nginx.ingress.kubernetes.io/configuration-snippet: |
      if ($request_method !~ ^(GET|HEAD)$) { return 403; }
      if ($http_x_env != "test") { return 403; }
spec:
  rules:
  - host: backend.demo.local
    http:
      paths:
      - path: /api/?(.*)
        pathType: Prefix
        backend:
          service:
            name: backend
            port:
              number: 80
```

여기에 Calico **L3/L4**(OSS) 정책으로 **Ingress Controller ↔ backend** 경로만 열어 최소 권한을 적용한다.

```yaml
# OSS GlobalNetworkPolicy: Ingress Controller Pod만 backend:80 접근 허용
apiVersion: projectcalico.org/v3
kind: GlobalNetworkPolicy
metadata:
  name: oss-backend-minimal
spec:
  selector: app == "backend"
  order: 100
  types: [Ingress]
  ingress:
  - action: Allow
    protocol: TCP
    source:
      selector: app == "ingress-nginx"  # 실제 라벨에 맞게 조정
    destination:
      ports: [80]
  - action: Deny
```

---

## 8. 운영 체크리스트

- 정책 파일에 **`order`**를 반드시 명시하고, 기본 Deny를 맨 아래 깔아라.  
- L7 매칭은 **TLS 종단 위치**에 좌우된다(종단 전 구간은 L4로만 보인다).  
- **네임스페이스 격리** + **서비스 아이덴티티(라벨/SA)**로 **소스 범위**를 좁혀라.  
- 변경 전 **스테이징**에서 시나리오 테스트(허용/차단 케이스).  
- 고 QPS 경로는 규칙을 단순화하고, 가능한 **prefix/exact**를 쓰고 **정규식 최소화**.  
- **관측**: 플로우 로그/엔드포인트 카운터/인그레스 접근 로그로 룰 히트 확인.  
- 대체 불가 L7 요구나 감사·레포팅이 중요하면 **Calico Enterprise 도입**을 검토하라.

---

## 9. 전체 예제(엔터프라이즈 경로) — 한 번에 적용하는 묶음

```yaml
# 1. 데모 네임스페이스(이미 생성했다면 생략)
apiVersion: v1
kind: Namespace
metadata:
  name: demo
---
# 2. backend 서비스(앞서 만든 Pod를 쓴다면 생략 가능)
apiVersion: v1
kind: Service
metadata:
  name: backend
  namespace: demo
spec:
  selector:
    app: backend
  ports:
  - port: 80
    targetPort: 80
---
# 3. L7 허용(프런트엔드→백엔드 GET /get, /api/v1/* + 헤더)
apiVersion: projectcalico.org/v3
kind: GlobalNetworkPolicy
metadata:
  name: demo-l7-allow
spec:
  selector: app == "backend"
  order: 100
  types: [Ingress]
  ingress:
  - action: Allow
    protocol: TCP
    source:
      namespaceSelector: projectcalico.org/name == "demo"
      selector: app == "frontend"
    destination:
      ports: [80]
    http:
      methods: ["GET"]
      paths:
        - exact: "/get"
        - prefix: "/api/v1/"
      headers:
        - name: "X-Env"
          exact: "test"
  - action: Deny
---
# 4. 기본 차단(안전망)
apiVersion: projectcalico.org/v3
kind: GlobalNetworkPolicy
metadata:
  name: demo-deny-backend
spec:
  selector: app == "backend"
  order: 1000
  types: [Ingress]
  ingress:
  - action: Deny
```

검증 요약:

```sh
# 허용
wget -S -qO- --header="X-Env: test" http://backend.demo.svc.cluster.local/get
wget -S -qO- --header="X-Env: test" http://backend.demo.svc.cluster.local/api/v1/users

# 차단
wget -S -qO- --post-data="x=1" http://backend.demo.svc.cluster.local/post
wget -S -qO- http://backend.demo.svc.cluster.local/get
```

---

## 10. 트러블슈팅

| 증상 | 원인 후보 | 점검/해결 |
|---|---|---|
| 기대한 L7 매칭이 동작하지 않음 | OSS에서 L7 기능 미지원 | 엔터프라이즈 기능 여부 확인. OSS는 7장의 대체 패턴을 사용 |
| 허용 규칙이 있는데도 차단됨 | `order`/우선순위 문제, 상위 Deny가 먼저 매치 | `order` 낮은 정책부터 평가. `calicoctl get policy -o wide`로 순서 확인 |
| 헤더/호스트 매칭이 먹지 않음 | TLS 종단 위치 문제, Host 헤더 불일치 | TLS Termination을 인그레스/프록시로 이동하거나 재구성. 실제 Host 값 확인 |
| 일부는 통과/일부는 403 | 경로·정규식/헤더 조건 미충족 | 규칙 단순화, 로그로 실제 요청 값 확인 |
| 성능 저하 | L7 검사·정규식 과다 | prefix/exact 우선, 고 QPS 엔드포인트 분리, 캐시/게이트웨이 앞단 배치 |

---

## 결론

- **Calico Enterprise**는 `GlobalNetworkPolicy`/`HTTPPolicy`의 `http:` 섹션으로 **HTTP method/path/host/header**까지 정밀 제어가 가능하다. **제로 트러스트** 네트워킹을 **네트워크 계층에서** 끌어올릴 때 유효하다.  
- **Calico OSS**만 쓸 때는 **L7 기능이 제한**되므로, **Ingress/Service Mesh/API Gateway**에서 L7 정책을 구현하고, Calico(OSS)는 **L3/L4 최소 권한**을 강제하는 **이중 레이어** 전략이 현실적이다.  
- 운영에서는 **정확한 우선순위(order)**, **기본 Deny**, **TLS 종단 고려**, **성능 측정/관측**을 포함한 **정책 수명주기 관리**가 필수다.

이 문서의 매니페스트들을 그대로 적용·수정하면, **엔터프라이즈 경로(L7 네이티브)**와 **OSS 우회 경로(L7은 프록시, L3/L4는 Calico)**를 모두 검증할 수 있다.