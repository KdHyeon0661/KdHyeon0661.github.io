---
layout: post
title: 웹해킹 - 로그 관리와 침해 대응 전략 &  DevSecOps 관점에서의 웹 애플리케이션 보안
date: 2025-09-30 16:25:23 +0900
category: 웹해킹
---
# 🧭 로그 관리와 침해 대응 전략 &  DevSecOps 관점에서의 웹 애플리케이션 보안

## 0. 큰 그림

- **로그 관리**는 “누가, 무엇을, 언제, 어디서, 어떻게”를 잃지 않게 하는 **증거 보존·관측 기반**입니다.
- **침해(Incident) 대응**은 **탐지 → 분류 → 격리/완화 → 근절 → 복구 → 교훈화**의 반복 가능한 프로세스입니다.
- **DevSecOps**는 개발–보안–운영을 **파이프라인/플랫폼**에 녹여 “기본 보안값(Default Secure)”을 자동화합니다.

---

# 1. 로그 관리 전략 (수집·전송·보관·탐지·가치화)

### 1.1 무엇을 기록할 것인가 (웹앱 기준 최소 세트)

1) **HTTP 접근 로그**: 메서드, 경로, 상태, 바이트, 지연, 클라이언트 IP, UA, **추적/상관 ID**
2) **인증/인가 이벤트**: 로그인 성공/실패, MFA, 비밀번호 변경, 세션 생성/폐기, 역할 변경
3) **보안 이벤트**: 속도 제한 발동, WAF 차단, 유효성 검증 실패(요약), CSRF/XSS/SQLi 패턴 탐지
4) **애플리케이션 이벤트**: 예외/오류(스택 포함), 중요 비즈니스 행위(주문/환불/관리자 작업)
5) **인프라/플랫폼**: 컨테이너 스케줄링, 재시작, K8s 감사, 클라우드 감사(CloudTrail/AuditLog 등)
6) **데이터 계층**: DB 감사(스키마 변경/대량 SELECT/권한 변경), 캐시 미스/키 패턴 이상

> 원칙: **개인·비밀 데이터는 최소화**(Data Minimization). 필요 시 적절히 **마스킹/해싱**.

---

### 1.2 표준화된 로그 스키마(JSON Lines 권장)

- **필수 필드**: `ts`(RFC3339/UTC), `severity`, `service`, `env`, `trace_id`, `span_id`, `corr_id`, `user_id`(또는 주체), `src_ip`, `method`, `path`, `status`, `latency_ms`, `bytes_out`, `ua`, `msg`, `tags`
- **추적 연계**: W3C Trace Context(`traceparent` 헤더) 또는 `X-Correlation-Id`

**예 — HTTP 접근 로그(JSON):**
```json
{"ts":"2025-10-27T03:12:45.321Z","severity":"INFO","service":"web","env":"prod",
 "trace_id":"a9f...","corr_id":"req_17c...","user_id":"u_123","src_ip":"203.0.113.10",
 "method":"POST","path":"/api/v1/login","status":200,"latency_ms":142,"bytes_out":512,
 "ua":"Mozilla/5.0 ...","msg":"http_access","tags":["auth","edge=alb"]}
```

**예 — 인증 실패 이벤트:**
```json
{"ts":"2025-10-27T03:13:07.003Z","severity":"WARN","service":"web","env":"prod",
 "event":"auth_failed","user_id":null,"src_ip":"203.0.113.10","username":"alice@example.com",
 "reason":"bad_credentials","attempt":5,"risk_score":42,"corr_id":"req_17d..."}
```

---

### 1.3 애플리케이션 레벨 구현 (상관 ID/PII 마스킹/구조화)

#### Node.js(Express) 미들웨어 — 상관 ID + 구조 로그 + 마스킹
```javascript
import crypto from "node:crypto";
import pino from "pino";
const logger = pino({ level: process.env.LOG_LEVEL || "info" });

function genId() { return crypto.randomBytes(12).toString("hex"); }

function requestLogger(req, res, next) {
  req.corrId = req.headers["x-correlation-id"] || genId();
  res.setHeader("X-Correlation-Id", req.corrId);

  const start = process.hrtime.bigint();
  const safePath = req.path; // 경로 정규화는 라우터에서 처리
  const srcIp = req.headers["x-forwarded-for"]?.split(",")[0]?.trim() || req.socket.remoteAddress;

  res.on("finish", () => {
    const latency = Number(process.hrtime.bigint() - start) / 1e6;
    logger.info({
      ts: new Date().toISOString(), severity: "INFO", service: "web", env: process.env.NODE_ENV,
      corr_id: req.corrId, trace_id: req.headers["traceparent"], src_ip: srcIp,
      method: req.method, path: safePath, status: res.statusCode, latency_ms: Math.round(latency),
      ua: req.headers["user-agent"], msg: "http_access"
    });
  });
  next();
}

// 간단 PII 마스킹(이메일/카드 등) — 실제로는 토큰화/해싱/정책 사용
function maskPII(obj) {
  const s = JSON.stringify(obj);
  return s
    .replace(/"email"\s*:\s*"([^"]+)"/gi, '"email":"***"')
    .replace(/\b\d{12,19}\b/g, "****"); // 카드번호 등 단순 패턴
}

function auditLog(event, details) {
  const payload = { ts: new Date().toISOString(), severity: "INFO", service: "web", env: process.env.NODE_ENV,
                    event, ...details };
  console.log(maskPII(payload));
}

export { requestLogger, auditLog, logger };
```

#### Flask — 로거/필터/상관 ID
```python
import logging, uuid, time
from flask import g, request

logger = logging.getLogger("app")
handler = logging.StreamHandler()
formatter = logging.Formatter('%(message)s')  # JSON 직렬화 직접 수행
handler.setFormatter(formatter); logger.addHandler(handler); logger.setLevel(logging.INFO)

@app.before_request
def before():
    g.corr_id = request.headers.get("X-Correlation-Id", uuid.uuid4().hex)
    g.t0 = time.time()

@app.after_request
def after(resp):
    latency = int((time.time() - g.t0)*1000)
    rec = {"ts": time.strftime("%Y-%m-%dT%H:%M:%S%z"), "service":"web", "env":"prod",
           "corr_id": g.corr_id, "method": request.method, "path": request.path,
           "status": resp.status_code, "latency_ms": latency, "src_ip": request.remote_addr,
           "ua": request.headers.get("User-Agent"), "msg":"http_access"}
    logger.info(json.dumps(rec))
    resp.headers["X-Correlation-Id"] = g.corr_id
    return resp
```

---

### 1.4 수집·전송 파이프라인 예 (Fluent Bit / OpenTelemetry / Loki)

**Fluent Bit → OTEL Collector → SIEM**
- **에이전트**: 컨테이너/노드에서 로그 tail
- **OTEL**: 파싱/샘플링/속성 추가/전송
- **목적지**: Elastic, Splunk, Loki, Cloud Logging, S3(아카이브)

**Fluent Bit 예:**
```ini
[INPUT]
    Name              tail
    Path              /var/log/app/*.log
    Parser            json
    Tag               app.*
    Mem_Buf_Limit     128MB
    Skip_Long_Lines   On

[FILTER]
    Name              modify
    Match             app.*
    Add               env prod
    Add               service web

[OUTPUT]
    Name              http
    Match             app.*
    Host              otel-collector
    Port              4318
    Format            json
    URI               /v1/logs
```

**OpenTelemetry Collector(Logs/Traces/Metrics 단일 파이프라인)**
```yaml
receivers:
  otlp:
    protocols: { http: {}, grpc: {} }

processors:
  batch: {}
  attributes:
    actions:
      - key: env
        value: prod
        action: upsert

exporters:
  loki:
    endpoint: http://loki:3100/loki/api/v1/push
  elasticsearch:
    endpoints: [ "http://es:9200" ]
    logs_index: "logs-web-%{+yyyy.MM.dd}"

service:
  pipelines:
    logs:
      receivers: [otlp]
      processors: [attributes, batch]
      exporters: [loki, elasticsearch]
    traces:
      receivers: [otlp]
      processors: [batch]
      exporters: [jaeger]
```

---

### 1.5 보존·무결성·접근 통제

- **보존 기간**: 운영 30~90일(핫), 규정 준수/포렌식 1~7년(아카이브/WORM)
- **암호화**: 전송 TLS, 저장 시 KMS로 암호화
- **무결성**: 서명/해시 체인, 버킷 **객체 잠금(WORM)** 사용, 접근감사 활성화
- **시간 동기화**: NTP, 모든 시스템 UTC 표준, RFC3339 정규화
- **권한**: 최소권한(RBAC), 운영/보안 구분, 쿼리 결과의 **PII 마스킹** 정책

**간단 해시 체인(교육용)**
```bash
# new.logl: 새 줄 추가 전 해시 누적
prev=$(tail -n1 chain.txt | cut -d' ' -f1)
h=$(printf "%s%s" "$prev" "$(cat line.json)" | sha256sum | cut -d' ' -f1)
echo "$h $(cat line.json)" >> chain.txt
```

---

### 1.6 탐지 쿼리/규칙 예 (Splunk/Elastic/Loki/Sigma)

**1) 과도한 로그인 실패(자격증명 대입 의심)**
- *Splunk SPL*:
```spl
index=auth sourcetype=app event=auth_failed
| bin _time span=5m
| stats count as fails by user_id, src_ip, _time
| where fails >= 20
```
- *Elastic KQL*:
```kql
event:"auth_failed" and @timestamp >= now()-5m
| stats count() by user_id, src_ip
| where count >= 20
```
- *Loki LogQL*:
```logql
{event="auth_failed"} | unwrap user_id | unwrap src_ip
| rate(5m) by (user_id, src_ip) >= 20
```

**2) 의심스러운 경로 탐색(Path Traversal 시도)**
```spl
index=web event=http_access path="*../*" OR path="*..%2f*" OR path="*%252e%252e%252f*"
| stats count by src_ip, path
```

**3) SSRF 의심(169.254.169.254 요청 발생)**
(프록시/백엔드 로그에서 대상 호스트가 인스턴스 메타데이터)
```spl
index=proxy event=egress host IN ("169.254.169.254","100.100.100.200")
| stats count by src_pod, dst_host, corr_id
```

**4) 관리자 권한 변경/고위험 작업 감시 (Sigma 예시)**
```yaml
title: Admin Role Granted
logsource:
  product: application
  service: web
detection:
  selection:
    event: "role_change"
    role: "admin"
  condition: selection
level: high
fields:
  - user_id
  - actor_id
  - corr_id
```

---

# 2. 침해 대응(Incident Response) 전략 & 플레이북

### 2.1 프로세스(요지)

1. **준비(Preparation)**: 자산/연락망/권한/연습/런북/로깅
2. **식별(Identification)**: 알림·탐지 규칙·휴리스틱으로 사건 여부 판단
3. **격리(Containment)**: 네트워크/계정/서비스 범위 제한(단기/장기)
4. **근절(Eradication)**: 악성 아티팩트 제거, 취약점 수정, 구성 강화
5. **복구(Recovery)**: 모니터링 하에 서비스 정상화, **재발 방지** 점검
6. **교훈(LL — Lessons Learned)**: RCA, 대응 지표 분석, 보안채무 상환 계획

**핵심 KPI**: MTTA(탐지까지 평균시간), MTTR(복구까지), MTTC(격리까지), FPR/FNR.

---

### 2.2 공통 런북 뼈대(템플릿)

**사건 카드(예)**
- 식별자: `INC-YYYYMMDD-###`
- 트리거: (경보/보고/외부통보)
- 첫 확인자/시간:
- 영향: (시스템/사용자/데이터 분류)
- 초기 조치: (격리/차단/계정잠금/토큰폐기)
- 커뮤니케이션: (내부/법무/PR/고객)
- 증거 보존: (로그/스냅샷/메모리/디스크)
- RCA 요약/개선안:

**체인오브커스터디(증거 관리)**
- 수집 주체/시간/방법/해시/보관 위치/접근자 기록

---

### 2.3 상황별 플레이북

#### A) **SQL Injection 의심**
- **신호**: “syntax error near”, “UNION SELECT”, ORM 경고, DB 대량 스캔, WAF 탐지
- **즉시**
  1) 공격 계정/IP 임시 차단(단, 대규모 IP 변이 주의)
  2) 취약 엔드포인트 **ReadOnly 모드** 또는 임시 내려
  3) DB 사용자 권한 점검(최소 권한/스키마 분리), 크레덴셜 **로테이션**
- **조사**
  - 애플리케이션 로그(쿼리 파라미터를 **요약/식별 가능 토큰** 형태로)
  - DB 감사 로그 / 슬로우쿼리 / 스냅샷 비교
  - 데이터 유출 영향평가(SELECT 로그/전송량/타임라인)
- **근절**
  - 쿼리 → **Prepared Statement/ORM**로 전환, 입력 **스키마 검증** 강화
  - 자동 테스트에 **악성 페이로드 세트** 추가
- **복구/LL**
  - 규칙/경보 조정, 고객 공지(필요 시), 규제 보고(해당 시)

#### B) **SSRF 의심**
- **신호**: 백엔드가 `169.254.169.254` 등 메타데이터로 아웃바운드, 비정상 egress
- **즉시**: egress 프록시/SG/NACL에서 **링크로컬/메타데이터 차단**, 크레덴셜 로테이션
- **근절**: URL **허용목록**, DNS/IP 재검증, 리다이렉트 차단, IMDSv2/헤더 요구 강제
- **복구/LL**: 코드 리뷰(프록시 강제), 테스트 케이스 추가

#### C) **계정 탈취(ATO) 의심**
- **신호**: 짧은 시간 다지역 로그인, 장치 프린트 변경, 쿠키 재사용
- **즉시**: 해당 계정 세션 **전부 무효화**, MFA 강제, 비밀번호 재설정 링크 발송
- **조사**: 로그인/토큰/장치 지문/리퍼러/비정상 IP 대역
- **근절**: 리스크 기반 로그인, WebAuthn 도입, 알림/승인 흐름
- **복구/LL**: 사용자 공지 템플릿, 지원 채널 준비

#### D) **업로드 후 웹셸 의심**
- **신호**: 업로드 디렉터리에서 실행, 낯선 프로세스 스폰(Falco/EDR)
- **즉시**: 경로 격리/읽기전용 마운트, 웹앱 컨테이너 교체(immutable infra)
- **근절**: 확장자/콘텐츠 스니핑/스토리지 분리(정적 CDN), 실행권한 제거, CDR/AV
- **복구/LL**: 업로드 파이프라인 재평가, CI 보안 테스트

---

### 2.4 알림·의사소통

- **내부 알림**: Slack/Teams(보안 채널) + On-call (PagerDuty/Alertmanager)
- **외부 커뮤니케이션**: 고객 공지/법무/규제 기관 보고(해당 시)
- **알림 템플릿(예)**:
```json
{
  "severity": "high",
  "incident": "INC-2025-10-27-003",
  "title": "[SQLi] /api/v1/search 파라미터에서 패턴 탐지",
  "start": "2025-10-27T03:15:00Z",
  "impacted": ["web-prod-a","rds-cluster-1"],
  "actions": ["WAF rule enabled","DB creds rotated","endpoint read-only"],
  "corr_id": "req_17e...",
  "runbook": "https://internal/wiki/runbooks/sqli"
}
```

**Alertmanager → Slack 라우팅 예**
{% raw %}
```yaml
receivers:
  - name: "sec-slack"
    slack_configs:
      - channel: "#security"
        send_resolved: true
        title: "[{{ .Status | toUpper }}] {{ .CommonLabels.alertname }}"
        text: "{{ range .Alerts }}*{{ .Labels.severity }}* - {{ .Annotations.summary }} ({{ .Labels.instance }})\n{{ end }}"
```
{% endraw %}

---

# 3. DevSecOps 관점의 웹 애플리케이션 보안

### 3.1 “Shift-left + Shift-right” 원칙

- **Shift-left(개발 단계)**: 설계/코드에서 **결함 예방** — Threat Modeling, SAST, SCA, IaC Scan, Secret Scan
- **Shift-right(운영 단계)**: **관측/탐지/대응** 자동화 — DAST, CWPP/KSPM, 런타임 보호(WAF/RASP), 관측(OTEL)

---

### 3.2 CI 파이프라인 예 (GitHub Actions 기준)

```yaml
name: ci-secure
on: [push, pull_request]

jobs:
  sast-semgrep:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: returntocorp/semgrep-action@v1
        with: { config: "p/owasp-top-ten" }

  deps-sbom:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - name: Build SBOM (CycloneDX)
        run: |
          npm ci
          npx @cyclonedx/cyclonedx-npm --output-file sbom.json
      - uses: actions/upload-artifact@v4
        with: { name: sbom, path: sbom.json }

  secret-scan:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: trufflesecurity/trufflehog@main
        with: { path: "." }

  iac-scan:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: bridgecrewio/checkov-action@master
        with: { directory: "." }

  dast-zap:
    needs: [sast-semgrep]
    runs-on: ubuntu-latest
    steps:
      - name: OWASP ZAP baseline
        uses: zaproxy/action-baseline@v0.12.0
        with:
          target: "https://staging.example.com"
          rules_file_name: ".zap/rules.tsv"
          cmd_options: "-d"
```

---

### 3.3 공급망/배포 보안

- **SBOM**: 릴리스마다 생성(예: CycloneDX), **취약 의존성** 사전 차단
- **서명/증명**: 컨테이너 이미지 **서명(cosign)**, **프로비넌스(SLSA)**
- **이미지 취약점 스캔**: trivy/grype → 차단 게이트
- **비밀 관리**: HashiCorp Vault / AWS Secrets Manager / K8s Sealed Secrets(평문 금지)

**예 — cosign 서명**
```bash
COSIGN_EXPERIMENTAL=1 cosign sign --key cosign.key registry/app:1.2.3
cosign verify --key cosign.pub registry/app:1.2.3
```

---

### 3.4 IaC/Kubernetes 정책-코드 (OPA/Gatekeeper, Kyverno)

**Rego 예 — Ingress는 TLS 필수**
```rego
package kubepolicy.ingress

deny[msg] {
  input.kind.kind == "Ingress"
  not input.spec.tls
  msg := sprintf("Ingress %s/%s has no TLS.", [input.metadata.namespace, input.metadata.name])
}
```

**Kyverno 예 — privileged 금지**
```yaml
apiVersion: kyverno.io/v1
kind: ClusterPolicy
metadata:
  name: disallow-privileged
spec:
  validationFailureAction: Enforce
  rules:
    - name: deny-priv
      match: { resources: { kinds: ["Pod"] } }
      validate:
        message: "Privileged containers are not allowed."
        pattern:
          spec:
            containers:
              - =(securityContext):
                  =(privileged): "false"
```

---

### 3.5 런타임 보안(Falco) — 웹셸/이상 행위 탐지

```yaml
# Nginx/Apache/PHP-FPM 컨테이너가 쉘 스폰 시 경보
- rule: WebServer Spawns Shell
  desc: Detect web server spawning a shell
  condition: proc.name in (bash,sh,zsh) and
             container and
             (parent.name in (nginx,apache2,httpd,php-fpm))
  output: "Web server spawned shell (user=%user.name proc=%proc.name parent=%proc.pname container=%container.name)"
  priority: CRITICAL
```

---

### 3.6 관측 통합(OpenTelemetry) — 로그·트레이스 상관

**Node(Express) — OTEL 초기화**
```javascript
import { NodeSDK } from "@opentelemetry/sdk-node";
import { getNodeAutoInstrumentations } from "@opentelemetry/auto-instrumentations-node";
import { OTLPTraceExporter } from "@opentelemetry/exporter-trace-otlp-http";

const sdk = new NodeSDK({
  traceExporter: new OTLPTraceExporter({ url: "http://otel-collector:4318/v1/traces" }),
  instrumentations: [getNodeAutoInstrumentations()]
});
sdk.start();
```

**로그에 trace_id/span_id 포함**
```javascript
logger.info({
  trace_id: (req.headers["traceparent"] || "").split("-")[1],
  span_id: "..." // 컨텍스트에서 추출
});
```

---

### 3.7 WAF/RASP/레이트 제한

- **WAF**: 관리형 규칙(SQLi/XSS/PRCE) + **애플리케이션 특화 규칙**(경로/메서드/속도)
- **레이트 제한**: 사용자/토큰/IP/엔드포인트 조합, **버스트+지속** 두 축
- **RASP**: 런타임 인젝션 시도 차단(적용 범위·성능 고려)

**Express – 레이트 제한 예**
```javascript
import rateLimit from "express-rate-limit";
const limiter = rateLimit({
  windowMs: 5*60*1000, max: 500,
  standardHeaders: true, legacyHeaders: false
});
app.use("/api/", limiter);
```

---

# 4. 정책·거버넌스·개인정보 보호

- **데이터 분류**: 공개/내부/기밀/고기밀
- **PII/민감정보 처리**: 수집 최소화, 저장 시 암호화, **로그 마스킹** 기본
- **접근심사**: 생산 데이터 접근은 **티켓/승인/만료**
- **보존 정책**: 서비스/법무/컴플라이언스와 합의, **자동 삭제**
- **위험평가/트레이드오프**: 성능·가용성·개발속도 vs. 보안 강화 항목을 **백로그에서 가시화**

---

# 5. 추가 예제 — 로그 스키마/대시보드/보고

### 5.1 Nginx 구조화 로그
```nginx
log_format json_combined escape=json
  '{ "ts":"$time_iso8601", "service":"edge", "env":"prod", "src_ip":"$remote_addr",'
  '"method":"$request_method","path":"$uri","status":$status,"bytes_out":$bytes_sent,'
  '"ref":"$http_referer","ua":"$http_user_agent","req_id":"$request_id","host":"$host",'
  '"req_time":$request_time,"upstream_time":"$upstream_response_time","msg":"edge_access" }';

access_log /var/log/nginx/access.json.log json_combined;
```

### 5.2 Elastic/Kibana KQL(대시보드 타일 예)
```kql
msg:"http_access" and env:"prod"
| stats avg(latency_ms) as p50_latency by path
```

### 5.3 주간 보안 리포트(템플릿)
- 요약: 주요 사건 N건, 심각도 분포, MTTA/MTTR, 재발 방지 진행상황
- Top 탐지 규칙 발생: (규칙명/건수/유효율)
- 인증 이상: 실패 상위 IP/ASN/국가
- 신규 취약 의존성: SBOM 비교 결과
- 조치 현황: 패치/정책 배포 리스트

---

# 6. 교육/훈련(게임데이/블루팀 드릴)

- **게임데이 시나리오**: SQLi 알림 → 런북 실행 → 격리/근절 → 포스트모템
- **근무시간 외 온콜 훈련**: 페일오버/롤백/키 로테이션 리허설
- **개발자 세션**: “로그는 누군가 나중에 읽는다” — 구조 로그·상관 ID·마스킹 실습

---

# 7. 체크리스트(요약)

- [ ] JSON 구조 로그/상관 ID/UTC 타임스탬프
- [ ] OTEL로 **로그·트레이스·메트릭** 통합
- [ ] Fluent Bit/Collector → SIEM/아카이브(WORM)
- [ ] 탐지 규칙(인증/입력 검증 실패/경로 탐색/egress 이상) & 알림 라우팅
- [ ] IR 런북/증거 보존/연락망/권한/정기 훈련
- [ ] CI: SAST/SCA/Secret/IaC/DAST & 차단 게이트
- [ ] SBOM/서명/프로비넌스/이미지 스캔
- [ ] Kubernetes 정책-코드(OPA/Kyverno), Falco 런타임 탐지
- [ ] WAF/레이트 제한/세션 보안/헤더 하드닝
- [ ] 개인정보/보존/암호화/접근심사/자동 삭제

---

## 맺음말

관측 가능한 시스템만이 **빠르게 대응**할 수 있습니다.
**로그 품질**(정확·구조·연계)과 **자동화된 DevSecOps 파이프라인**을 기반으로, 감지–격리–복구–교훈의 **루프 속도를 끌어올리세요**.
