---
layout: post
title: Django - 운영 대시보드 · Admin 성능 최적화
date: 2025-10-06 17:25:23 +0900
category: Django
---
# 운영 대시보드 · Admin 성능 최적화(비동기 대량 액션) · 멀티테넌트 Admin 분리

## 1. 운영 대시보드

### 1-1. 데이터 모델(요약) — KPI · 감사 로그

```python
# apps/ops/models.py
from django.db import models
from django.conf import settings

class DailyKPI(models.Model):
    day = models.DateField(unique=True, db_index=True)
    signups = models.PositiveIntegerField(default=0)
    dau = models.PositiveIntegerField(default=0)          # Daily Active Users
    orders = models.PositiveIntegerField(default=0)
    revenue = models.BigIntegerField(default=0)           # KRW
    churned = models.PositiveIntegerField(default=0)

    class Meta:
        indexes = [models.Index(fields=["-day"])]

class AuditLog(models.Model):
    actor = models.ForeignKey(settings.AUTH_USER_MODEL, on_delete=models.SET_NULL, null=True)
    ip = models.GenericIPAddressField(null=True, blank=True)
    path = models.CharField(max_length=255)
    method = models.CharField(max_length=8)
    action = models.CharField(max_length=80)              # e.g. admin.create_product
    target_model = models.CharField(max_length=100, blank=True)
    target_pk = models.CharField(max_length=64, blank=True)
    meta = models.JSONField(default=dict, blank=True)
    created_at = models.DateTimeField(auto_now_add=True)

    class Meta:
        indexes = [
            models.Index(fields=["-created_at"]),
            models.Index(fields=["actor", "-created_at"]),
            models.Index(fields=["action"])
        ]
```

#### 감사 로그 미들웨어

```python
# apps/ops/middleware.py
from .models import AuditLog
from django.utils.deprecation import MiddlewareMixin

SAFE_PATH_PREFIXES = ("/admin/", "/api/admin/")

class AuditLogMiddleware(MiddlewareMixin):
    def process_view(self, request, view_func, view_args, view_kwargs):
        request._audit_ctx = {
            "actor": getattr(request, "user", None) if request.user.is_authenticated else None,
            "ip": request.META.get("REMOTE_ADDR"),
            "path": request.path,
            "method": request.method,
        }
        return None

    def process_response(self, request, response):
        ctx = getattr(request, "_audit_ctx", None)
        if not ctx:
            return response
        # 관리/운영 경로만 기록(필요 시 확장)
        if any(request.path.startswith(p) for p in SAFE_PATH_PREFIXES):
            try:
                AuditLog.objects.create(
                    actor=ctx["actor"],
                    ip=ctx["ip"],
                    path=ctx["path"],
                    method=ctx["method"],
                    action=getattr(request, "_audit_action", "") or "",
                    target_model=getattr(request, "_audit_target_model", ""),
                    target_pk=str(getattr(request, "_audit_target_pk", "") or ""),
                    meta=getattr(request, "_audit_meta", {}) or {},
                )
            except Exception:
                pass
        return response
```

**뷰/액션**에서 추가 컨텍스트를 남기고 싶을 때:

```python
# 예: 관리자에서 특정 상품을 수정
def admin_update_product(request, pk):
    # ... 업데이트 로직 ...
    request._audit_action = "admin.update_product"
    request._audit_target_model = "shop.Product"
    request._audit_target_pk = pk
    request._audit_meta = {"fields": ["price", "stock"]}
    # 응답
```

> 운영 팁: 개인정보/민감정보는 **메타에 남기지 않도록** 필터링.

---

### 1-2. 접근 제어 — RBAC(역할 기반) & 오브젝트 레벨

```python
# apps/ops/roles.py
from enum import StrEnum
class Role(StrEnum):
    SUPERADMIN = "superadmin"
    OPS = "ops"
    PARTNER_ADMIN = "partner_admin"
```

```python
# apps/accounts/models.py (예시)
class RoleBinding(models.Model):
    user = models.ForeignKey(settings.AUTH_USER_MODEL, on_delete=models.CASCADE)
    role = models.CharField(max_length=40)  # Role 값
    tenant_id = models.PositiveIntegerField(null=True, blank=True)  # 멀티테넌트용
```

권한 헬퍼:

```python
# apps/ops/perm.py
from .roles import Role

def has_role(user, role: str, tenant_id: int | None = None) -> bool:
    if getattr(user, "is_superuser", False):
        return True
    qs = user.rolebinding_set.filter(role=role)
    if tenant_id is not None:
        qs = qs.filter(tenant_id=tenant_id)
    return qs.exists()
```

데코레이터:

```python
from django.http import HttpResponseForbidden
from functools import wraps
from .perm import has_role

def role_required(role: str):
    def inner(fn):
        @wraps(fn)
        def wrapped(request, *args, **kwargs):
            tenant_id = request.GET.get("tenant") or request.session.get("tenant_id")
            if not has_role(request.user, role, int(tenant_id) if tenant_id else None):
                return HttpResponseForbidden("권한이 없습니다.")
            return fn(request, *args, **kwargs)
        return wrapped
    return inner
```

---

### 1-3. 운영 대시보드 뷰(TemplateView) — 카드, 테이블, 필터

```python
# apps/ops/views.py
from django.views.generic import TemplateView, ListView
from django.utils import timezone
from datetime import timedelta
from .models import DailyKPI, AuditLog
from .perm import has_role
from .roles import Role

class OpsDashboardView(TemplateView):
    template_name = "ops/dashboard.html"

    def dispatch(self, request, *args, **kwargs):
        if not has_role(request.user, Role.OPS):
            return self.handle_no_permission()
        return super().dispatch(request, *args, **kwargs)

    def handle_no_permission(self):
        from django.http import HttpResponseForbidden
        return HttpResponseForbidden("OPS 권한 필요")

    def get_context_data(self, **kwargs):
        ctx = super().get_context_data(**kwargs)
        today = timezone.localdate()
        days = [today - timedelta(days=i) for i in range(0, 14)][::-1]
        kpis = DailyKPI.objects.filter(day__in=days).order_by("day")
        # 차트/카드용 데이터
        ctx["series"] = {
            "labels": [d.isoformat() for d in days],
            "signups": [next((k.signups for k in kpis if k.day == d), 0) for d in days],
            "dau": [next((k.dau for k in kpis if k.day == d), 0) for d in days],
            "revenue": [next((k.revenue for k in kpis if k.day == d), 0) for d in days],
        }
        # 최근 감사 로그
        ctx["logs"] = AuditLog.objects.select_related("actor").order_by("-created_at")[:50]
        return ctx

class AuditLogListView(ListView):
    model = AuditLog
    paginate_by = 50
    template_name = "ops/audit_list.html"

    def get_queryset(self):
        qs = super().get_queryset().select_related("actor").order_by("-created_at")
        actor = self.request.GET.get("actor")
        action = self.request.GET.get("action")
        if actor:
            qs = qs.filter(actor_id=actor)
        if action:
            qs = qs.filter(action__icontains=action)
        return qs
```

템플릿(요약):

```html
{# templates/ops/dashboard.html #}
<h1>운영 대시보드</h1>
<div class="cards">
  <div class="card">오늘 가입: {{ series.signups|last }}</div>
  <div class="card">오늘 DAU: {{ series.dau|last }}</div>
  <div class="card">오늘 매출: {{ series.revenue|last|intcomma }}원</div>
</div>

<canvas id="chart1"></canvas>
<script>
  // Chart.js 등 사용 가능(정적 번들에 포함)
  const labels = {{ series.labels|safe }};
  const signups = {{ series.signups|safe }};
  // ... 차트 초기화 ...
</script>

<h2>최근 감사 로그</h2>
<table>
  <thead><tr><th>시각</th><th>사용자</th><th>행동</th><th>경로</th><th>IP</th></tr></thead>
  <tbody>
    {% for log in logs %}
      <tr>
        <td>{{ log.created_at }}</td>
        <td>{{ log.actor or "-" }}</td>
        <td>{{ log.action }}</td>
        <td>{{ log.path }}</td>
        <td>{{ log.ip }}</td>
      </tr>
    {% endfor %}
  </tbody>
</table>
```

> **성능 팁**: KPI는 실시간 계산 대신 **야간 배치**(Celery Beat)로 `DailyKPI`에 저장. 감사 로그 조회는 **필터/인덱스** 기반.

---

### 1-4. KPI 산출 배치 (Celery Beat)

```python
# apps/ops/tasks.py
from celery import shared_task
from django.utils import timezone
from datetime import timedelta
from apps.accounts.models import User
from apps.orders.models import Order
from .models import DailyKPI

@shared_task
def build_daily_kpi(day=None):
    day = day or timezone.localdate() - timedelta(days=1)
    start = timezone.make_aware(timezone.datetime.combine(day, timezone.datetime.min.time()))
    end = timezone.make_aware(timezone.datetime.combine(day, timezone.datetime.max.time()))
    signups = User.objects.filter(date_joined__date=day).count()
    dau = User.objects.filter(last_login__range=(start, end)).count()
    orders = Order.objects.filter(created_at__date=day).count()
    revenue = (Order.objects.filter(created_at__date=day, status="PAID")
               .aggregate(s=models.Sum("amount"))["s"] or 0)
    DailyKPI.objects.update_or_create(day=day, defaults={
        "signups": signups, "dau": dau, "orders": orders, "revenue": revenue
    })
```

---

## 2. Admin 성능 최적화 — “대량 액션 비동기화”

### 2-1. 문제가 되는 패턴
- 수만 개 객체에 대해 **Admin 액션**에서 바로 `.update()`/루프 처리 → **타임아웃/응답 지연**.
- 해결: **액션은 큐에 태스크 등록**만 하고 **즉시 응답**. 진행 상황은 **상태 모델**로 추적.

상태 모델:

```python
# apps/ops/models.py (추가)
class BulkJob(models.Model):
    name = models.CharField(max_length=120)           # 예: "discount_10"
    actor = models.ForeignKey(settings.AUTH_USER_MODEL, on_delete=models.SET_NULL, null=True)
    queryset_repr = models.TextField()                # 필터 요약/ID 리스트
    total = models.PositiveIntegerField(default=0)
    processed = models.PositiveIntegerField(default=0)
    status = models.CharField(max_length=20, default="PENDING")  # PENDING/RUNNING/DONE/FAILED
    created_at = models.DateTimeField(auto_now_add=True)
    started_at = models.DateTimeField(null=True, blank=True)
    finished_at = models.DateTimeField(null=True, blank=True)
    meta = models.JSONField(default=dict, blank=True)
```

Admin 액션 → 큐 등록:

```python
# apps/shop/admin_actions.py (비동기 버전)
from django.contrib import messages
from django.urls import reverse
from django.utils import timezone
from apps.ops.models import BulkJob
from apps.shop.tasks import bulk_discount_10_task

def bulk_discount_10_async(modeladmin, request, queryset):
    ids = list(queryset.values_list("id", flat=True))
    job = BulkJob.objects.create(
        name="discount_10",
        actor=request.user,
        queryset_repr=f"Product ids: {ids[:5]}{'...' if len(ids)>5 else ''}",
        total=len(ids),
        status="PENDING",
        meta={"percent": 10}
    )
    bulk_discount_10_task.delay(job.id, ids)
    messages.success(request, f"대량 할인 작업을 시작했습니다 (총 {len(ids)}개). 진행상황은 운영 대시보드에서 확인하세요.")
bulk_discount_10_async.short_description = "선택 상품 10% 할인(비동기)"
```

Celery 태스크:

```python
# apps/shop/tasks.py
from celery import shared_task
from django.db import transaction
from django.utils import timezone
from django.db.models import F
from apps.ops.models import BulkJob
from .models import Product

@shared_task(bind=True, autoretry_for=(Exception,), retry_backoff=True, max_retries=3)
def bulk_discount_10_task(self, job_id, ids: list[int]):
    job = BulkJob.objects.get(pk=job_id)
    job.status = "RUNNING"
    job.started_at = timezone.now()
    job.save(update_fields=["status", "started_at"])

    processed = 0
    CHUNK = 500
    try:
        for i in range(0, len(ids), CHUNK):
            chunk = ids[i:i+CHUNK]
            with transaction.atomic():
                Product.objects.filter(id__in=chunk).update(price=F("price") * 0.9)
            processed += len(chunk)
            BulkJob.objects.filter(pk=job_id).update(processed=processed)
        BulkJob.objects.filter(pk=job_id).update(status="DONE", finished_at=timezone.now())
    except Exception as e:
        BulkJob.objects.filter(pk=job_id).update(status="FAILED", finished_at=timezone.now(), meta={"error": str(e)})
        raise
```

상태 확인 페이지(간단):

```python
# apps/ops/views.py (추가)
from django.views.generic import ListView
from .models import BulkJob

class BulkJobListView(ListView):
    model = BulkJob
    paginate_by = 20
    template_name = "ops/bulk_jobs.html"
    ordering = "-created_at"
```

템플릿:

```html
{# templates/ops/bulk_jobs.html #}
<h1>대량 작업</h1>
<table>
  <thead><tr><th>시간</th><th>작업</th><th>상태</th><th>진행</th><th>요청자</th></tr></thead>
  <tbody>
  {% for j in object_list %}
    <tr>
      <td>{{ j.created_at }}</td>
      <td>{{ j.name }}</td>
      <td>{{ j.status }}</td>
      <td>{{ j.processed }}/{{ j.total }}</td>
      <td>{{ j.actor }}</td>
    </tr>
  {% endfor %}
  </tbody>
</table>
```

> **핵심**:  
> - Admin 액션은 **큐 등록**만.  
> - 작업은 **CHUNK + 트랜잭션**으로 나누어 처리.  
> - 진행률은 **상태 테이블** 업데이트.  
> - 실패 시 **재시도/로그**.

---

### 2-2. Admin Changelist에서 “대량 작업 보기” 버튼

```python
# apps/shop/admin.py
from django.urls import reverse
from django.utils.html import format_html
from django.contrib import admin
from .admin_actions import bulk_discount_10_async
from apps.ops.models import BulkJob

@admin.register(Product)
class ProductAdmin(admin.ModelAdmin):
    actions = [bulk_discount_10_async]
    change_list_template = "admin/shop/product/change_list.html"

    def changelist_view(self, request, extra_context=None):
        extra_context = extra_context or {}
        extra_context["bulk_jobs_url"] = reverse("ops:bulk_jobs")
        return super().changelist_view(request, extra_context)
```

```html
{# templates/admin/shop/product/change_list.html #}
{% extends "admin/change_list.html" %}
{% block object-tools-items %}
  {{ block.super }}
  <li><a href="{{ bulk_jobs_url }}">대량 작업 상태</a></li>
{% endblock %}
```

---

### 2-3. 기타 성능 최적화 체크
- `list_select_related`/`prefetch_related` 적극 사용.  
- `search_fields` 에서 **대용량 텍스트**는 피하고, 필요한 경우 **Trigram/GIN 인덱스**.  
- 대량 Export는 **비동기 파일 생성(스토리지 저장) + 서명 URL 다운로드**.

---

## 3. 멀티테넌트 Admin 분리

목표:
- 테넌트(고객사/조직)별로 **데이터 경계**.  
- 테넌트 관리자(Partner Admin)는 **자기 테넌트 데이터만** 보게.  
- 내부 운영 Admin은 **전체** 접근.

### 3-1. 테넌트 모델/스코프

```python
# apps/tenancy/models.py
from django.db import models
from django.conf import settings

class Tenant(models.Model):
    name = models.CharField(max_length=120, unique=True)
    slug = models.SlugField(unique=True)
    created_at = models.DateTimeField(auto_now_add=True)

class TenantUser(models.Model):
    tenant = models.ForeignKey(Tenant, on_delete=models.CASCADE)
    user = models.ForeignKey(settings.AUTH_USER_MODEL, on_delete=models.CASCADE)
    role = models.CharField(max_length=40, default="member")  # admin/member 등
```

도메인 모델에 `tenant` 필드 도입(예: `Shop`, `Product`):

```python
# apps/shop/models.py (일부)
class Shop(models.Model):
    tenant = models.ForeignKey("tenancy.Tenant", on_delete=models.PROTECT, related_name="shops")
    name = models.CharField(max_length=120)
    # ...

class Product(models.Model):
    tenant = models.ForeignKey("tenancy.Tenant", on_delete=models.PROTECT, related_name="products")
    shop = models.ForeignKey(Shop, on_delete=models.CASCADE, related_name="products")
    # ...
```

> 가능한 한 모든 비즈니스 테이블에 `tenant` FK를 포함해 **쿼리 가드**를 단순화.

---

### 3-2. AdminSite 분리 — 내부용 / 파트너용

```python
# apps/core/adminsites.py
from django.contrib.admin import AdminSite

class InternalAdminSite(AdminSite):
    site_header = "내부 운영 Admin"
    site_title = "Internal Admin"

class PartnerAdminSite(AdminSite):
    site_header = "파트너 Admin"
    site_title = "Partner Admin"

internal_admin = InternalAdminSite(name="internal_admin")
partner_admin = PartnerAdminSite(name="partner_admin")
```

등록:

```python
# apps/shop/admin_internal.py
from django.contrib import admin
from apps.core.adminsites import internal_admin
from .models import Product, Shop
@internal_admin.register(Product)
class ProductAdmin(admin.ModelAdmin):
    list_display = ("id","tenant","shop","name","price","is_public")

@internal_admin.register(Shop)
class ShopAdmin(admin.ModelAdmin):
    list_display = ("id","tenant","name")
```

```python
# apps/shop/admin_partner.py
from django.contrib import admin
from apps.core.adminsites import partner_admin
from .models import Product, Shop

class TenantScopedMixin:
    """요청 사용자의 tenant 스코프만 보여주도록 쿼리 가드"""
    def get_queryset(self, request):
        qs = super().get_queryset(request)
        tenant = getattr(request, "tenant", None)
        if tenant and not request.user.is_superuser:
            qs = qs.filter(tenant=tenant)
        return qs

@partner_admin.register(Product)
class ProductPartnerAdmin(TenantScopedMixin, admin.ModelAdmin):
    list_display = ("id","shop","name","price","is_public")
    list_select_related = ("shop",)
    def save_model(self, request, obj, form, change):
        if not change:
            obj.tenant = getattr(request, "tenant")
        super().save_model(request, obj, form, change)

@partner_admin.register(Shop)
class ShopPartnerAdmin(TenantScopedMixin, admin.ModelAdmin):
    list_display = ("id","name")
    def save_model(self, request, obj, form, change):
        if not change:
            obj.tenant = getattr(request, "tenant")
        super().save_model(request, obj, form, change)
```

URL 라우팅:

```python
# config/urls.py
from django.urls import path
from apps.core.adminsites import internal_admin, partner_admin

urlpatterns = [
    path("admin/", internal_admin.urls),      # 내부 운영
    path("partner/", partner_admin.urls),     # 파트너
]
```

---

### 3-3. 요청에 tenant 주입 — 서브도메인/헤더/세션

```python
# apps/tenancy/middleware.py
from .models import Tenant
from django.http import HttpResponseForbidden

class TenantResolverMiddleware:
    def __init__(self, get_response): self.get_response = get_response
    def __call__(self, request):
        host = request.get_host().split(":")[0]
        # 예: <slug>.partner.example.com
        parts = host.split(".")
        tenant = None
        if len(parts) >= 3 and parts[-3] != "partner":  # app.partner.example.com
            slug = parts[0]
            tenant = Tenant.objects.filter(slug=slug).first()
        # 또는 세션/헤더 기반
        request.tenant = tenant
        return self.get_response(request)
```

파트너 Admin 접근 가드:

```python
# apps/core/adminsites.py (파트너 전용 권한)
class PartnerAdminSite(AdminSite):
    site_header = "파트너 Admin"; site_title = "Partner Admin"
    def has_permission(self, request):
        # tenant 존재 + TenantUser 권한 체크
        if not getattr(request, "tenant", None):
            return False
        return request.user.is_authenticated and request.user.tenantuser_set.filter(tenant=request.tenant, role__in=["admin","manager"]).exists()
```

> 내부 Admin은 `is_staff`/`is_superuser` 로 통상 허용.

---

### 3-4. 테넌트 스코프 보장 — ModelAdmin 가드 + 폼

- `get_queryset()` 에서 **항상** `tenant=...` 필터.  
- `save_model()` 신규 생성 시 `obj.tenant = request.tenant`.  
- 인라인/외래키 필드에도 `formfield_for_foreignkey()` 로 테넌트 필터.

```python
class TenantScopedMixin(admin.ModelAdmin):
    def get_queryset(self, request):
        qs = super().get_queryset(request)
        tenant = getattr(request, "tenant", None)
        return qs if (request.user.is_superuser or not tenant) else qs.filter(tenant=tenant)

    def formfield_for_foreignkey(self, db_field, request, **kwargs):
        tenant = getattr(request, "tenant", None)
        if tenant and db_field.name in ("shop","category"):
            qs = db_field.remote_field.model.objects.filter(tenant=tenant)
            kwargs["queryset"] = qs
        return super().formfield_for_foreignkey(db_field, request, **kwargs)
```

> **테스트 필수**: 파트너 Admin에서 다른 테넌트 레코드가 보이지 않는지.

---

### 3-5. 멀티테넌트 감사 로그

```python
# apps/ops/middleware.py (추가)
def process_response(self, request, response):
    # ...
    if any(request.path.startswith(p) for p in SAFE_PATH_PREFIXES):
        tenant_id = getattr(request, "tenant", None)
        AuditLog.objects.create(
            # ...
            meta={"tenant": getattr(tenant_id, "id", None), **(getattr(request,"_audit_meta",{}) or {})}
        )
    return response
```

> 감사 로그는 **tenant** 를 포함하여 검색/이슈 추적을 쉽게.

---

## 4. 운영 대시보드 + Admin 통합 UX

- 내부 Admin의 메뉴에 **운영 대시보드 링크** 추가.  
- 파트너 Admin에도 **자기 테넌트 대시보드** 제공(쿼리 자동 필터).

```python
# templates/admin/base.html (또는 change_list 확장)
<li><a href="{% url 'ops:dashboard' %}">운영 대시보드</a></li>
<li><a href="{% url 'ops:bulk_jobs' %}">대량 작업</a></li>
```

파트너 버전 대시보드:

```python
class PartnerDashboardView(TemplateView):
    template_name = "ops/partner_dashboard.html"
    def get_context_data(self, **kwargs):
        ctx = super().get_context_data(**kwargs)
        t = getattr(self.request, "tenant", None)
        # KPI를 tenant별로 저장/집계해두었다면 필터:
        days = ...  # 위와 동일
        kpis = DailyKPI.objects.filter(day__in=days, **({"tenant": t} if t else {})).order_by("day")
        # ...
        return ctx
```

---

## 5. 모니터링/알림과의 연결

- **BulkJob 상태 변화** 시 `transaction.on_commit` 으로 **WebSocket/Email** 알림.  
- **AuditLog 중요 이벤트(action prefix)** 에 **Sentry breadcrumb** 또는 Slack 웹훅.

```python
# apps/ops/signals.py (예시)
from django.db.models.signals import post_save
from django.dispatch import receiver
from .models import BulkJob
from asgiref.sync import async_to_sync
from channels.layers import get_channel_layer

@receiver(post_save, sender=BulkJob)
def notify_bulkjob(sender, instance, created, **kwargs):
    if not created and instance.status in ("DONE","FAILED"):
        layer = get_channel_layer()
        async_to_sync(layer.group_send)(
            f"user:{getattr(instance.actor,'pk',0)}",
            {"type": "notify", "data": {"event":"bulkjob", "status":instance.status, "id": instance.id}}
        )
```

---

## 6. 보안/품질 체크리스트

**대시보드**
- [ ] KPI는 **배치 집계 + 인덱스**  
- [ ] 감사 로그는 **행 수 폭증** 대비 파티션/보관정책(예: 90일)  
- [ ] 접근 제어: **RBAC + 오브젝트 레벨** (테넌트)

**Admin 비동기**
- [ ] 액션은 큐에 **태스크 등록만**  
- [ ] 상태 테이블로 진행률/오류 추적  
- [ ] 실패 로그, 재시도, 멱등성(F-Expression/where 조건) 보장

**멀티테넌트**
- [ ] 모든 도메인 모델에 `tenant` FK (예외 최소화)  
- [ ] `get_queryset`/`formfield_for_foreignkey` 에 **스코프 필터**  
- [ ] AdminSite 분리 + `has_permission` 가드  
- [ ] 감사 로그에 `tenant` 포함

---

## 7. 테스트 스니펫

```python
# tests/test_partner_admin_scope.py
import pytest
from django.urls import reverse
pytestmark = pytest.mark.django_db

def test_partner_admin_cannot_see_other_tenant(client, django_user_model, tenant_factory, product_factory):
    t1 = tenant_factory()
    t2 = tenant_factory()
    p1 = product_factory(tenant=t1)
    p2 = product_factory(tenant=t2)
    user = django_user_model.objects.create_user("u","u@x.com","pw", is_staff=True)
    # 테넌트 권한 부여(파트너)
    # TenantUser.objects.create(tenant=t1, user=user, role="admin")
    client.force_login(user)
    # 미들웨어로 tenant=t1 세팅되었다고 가정(또는 세션 지정)
    session = client.session; session["tenant_id"] = t1.id; session.save()
    res = client.get("/partner/shop/product/")
    assert str(p1.id) in res.content.decode()
    assert str(p2.id) not in res.content.decode()
```

---

## 8. 추가 스니펫 모음

### 8-1. Admin 목록 캐싱(프래그먼트 수준)
```python
# heavy list_display 계산에 캐시 사용(시간 한정)
from django.core.cache import cache
class ProductAdmin(admin.ModelAdmin):
    def expensive_col(self, obj):
        key = f"admin:product:exp:{obj.pk}"
        val = cache.get(key)
        if val is None:
            # ... 비싼 계산 ...
            val = obj.reviews.count()
            cache.set(key, val, 60)
        return val
```

### 8-2. 대량 Export 비동기
```python
# 액션 → Celery가 CSV/XLSX 생성 → S3 저장 → 서명 URL 반환
def export_products_async(modeladmin, request, queryset):
    ids = list(queryset.values_list("id", flat=True))
    job = BulkJob.objects.create(name="export_products", actor=request.user, total=len(ids))
    tasks.export_products.delay(job.id, ids)
    messages.info(request, "내보내기를 시작했습니다. 완료되면 알림이 전송됩니다.")
```

---

## 마무리

- **운영 대시보드**는 **사전 집계된 KPI**와 **감사 로그**를 바탕으로 **운영 판단**과 **감사 추적**을 빠르게 합니다.  
- **Admin 비동기화**로 **대량 작업의 타임아웃/잠김**을 제거하고, **상태/알림**으로 가시성을 확보하세요.  
- **멀티테넌트 분리**는 **데이터 경계**의 기본입니다. AdminSite/쿼리 가드/미들웨어로 **테넌트 스코프**를 시스템적으로 보장하여 보안·운영 리스크를 줄이세요.
