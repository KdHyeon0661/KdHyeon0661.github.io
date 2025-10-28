---
layout: post
title: Django - CQRS, 읽기 모델 분리 · (선별적) 도메인 이벤트 소싱 · 모놀리스
date: 2025-10-06 19:25:23 +0900
category: Django
---
# CQRS / 읽기 모델 분리 · (선별적) 도메인 이벤트 소싱 · 모놀리스 → 모듈 분해 전략

## 1) CQRS — 읽기 모델 분리

### 1-1. 왜 CQRS인가?

- **쓰기(명령)**: 불변 조건/불일치 감지/권한 검사/트랜잭션 중심.  
- **읽기(조회)**: 조인/집계/페이지네이션/캐시/검색 최적화 중심.  
- 하나의 ORM 모델로 두 요구를 모두 만족시키려면 복잡성이 급격히 증가합니다. **CQRS는 쓰기 모델과 읽기 모델을 분리**하여 각자 최적화합니다.

> 실무 팁: “모든 곳에 CQRS”가 아니라, **트래픽이 높거나 조회가 복잡한 경로부터** 부분 적용하세요.

---

### 1-2. 최소 적용 구조

```
apps/
  orders/
    domain/        # 엔티티·도메인 규칙
    app/           # Command Handler (서비스)
    infra/         # Repository(ORM), Outbox
    ui/            # 명령용 View/API
  readmodels/
    orders/        # Projection (조회 테이블/문서)
      infra/       # projector, consumer
      ui/          # 조회용 API/뷰(가벼운 쿼리)
shared/
  outbox/          # outbox 모델, 게시 태스크
```

- **쓰기 모델**: 정규화된 스키마, 외래키/제약/트랜잭션.  
- **읽기 모델**: 목록/상세/카운트/검색에 맞춘 **비정규화 테이블**(또는 JSON 문서/캐시).

---

### 1-3. 예제 도메인 (주문)

#### 쓰기 모델(정규화)

```python
# apps/orders/infra/models.py
from django.db import models
from django.conf import settings

class Order(models.Model):
    user = models.ForeignKey(settings.AUTH_USER_MODEL, on_delete=models.PROTECT)
    status = models.CharField(max_length=20, default="CREATED")  # CREATED/PAID/CANCELLED
    total_amount = models.PositiveIntegerField(default=0)         # 원화 정수
    created_at = models.DateTimeField(auto_now_add=True)

class OrderItem(models.Model):
    order = models.ForeignKey(Order, on_delete=models.CASCADE, related_name="items")
    product_id = models.IntegerField()
    product_name = models.CharField(max_length=180)  # 스냅샷
    unit_price = models.PositiveIntegerField()
    qty = models.PositiveIntegerField()

    class Meta:
        indexes = [models.Index(fields=["order", "product_id"])]
```

#### 읽기 모델(프로젝션)

```python
# apps/readmodels/orders/infra/models.py
from django.db import models

class OrderSummary(models.Model):
    order_id = models.IntegerField(primary_key=True)
    user_id = models.IntegerField(db_index=True)
    status = models.CharField(max_length=20, db_index=True)
    total_amount = models.PositiveIntegerField()
    item_count = models.PositiveIntegerField()
    created_date = models.DateField(db_index=True)
    # 목록/필터/정렬 최적화 필드
    latest_item_name = models.CharField(max_length=180, blank=True)

    class Meta:
        indexes = [
            models.Index(fields=["user_id", "-order_id"]),
            models.Index(fields=["status", "-order_id"]),
            models.Index(fields=["created_date", "-order_id"]),
        ]
```

---

### 1-4. 동기화: Outbox → Projector

**문제**: “주문 생성/결제” 트랜잭션과 “읽기 모델 반영”을 **원자적으로** 맞추기 어렵습니다.  
**해결**: **Outbox 패턴** — 쓰기 트랜잭션 커밋과 **같은 DB**에 이벤트 레코드 저장 → 백그라운드 워커가 안전하게 읽어 프로젝션 반영.

#### Outbox 모델/API

```python
# shared/outbox/models.py
from django.db import models

class OutboxEvent(models.Model):
    topic = models.CharField(max_length=80)     # e.g. "orders"
    type = models.CharField(max_length=80)      # e.g. "OrderCreated"
    aggregate_id = models.CharField(max_length=64, db_index=True)
    payload = models.JSONField()
    published = models.BooleanField(default=False)
    created_at = models.DateTimeField(auto_now_add=True)

    class Meta:
        indexes = [models.Index(fields=["published", "created_at"])]
```

```python
# shared/outbox/api.py
from django.db import transaction
from .models import OutboxEvent

def add_outbox_event(topic: str, type: str, aggregate_id: str, payload: dict):
    # 트랜잭션 commit 후 Outbox에 기록
    def _save():
        OutboxEvent.objects.create(topic=topic, type=type, aggregate_id=aggregate_id, payload=payload)
    transaction.on_commit(_save)
```

#### Command Handler(쓰기)에서 발행

```python
# apps/orders/app/services.py
from django.db import transaction
from shared.outbox.api import add_outbox_event
from apps.orders.infra.models import Order, OrderItem

class OrderService:
    def create_order(self, user_id: int, items: list[dict]) -> int:
        with transaction.atomic():
            o = Order.objects.create(user_id=user_id)
            total = 0
            for it in items:
                OrderItem.objects.create(
                    order=o,
                    product_id=it["product_id"],
                    product_name=it["name"],
                    unit_price=it["price"],
                    qty=it["qty"],
                )
                total += it["price"] * it["qty"]
            o.total_amount = total
            o.save(update_fields=["total_amount"])

            add_outbox_event(
                topic="orders",
                type="OrderCreated",
                aggregate_id=str(o.id),
                payload={"order_id": o.id, "user_id": user_id, "total": total, "items": [
                    {"name": it["name"], "qty": it["qty"]} for it in items
                ]},
            )
            return o.id

    def pay_order(self, order_id: int):
        with transaction.atomic():
            o = Order.objects.select_for_update().get(pk=order_id)
            if o.status != "CREATED": return
            o.status = "PAID"
            o.save(update_fields=["status"])
            add_outbox_event("orders","OrderPaid",str(o.id),{"order_id":o.id})
```

#### Projector(읽기 모델 반영)

```python
# apps/readmodels/orders/infra/projector.py
from django.db import transaction
from shared.outbox.models import OutboxEvent
from .models import OrderSummary

def _apply_order_created(ev):
    p = ev.payload
    with transaction.atomic():
        OrderSummary.objects.update_or_create(
            order_id=p["order_id"],
            defaults={
                "user_id": p["user_id"],
                "status": "CREATED",
                "total_amount": p["total"],
                "item_count": len(p["items"]),
                "created_date": ev.created_at.date(),
                "latest_item_name": p["items"][0]["name"] if p["items"] else "",
            },
        )

def _apply_order_paid(ev):
    p = ev.payload
    OrderSummary.objects.filter(order_id=p["order_id"]).update(status="PAID")

HANDLERS = {
    "OrderCreated": _apply_order_created,
    "OrderPaid": _apply_order_paid,
}

def run_projection(batch=200):
    # Outbox의 미게시 이벤트를 순차 처리
    qs = OutboxEvent.objects.filter(published=False, topic="orders").order_by("id")[:batch]
    for ev in qs:
        try:
            HANDLERS[ev.type](ev)
            ev.published = True
            ev.save(update_fields=["published"])
        except Exception:
            # 로그/재시도, poison message 격리 전략
            continue
```

> **특징**: 쓰기 경로는 “주문” 스키마에만 집중, 읽기 경로는 `OrderSummary`만 조회.  
> **일관성**: 읽기 모델은 **최종적 일관성**(eventual consistency). UI는 상태가 잠시 늦게 반영될 수 있음을 알려야 합니다(예: 스피너/토스트).

---

### 1-5. 읽기 API/뷰

```python
# apps/readmodels/orders/ui/views.py
from django.views.generic import ListView
from ..infra.models import OrderSummary

class MyOrdersView(ListView):
    model = OrderSummary
    template_name = "orders/list.html"
    paginate_by = 20

    def get_queryset(self):
        return OrderSummary.objects.filter(user_id=self.request.user.id).order_by("-order_id")
```

---

### 1-6. CQRS 운영 체크리스트

- [ ] **Outbox**: 같은 DB/트랜잭션에서 이벤트 기록, **전용 워커**로 투입  
- [ ] **Projector**: idempotent(중복 적용 안전), 재시도/Poison 분리  
- [ ] **읽기 모델 재생성 스크립트**: 초기 백필/재빌드 가능(리플레이)  
- [ ] **스키마 버전**: 이벤트/프로젝션 양쪽에 버전 태그, 점진 이행  
- [ ] **모니터링**: Outbox 적체, Projection 지연, 읽기-쓰기 **레이턴시** 대시보드화

---

## 2) (선별적) 도메인 이벤트 소싱

> “모든 애그리게이트를 이벤트 소싱”이 아니라, **감사·규정 준수·예측/팩트 재생**이 필요한 **핵심 애그리게이트만** 이벤트 소싱을 적용합니다.

### 2-1. 핵심 개념

- **Event Store**: append-only. `aggregate_id`, `version`, `type`, `payload`, `ts`.  
- **Aggregate**: 이벤트 시퀀스를 **재생(replay)** 해서 현재 상태를 구함.  
- **명령 → 이벤트**: 상태 변경은 “이벤트 생성”으로 표현.  
- **스냅샷**: 이벤트가 수천/수만 건이 되면 중간 상태를 저장해 재생 단축.

---

### 2-2. Event Store 스키마

```python
# shared/es/models.py
from django.db import models

class Event(models.Model):
    stream = models.CharField(max_length=80)          # 예: "orders"
    aggregate_id = models.CharField(max_length=64, db_index=True)
    version = models.PositiveIntegerField()           # 애그리게이트 내 버전(낙관적 락)
    type = models.CharField(max_length=80)            # 예: "OrderCreated"
    payload = models.JSONField()
    created_at = models.DateTimeField(auto_now_add=True)

    class Meta:
        unique_together = [("aggregate_id", "version")]
        indexes = [models.Index(fields=["stream","aggregate_id","version"])]
```

스냅샷:

```python
class Snapshot(models.Model):
    stream = models.CharField(max_length=80)
    aggregate_id = models.CharField(max_length=64, db_index=True)
    version = models.PositiveIntegerField()
    state = models.JSONField()                        # 애그리게이트 직렬화
    created_at = models.DateTimeField(auto_now_add=True)

    class Meta:
        unique_together = [("stream","aggregate_id")]
```

---

### 2-3. 애그리게이트 정의(순수 파이썬)

```python
# apps/orders/domain/es_entities.py
from dataclasses import dataclass, field

@dataclass
class OrderAgg:
    id: int | None = None
    user_id: int | None = None
    status: str = "CREATED"
    total: int = 0
    items: list[dict] = field(default_factory=list)
    version: int = 0

    def apply(self, event_type: str, payload: dict):
        if event_type == "OrderCreated":
            self.id = payload["order_id"]
            self.user_id = payload["user_id"]
            self.total = payload["total"]
            self.items = payload["items"]
            self.status = "CREATED"
        elif event_type == "OrderPaid":
            self.status = "PAID"
        elif event_type == "OrderCancelled":
            self.status = "CANCELLED"
        self.version += 1

    @classmethod
    def replay(cls, events: list[tuple[str, dict]]):
        agg = cls()
        for et, pl in events:
            agg.apply(et, pl)
        return agg
```

---

### 2-4. 저장소(이벤트 저장 & 동시성 제어)

```python
# apps/orders/infra/es_repository.py
from django.db import transaction
from shared.es.models import Event, Snapshot
from apps.orders.domain.es_entities import OrderAgg

class OrderAggRepository:
    STREAM = "orders"

    def load(self, order_id: int) -> OrderAgg:
        # 스냅샷 우선
        snap = Snapshot.objects.filter(stream=self.STREAM, aggregate_id=str(order_id)).first()
        start_version = 0
        agg = OrderAgg()
        if snap:
            agg = OrderAgg(**snap.state)
            start_version = snap.version

        evs = Event.objects.filter(stream=self.STREAM, aggregate_id=str(order_id), version__gt=start_version).order_by("version")
        for ev in evs:
            agg.apply(ev.type, ev.payload)
        return agg

    def append(self, order_id: int, expected_version: int, new_events: list[tuple[str, dict]]):
        with transaction.atomic():
            # 낙관적 락: 현재 마지막 버전 조회
            last = Event.objects.filter(stream=self.STREAM, aggregate_id=str(order_id)).order_by("-version").first()
            current_version = last.version if last else 0
            if current_version != expected_version:
                raise ConcurrencyError(f"expected {expected_version} but {current_version}")

            v = current_version
            for et, pl in new_events:
                v += 1
                Event.objects.create(
                    stream=self.STREAM,
                    aggregate_id=str(order_id),
                    version=v, type=et, payload=pl
                )
            # 스냅샷 정책(예: 100개마다)
            if v % 100 == 0:
                agg = self.load(order_id)
                Snapshot.objects.update_or_create(
                    stream=self.STREAM, aggregate_id=str(order_id),
                    defaults={"version": agg.version, "state": agg.__dict__}
                )
```

`ConcurrencyError` 정의:

```python
class ConcurrencyError(RuntimeError): pass
```

---

### 2-5. Command Handler (ES 버전)

```python
# apps/orders/app/es_service.py
from apps.orders.infra.es_repository import OrderAggRepository
from shared.outbox.api import add_outbox_event

class EOrderService:
    def __init__(self):
        self.repo = OrderAggRepository()

    def create_order(self, order_id: int, user_id: int, items: list[dict]):
        total = sum(it["price"] * it["qty"] for it in items)
        events = [("OrderCreated", {"order_id": order_id, "user_id": user_id, "total": total, "items": items})]
        self.repo.append(order_id, expected_version=0, new_events=events)
        # 읽기 모델/외부 통지용 outbox
        add_outbox_event("orders","OrderCreated",str(order_id),events[0][1])

    def pay_order(self, order_id: int):
        agg = self.repo.load(order_id)
        if agg.status == "PAID": return
        self.repo.append(order_id, expected_version=agg.version, new_events=[("OrderPaid",{"order_id":order_id})])
        add_outbox_event("orders","OrderPaid",str(order_id),{"order_id": order_id})
```

> **핵심**: 쓰기 저장은 “상태를 저장”하지 않고 **이벤트**만 저장합니다. 현재 상태는 **재생**을 통해 얻습니다(스냅샷으로 가속).

---

### 2-6. 리플레이/백필/재빌드

- “읽기 모델 파손/스키마 변경” 시: **Outbox** 또는 **Event Store** 를 **처음부터 재생**하여 새 읽기 모델을 재구축.

```python
# apps/readmodels/orders/management/commands/rebuild_read_model.py
from django.core.management.base import BaseCommand
from shared.outbox.models import OutboxEvent
from apps.readmodels.orders.infra.projector import HANDLERS

class Command(BaseCommand):
    help = "읽기 모델 전체 재구축(Outbox 기준)"

    def handle(self, *args, **kwargs):
        # 1) 기존 읽기 모델 초기화
        from apps.readmodels.orders.infra.models import OrderSummary
        OrderSummary.objects.all().delete()

        # 2) Outbox 전체 재생 (또는 shared.es.models.Event 에서)
        for ev in OutboxEvent.objects.filter(topic="orders").order_by("id").iterator(chunk_size=1000):
            try: HANDLERS[ev.type](ev)
            except Exception as e: self.stderr.write(f"ev#{ev.id} 실패: {e}")
        self.stdout.write(self.style.SUCCESS("재구축 완료"))
```

---

### 2-7. 이벤트 버전 관리(업그레이드/다운그레이드)

- 이벤트 스키마는 **불변**이 이상적이나 현실적으로 진화합니다.  
- **버전 필드** 또는 **type 네이밍**(`OrderCreated.v2`)으로 구분하고, Projection/Replay 시 **업캐스트(Upcast)** 합니다.

```python
def upcast(ev):
    # 예: v1: {"order_id", "user_id", "total"}
    # v2: {"order_id","user_id","total","currency"}
    if ev.type == "OrderCreated" and "currency" not in ev.payload:
        ev.payload["currency"] = "KRW"
    return ev
```

모든 소비자는 `upcast(ev)`를 거쳐 처리.

---

### 2-8. ES 적용 판단 기준

**적합**
- 규제/감사 추적 필요(모든 변경 이력)  
- 과거 시점 재현/분석(“그때 상태로 돌아가 보기”)  
- 복잡한 상태 전이/사가(Saga)와의 결합

**주의**
- 저장소/리플레이/스냅샷/업캐스트 등 **운영 비용**이 큼  
- 모든 애그리게이트에 무차별 적용 금지 → **핵심만 선택**

---

## 3) 모놀리스 → 모듈/서비스 분해 전략

> 목표는 “마이크로서비스”가 아니라 **경계 명확화 + 독립 배포 가능한 모듈**로의 안전한 단계적 전환입니다.

### 3-1. 분해 전 진단 — 경계 도출

1) **유스케이스/데이터 흐름**을 **행렬**로 적어 **강한 응집**과 **약한 결합** 영역을 찾습니다.  
2) “함께 변경되는 것”을 묶고, “서로 기다리는 것”을 분리: **변경률/규모/팀** 기준.  
3) **공유 DB 안티패턴** 파악: 테이블/조인을 무차별 공유하고 있다면 경계가 흐립니다.

산출물 예:
```
[accounts] —(user_id)→ [orders]
[catalog]  —(product_id, price snapshot)→ [orders]
[payments] ←→ [orders] (사가)
[readmodels] (독립 생성 가능)
```

---

### 3-2. 1단계: 모놀리스 내부 모듈화

- 앞선 **기능 중심 + 레이어 구조**로 앱 디렉터리 재편.  
- **UI→APP→DOMAIN**, **INFRA**는 구현.  
- **의존성 규칙 테스트**(import lint)로 구조 강제.

간단한 import 린터(예시):

```python
# tools/check_imports.py
import os, ast, sys, re
RULES = [
    (re.compile(r"apps\.(\w+)\.ui\."), re.compile(r"apps\.\1\.app|apps\.\1\.domain|apps\.shared")),
    (re.compile(r"apps\.(\w+)\.app\."), re.compile(r"apps\.\1\.domain|apps\.shared")),
]
def main():
    ok = True
    for root, _, files in os.walk("apps"):
        for f in files:
            if not f.endswith(".py"): continue
            code = open(os.path.join(root,f),"r").read()
            try: tree = ast.parse(code)
            except: continue
            for node in ast.walk(tree):
                if isinstance(node, ast.ImportFrom) and node.module:
                    mod = node.module
                    for src, allowed in RULES:
                        m = src.match(mod)
                        if m and not allowed.match(mod):
                            print("Import violation:", mod, "in", os.path.join(root,f))
                            ok = False
    sys.exit(0 if ok else 1)
if __name__ == "__main__": main()
```

CI에서 실행해 구조 일관성 보장.

---

### 3-3. 2단계: 읽기 모델/외부 통지 분리

- **Outbox + Projector** 를 먼저 도입하면 **데이터 쓰기 경로**를 건드리지 않고 **읽기 경로**를 독립화할 수 있습니다.  
- 외부 통지(Slack, 결제 웹훅, 검색 인덱싱)는 **Outbox 소비자**로 이동.

이 단계만 해도:
- 보고서/목록/검색 부하가 **앱 DB**에서 **읽기 DB/캐시**로 이동.  
- 배포/장애 영향 반경 축소.

---

### 3-4. 3단계: 도메인 경계 단위 독립 서비스화(선별)

**원칙**
- 데이터는 **각 서비스 소유**. 공유 DB 금지.  
- 데이터 필요 시 **이벤트 복제**(CDC/Outbox) 또는 **동기 조회(조심)**.

**예시**: `payments` 를 분리
- `orders` 는 “결제 필요” 이벤트 발행  
- `payments` 가 처리 후 “결제 승인” 이벤트 발행  
- `orders` 가 소비해 상태 전이(사가 코디네이션)

사가(간단 흐름):

```
Orders: OrderCreated → (Outbox) → Payments: Authorize → (Outbox) → Orders: OrderPaid
```

Transact 보상:
- 실패 시 `OrderCancelled` 이벤트.

---

### 3-5. 통신/트랜잭션 전략

- **비동기 우선**: Outbox(HTTP/Webhook, 큐)를 통해 최종 일관성.  
- **동기 호출**은 최소화: 진짜 필요할 때만(예: 사용자 프로필 조회).  
- **2PC 금지**, 대신 **사가/보상 트랜잭션** 사용.

API 계약/스키마:
- 이벤트/DTO에 **버전** 필드 부여.  
- 하위호환 유지 → 제거는 **마지막 단계**.

---

### 3-6. 배포/운영 전환

- **블루-그린/카나리**로 새 경로 점진 전환.  
- **Dual write 금지**: 쓰기는 항상 **소스 서비스**에서만.  
- 읽기 경로는 점진적으로 **새 readmodel**을 사용하도록 플래그 전환.

피처 플래그(간단):

```python
# shared/flags.py
import os
def enabled(name: str) -> bool:
    return os.getenv(f"FLAG_{name.upper()}", "0") == "1"
```

뷰에서:

```python
from shared.flags import enabled
if enabled("NEW_READMODEL_ORDERS"):
    # OrderSummary 조회
else:
    # 기존 조인 쿼리
```

---

### 3-7. 데이터 마이그레이션/백필

- 서비스 분리 시 **새 저장소**로 데이터 **백필(backfill)** → 신규 쓰기는 새쪽에서만.  
- 백필 절차:
  1) 과거 데이터 bulk export/import  
  2) Outbox 이벤트를 **백필 기준일 이후**만 소비  
  3) 양쪽 지표 대조(샘플링 검증)  
  4) 플래그 전환

---

### 3-8. 관측성/오류복구

- **Outbox 적체 알람**: 레이턴시 상한, 배치 사이즈 조정  
- **Saga 실패 경로** 대시보드/재시도 버튼(운영 UI)  
- 각 서비스 **트레이스 상관관계**(TraceID 전파)

구조화 로그(요약):

```python
log.info("saga.order_payment.begin", order_id=oid, trace_id=tid)
...
log.info("saga.order_payment.end", order_id=oid, status="PAID", trace_id=tid)
```

---

## 4) 테스트 전략

### 4-1. 단위 테스트

- **도메인/서비스**는 **인메모리 리포지토리**/페이크 게이트웨이로 테스트.  
- 이벤트 소싱은 **이벤트 시퀀스**를 준비해 `replay` 결과를 검증.

```python
def test_replay_paid():
    events = [("OrderCreated", {...}), ("OrderPaid", {"order_id":1})]
    agg = OrderAgg.replay(events)
    assert agg.status == "PAID"
```

### 4-2. 통합 테스트

- Outbox 기록 → Projector 실행 → ReadModel 업데이트 검증.

```python
def test_projection_updates_readmodel(db):
    oid = OrderService().create_order(1, items=[...])
    run_projection()
    assert OrderSummary.objects.filter(order_id=oid).exists()
```

### 4-3. 회귀/리플레이 테스트

- 실제 운영 Outbox/Event Store의 **샘플**을 테스트에 주입해 **리플레이 실패 케이스**를 잡아냅니다.

---

## 5) 보안/컴플라이언스 고려

- 이벤트 payload에 **개인정보 최소화**(키만, 본문은 tokenized).  
- **삭제 요청(DRR)**: 이벤트 소싱 환경에선 삭제가 어려움 → **암호화/키 폐기** 방식, 또는 **파생 읽기 모델**에서 삭제·마스킹.  
- 감사를 위해 **Event Store 접근 권한 최소화**, 변경 불가(append-only) 보장.

---

## 6) 운영 체크리스트 (요약)

**CQRS**
- [ ] Outbox + Projector 도입  
- [ ] 읽기 모델 idempotent 적용  
- [ ] 전체 재빌드 커맨드/잡 마련  
- [ ] 이벤트/프로젝션 버전 업캐스트 체계

**이벤트 소싱(선별)**
- [ ] Event Store/스냅샷 정책  
- [ ] 낙관적 락(버전) 충돌 처리  
- [ ] 리플레이/스키마 업캐스트/백필 스크립트  
- [ ] 모니터링(리플레이 시간, 스냅샷 간격)

**모듈 분해**
- [ ] 경계 도출(응집/결합) 문서화  
- [ ] 모놀리스 내부 모듈화 + import 린팅  
- [ ] 읽기 경로 분리 → 사가로 일부 서비스 분리  
- [ ] 플래그/카나리/롤백 절차

---

## 7) 부록: End-to-End 시나리오 (요약 코드)

1) **주문 생성(쓰기)** → Order/OrderItem 저장 + Outbox(`OrderCreated`).  
2) **Projector** → `OrderSummary` upsert.  
3) **결제 승인** → Outbox(`OrderPaid`) → Projector가 `status=PAID` 반영.  
4) **서비스 분리**: Payments가 `OrderPaymentAuthorized` 발행 → Orders가 소비.  
5) **리플레이**: 신규 통계 읽기 모델 추가 시 Outbox 전체 재생으로 채움.

---

## 8) FAQ

- **Django에서 CQRS가 과하지 않나요?**  
  - CRUD 위주 소규모 앱이면 과합니다. **읽기 트래픽이 크거나 목록/검색이 복잡한 핵심 흐름**만 부분 적용하세요.

- **ES와 Outbox를 둘 다 써야 하나요?**  
  - 반드시 둘 다는 아닙니다. **ES는 쓰기 저장 자체를 이벤트로** 바꾸는 선택입니다. Outbox는 **트랜잭션 외부 전달**과 **읽기 모델 반영**을 안전하게 하기 위한 범용 패턴입니다.

- **최종 일관성으로 사용자 혼란이 생기면?**  
  - “처리 중” 상태 UI/토스트, **일시적 카운트 차이 안내**, 새로고침/폴링/푸시로 보완하세요. SLA에 맞게 Projector 빈도/배치 크기 조정.

- **이벤트 버전 충돌/업캐스트가 복잡합니다.**  
  - **새 필드 추가 → 기본값 상수화 → 소비자에 업캐스터 추가** 순으로 **점진 이행**을 습관화하세요.

---

## 9) 마무리

- **CQRS** 로 읽기 경로를 가볍고 빠르게, **쓰기 경로**는 도메인 규칙과 트랜잭션에 집중시킵니다.  
- **선별적 이벤트 소싱**은 진짜 필요한 애그리게이트에만 도입하여 **감사/재생/확장성**을 확보하세요.  
- **모놀리스 → 모듈/서비스 분해**는 **Outbox, 읽기 모델, 사가**를 축으로 **작고 안전한 단계**로 진행하는 것이 성공의 지름길입니다.
