---
layout: post
title: Django - API & 실무 연동
date: 2025-10-01 20:25:23 +0900
category: Django
---
# 4. API & 실무 연동

## A. Django REST Framework — Serializer / ViewSet / Router, 인증·스로틀링, 문서화

### A-0. 설치 & 기본 세팅

```bash
pip install djangorestframework drf-spectacular
```

```python
# config/settings/base.py
INSTALLED_APPS += [
    "rest_framework",
    "drf_spectacular",
]

REST_FRAMEWORK = {
    "DEFAULT_SCHEMA_CLASS": "drf_spectacular.openapi.AutoSchema",
    "DEFAULT_AUTHENTICATION_CLASSES": [
        "rest_framework.authentication.SessionAuthentication",
        "rest_framework.authentication.BasicAuthentication",
        # 토큰/헤더 기반 인증을 쓰려면 추가 (예: JWT)
        # "rest_framework_simplejwt.authentication.JWTAuthentication",
    ],
    "DEFAULT_PERMISSION_CLASSES": [
        "rest_framework.permissions.IsAuthenticatedOrReadOnly",
    ],
    "DEFAULT_THROTTLE_CLASSES": [
        "rest_framework.throttling.UserRateThrottle",
        "rest_framework.throttling.AnonRateThrottle",
    ],
    "DEFAULT_THROTTLE_RATES": {
        "user": "1000/day",
        "anon": "100/day",
    },
    "DEFAULT_PAGINATION_CLASS": "rest_framework.pagination.PageNumberPagination",
    "PAGE_SIZE": 20,
}
```

### A-1. 모델 예시 (공통)

```python
# apps/api/models.py
from django.db import models
from django.conf import settings

class Post(models.Model):
    author = models.ForeignKey(settings.AUTH_USER_MODEL, on_delete=models.CASCADE, related_name="posts")
    title = models.CharField(max_length=150)
    body = models.TextField()
    is_public = models.BooleanField(default=True)
    created_at = models.DateTimeField(auto_now_add=True)

class Comment(models.Model):
    post = models.ForeignKey(Post, on_delete=models.CASCADE, related_name="comments")
    author = models.ForeignKey(settings.AUTH_USER_MODEL, on_delete=models.CASCADE)
    body = models.TextField()
    created_at = models.DateTimeField(auto_now_add=True)
```

### A-2. Serializer — ModelSerializer, 중첩/쓰기전용/읽기전용

```python
# apps/api/serializers.py
from rest_framework import serializers
from .models import Post, Comment

class CommentSerializer(serializers.ModelSerializer):
    author_username = serializers.CharField(source="author.username", read_only=True)

    class Meta:
        model = Comment
        fields = ["id", "author", "author_username", "body", "created_at"]
        read_only_fields = ["id", "author_username", "created_at"]

class PostSerializer(serializers.ModelSerializer):
    author_username = serializers.CharField(source="author.username", read_only=True)
    # 중첩 읽기 전용
    comments = CommentSerializer(many=True, read_only=True)
    # 쓰기 전용 필드 예: 생성 시에만 허용하고 응답에는 안 보이게 하려면 write_only=True
    secret_note = serializers.CharField(write_only=True, required=False)

    class Meta:
        model = Post
        fields = [
            "id", "author", "author_username",
            "title", "body", "is_public",
            "created_at", "comments", "secret_note"
        ]
        read_only_fields = ["id", "author_username", "created_at", "comments"]

    def validate_title(self, v):
        if len(v.strip()) < 3:
            raise serializers.ValidationError("제목은 3자 이상이어야 합니다.")
        return v

    def create(self, validated_data):
        validated_data.pop("secret_note", None)  # 예: 내부 로깅 등에만 사용
        return super().create(validated_data)
```

> 팁
> - **읽기/쓰기 분리**가 필요한 복잡 모델은 **ReadSerializer / WriteSerializer** 를 나눠서 사용하거나, View 수준에서 `get_serializer_class()` 로 동적으로 스위칭합니다.
> - 대량 관계 입력은 **PrimaryKeyRelatedField(many=True)** 나 **SlugRelatedField** 로 처리(중첩 쓰기는 명시적 구현 필요).

### A-3. ViewSet & Router — CRUD/권한/쿼리최적화

```python
# apps/api/views.py
from rest_framework import viewsets, permissions
from rest_framework.decorators import action
from rest_framework.response import Response
from django.db.models import Prefetch
from .models import Post, Comment
from .serializers import PostSerializer, CommentSerializer

class IsAuthorOrReadOnly(permissions.BasePermission):
    def has_object_permission(self, request, view, obj):
        if request.method in ("GET", "HEAD", "OPTIONS"):
            return True
        return getattr(obj, "author_id", None) == getattr(request.user, "id", None)

class PostViewSet(viewsets.ModelViewSet):
    queryset = Post.objects.select_related("author").prefetch_related(
        Prefetch("comments", queryset=Comment.objects.select_related("author").order_by("-id"))
    )
    serializer_class = PostSerializer
    permission_classes = [IsAuthorOrReadOnly]

    def perform_create(self, serializer):
        serializer.save(author=self.request.user)

    @action(detail=True, methods=["get"])
    def comments(self, request, pk=None):
        post = self.get_object()
        ser = CommentSerializer(post.comments.all(), many=True)
        return Response(ser.data)

class CommentViewSet(viewsets.ModelViewSet):
    queryset = Comment.objects.select_related("post", "author").all()
    serializer_class = CommentSerializer
    permission_classes = [permissions.IsAuthenticatedOrReadOnly]

    def perform_create(self, serializer):
        serializer.save(author=self.request.user)
```

```python
# apps/api/urls.py
from rest_framework.routers import DefaultRouter
from .views import PostViewSet, CommentViewSet

router = DefaultRouter()
router.register("posts", PostViewSet, basename="post")
router.register("comments", CommentViewSet, basename="comment")
urlpatterns = router.urls
```

### A-4. 인증 — 세션/Basic/JWT(API에선 보통 JWT)

```bash
pip install djangorestframework-simplejwt
```

```python
# config/settings/base.py
REST_FRAMEWORK["DEFAULT_AUTHENTICATION_CLASSES"] = [
    "rest_framework_simplejwt.authentication.JWTAuthentication",
]
```

```python
# config/urls.py
from django.urls import path, include
from rest_framework_simplejwt.views import TokenObtainPairView, TokenRefreshView

urlpatterns = [
    path("api/", include("apps.api.urls")),
    path("api/token/", TokenObtainPairView.as_view(), name="token_obtain_pair"),
    path("api/token/refresh/", TokenRefreshView.as_view(), name="token_refresh"),
]
```

> 운영 팁
> - **JWT TTL/리프레시 주기**를 비즈니스/보안 레벨에 맞게 조정.
> - 모바일/SPA는 **Refresh Token Rotation** & **로그아웃/블랙리스트** 전략 포함.
> - 서버 사이드 렌더링/HTMX는 세션 인증 + CSRF가 자연스럽습니다.

### A-5. 스로틀링 — 전역/뷰별/사용자·익명 분리

```python
# 전역 설정 외, 뷰에서 개별 적용
from rest_framework.throttling import UserRateThrottle, AnonRateThrottle

class BurstUserThrottle(UserRateThrottle):
    scope = "burst_user"
class BurstAnonThrottle(AnonRateThrottle):
    scope = "burst_anon"

# settings.py
REST_FRAMEWORK["DEFAULT_THROTTLE_RATES"].update({
    "burst_user": "60/min",
    "burst_anon": "30/min",
})
```

```python
class PostViewSet(viewsets.ModelViewSet):
    throttle_classes = [BurstUserThrottle, BurstAnonThrottle]
    ...
```

> 외부 연동/결제 콜백 엔드포인트엔 **IP allowlist** 또는 **서명 검증**을 추가하고, 스로틀링을 엄격히 하세요.

### A-6. 필터/검색/정렬

```bash
pip install django-filter
```

```python
# settings
INSTALLED_APPS += ["django_filters"]
REST_FRAMEWORK["DEFAULT_FILTER_BACKENDS"] = [
    "django_filters.rest_framework.DjangoFilterBackend",
    "rest_framework.filters.SearchFilter",
    "rest_framework.filters.OrderingFilter",
]
```

```python
# views
class PostViewSet(viewsets.ModelViewSet):
    ...
    filterset_fields = ["author", "is_public"]
    search_fields = ["title", "body"]
    ordering_fields = ["created_at", "title"]
    ordering = ["-created_at"]
```

### A-7. 문서화 — drf-spectacular (OpenAPI 3)

```python
# config/urls.py
from drf_spectacular.views import SpectacularAPIView, SpectacularSwaggerView

urlpatterns += [
    path("api/schema/", SpectacularAPIView.as_view(), name="schema"),
    path("api/docs/", SpectacularSwaggerView.as_view(url_name="schema"), name="docs"),
]
```

`serializer`, `view` 에 docstring/`extend_schema` 를 달면 스웨거/레드독 도큐먼트가 정교해집니다.

```python
from drf_spectacular.utils import extend_schema, OpenApiParameter

@extend_schema(
  summary="게시글 목록",
  parameters=[OpenApiParameter("search", str, OpenApiParameter.QUERY)],
)
class PostViewSet(...):
    pass
```

### A-8. 성능/보안 체크리스트

- [ ] **select_related / prefetch_related** 로 N+1 방지
- [ ] **페이지네이션** 기본값 설정(무제한 금지)
- [ ] **Throttle**/RateLimit, **CORS** 설정(외부 접근 범위 제한)
- [ ] **에러 응답 스펙** 통일(코드/메시지/필드 오류 포맷)
- [ ] **Schema 문서** 자동 생성/배포 파이프라인 포함
- [ ] 민감 데이터 마스킹/감사 로그/리퀘스트 ID 로깅

---

## B. GraphQL — Strawberry / Graphene 비교, 스키마/리졸버, N+1 방지

### B-0. 언제 GraphQL?

- **프론트엔드가 데이터 shape를 주도**(모바일/SPA/다수 화면)
- **여러 리소스 조합**을 한 번에 질의, Over/Under-fetch 감소
- 강력한 **스키마 타입**과 **툴링(GraphiQL, codegen)**

### B-1. 라이브러리 선택 — Strawberry vs Graphene

| 항목 | Strawberry | Graphene |
|---|---|---|
| 스타일 | **Python 타입힌트** 기반, 데코레이터 | 클래스/메타 기반 |
| 학습곡선 | 직관적(타입힌트 친화) | 모델-스키마 매핑 풍부 |
| Django 통합 | `strawberry-django` 패키지 | `graphene-django` 패키지 |
| DataLoader | 공식 가이드/예시 많음 | community 예제 풍부 |
| 생태/문서 | 활발, 근래 인기 | 역사 길고 사용자층 넓음 |

둘 다 프로덕션 가능. 팀의 선호/기존 코드 스타일에 맞춰 선택하세요.

### B-2. Strawberry 기본 예제

```bash
pip install strawberry-graphql strawberry-graphql-django
```

```python
# apps/gql/schema.py
import strawberry
from typing import List
from strawberry.types import Info
from strawberry import auto
from strawberry_django import type, field
from apps.api.models import Post, Comment

@type(Post)
class PostType:
    id: auto
    title: auto
    body: auto
    is_public: auto
    author_id: auto

@type(Comment)
class CommentType:
    id: auto
    body: auto
    author_id: auto
    post_id: auto

@strawberry.type
class Query:
    @strawberry.field
    def post(self, info: Info, id: int) -> PostType | None:
        return Post.objects.select_related("author").filter(pk=id).first()

    # 필터/페이지네이션은 strawberry-django의 필터셋/relay 기능 활용 권장
    @strawberry.field
    def posts(self, info: Info) -> List[PostType]:
        return Post.objects.filter(is_public=True).select_related("author").order_by("-id")[:100]

@strawberry.type
class Mutation:
    @strawberry.mutation
    def create_post(self, info: Info, title: str, body: str) -> PostType:
        user = info.context.request.user
        p = Post.objects.create(author=user, title=title, body=body)
        return p

schema = strawberry.Schema(query=Query, mutation=Mutation)
```

```python
# config/urls.py
from django.urls import path
from strawberry.django.views import GraphQLView
from apps.gql.schema import schema

urlpatterns += [
    path("graphql", GraphQLView.as_view(schema=schema)),
]
```

#### 요청 예시 (GraphiQL)
```graphql
query {
  posts {
    id
    title
    authorId
  }
}
```

### B-3. Graphene 기본 예제

```bash
pip install graphene-django
```

```python
# apps/gql/schema_graphene.py
import graphene
from graphene_django import DjangoObjectType
from apps.api.models import Post

class PostType(DjangoObjectType):
    class Meta:
        model = Post
        fields = ("id", "title", "body", "author_id", "is_public")

class Query(graphene.ObjectType):
    post = graphene.Field(PostType, id=graphene.Int(required=True))
    posts = graphene.List(PostType)

    def resolve_post(root, info, id):
        return Post.objects.select_related("author").filter(pk=id).first()

    def resolve_posts(root, info):
        return Post.objects.filter(is_public=True).select_related("author").order_by("-id")[:100]

class CreatePost(graphene.Mutation):
    class Arguments:
        title = graphene.String(required=True)
        body = graphene.String(required=True)
    post = graphene.Field(PostType)

    @classmethod
    def mutate(cls, root, info, title, body):
        user = info.context.user
        p = Post.objects.create(author=user, title=title, body=body)
        return CreatePost(post=p)

class Mutation(graphene.ObjectType):
    create_post = CreatePost.Field()

schema = graphene.Schema(query=Query, mutation=Mutation)
```

### B-4. 인증/권한 & 컨텍스트

```python
# GraphQLView에 인증이 붙은 Django 뷰를 사용하면 info.context.request.user 사용 가능
# Strawberry
GraphQLView.as_view(schema=schema, graphiql=True)
# 커스텀: @login_required 데코레이터와 혼합하거나, resolver 내부에서 user 검사
```

```python
# Resolver에서 권한 검사 예시
def resolve_posts(root, info):
    user = info.context.user
    qs = Post.objects.filter(is_public=True)
    if user.is_authenticated:
        qs = Post.objects.filter(author=user) | qs
    return qs.select_related("author").distinct()
```

### B-5. N+1 방지 — DataLoader

GraphQL은 필드 단위로 리졸버가 반복 호출되어 **N+1** 이 빈번합니다. **DataLoader** 로 배치 로딩하세요.

```bash
pip install aiodataloader
```

```python
# apps/gql/dataloaders.py
from aiodataloader import DataLoader
from apps.api.models import Post

class PostLoader(DataLoader):
    async def batch_load_fn(self, keys):
        # keys: [id, id, ...]
        qs = Post.objects.filter(id__in=keys)
        mapping = {p.id: p for p in qs}
        return [mapping.get(k) for k in keys]
```

```python
# apps/gql/context.py
from .dataloaders import PostLoader

def get_context(request):
    return {"request": request, "loaders": {"post": PostLoader()}}
```

```python
# urls.py (Strawberry)
GraphQLView.as_view(schema=schema, context_getter=get_context)
```

```python
# schema.py (Strawberry)
@strawberry.field
async def post(self, info: Info, id: int) -> PostType | None:
    loader = info.context["loaders"]["post"]
    return await loader.load(id)
```

> **select_related/prefetch_related** + **DataLoader** 를 조합하면 대부분의 N+1을 해결할 수 있습니다.

### B-6. 파일 업로드, 서브스크립션

- 파일 업로드: **GraphQL multipart** 프로토콜 지원(뷰/클라이언트 설정 필요)
- 실시간 구독: WebSocket 기반. `strawberry` 는 ASGI로 **subscription** 지원(별도 서버/라우팅 고려)

### B-7. 스키마/버저닝/모니터링

- GraphQL은 **단일 엔드포인트** → 스키마 변경은 **deprecation** 정책과 릴리스 노트로 관리
- 스키마 문서/플레이그라운드(GraphiQL)를 **보안 경계 안**으로(운영 공개 금지 권장)
- **쿼리 복잡도 제한**(depth/complexity)과 **쿼리 화이트리스트** 도입 검토

---

## C. 웹훅/외부 연동 — 결제(아임포트/Stripe), 소셜로그인, 이메일(SMTP/Transactional)

### C-0. 공통 원칙

- **서명(signature) 검증** 혹은 **허용 IP** 확인
- **Idempotency(멱등성)**: 같은 이벤트 재시도에 안전하게
- **감사 로그**: 원문 payload 저장(민감 정보 마스킹)
- **타임아웃/재시도** 대비
- **테스트 모드 분리**(Sandbox 키/엔드포인트)

---

### C-1. 결제 — Stripe 예제 (웹훅 + 결제 확인)

```bash
pip install stripe
```

```python
# config/settings/base.py
STRIPE_WEBHOOK_SECRET = env("STRIPE_WEBHOOK_SECRET", default="")
STRIPE_API_KEY = env("STRIPE_API_KEY", default="")
```

```python
# apps/payments/webhooks.py
import stripe
from django.conf import settings
from django.http import HttpResponse, HttpResponseBadRequest
from django.views.decorators.csrf import csrf_exempt
from django.utils.encoding import force_str
from .models import Order  # 예시

@csrf_exempt
def stripe_webhook(request):
    payload = request.body
    sig = request.headers.get("Stripe-Signature", "")
    try:
        event = stripe.Webhook.construct_event(
            payload=payload, sig_header=sig, secret=settings.STRIPE_WEBHOOK_SECRET
        )
    except (ValueError, stripe.error.SignatureVerificationError):
        return HttpResponseBadRequest("Invalid payload or signature")

    if event["type"] == "payment_intent.succeeded":
        intent = event["data"]["object"]
        idempotency_key = intent["id"]
        order_id = intent["metadata"].get("order_id")
        # 멱등 처리
        from django.db import transaction
        from django.db.models import F

        with transaction.atomic():
            order = Order.objects.select_for_update().get(pk=order_id)
            if order.paid:
                return HttpResponse("already done")
            order.paid = True
            order.payment_ref = idempotency_key
            order.save(update_fields=["paid", "payment_ref"])
    return HttpResponse("ok")
```

> Stripe는 **서명 헤더 검증**을 제공; 반드시 사용하세요. 또한, **idempotency-key** (요청 측)와 별개로 **웹훅 이벤트 id**를 사용한 **멱등 처리**가 필요합니다.

#### 아임포트(IMP) 개념 흐름
- 결제 완료 콜백에서 **imp_uid** 수신 → **REST API로 영수증 검증**(금액/상태 일치 확인) → 주문 확정
- **서버 사이드 검증** 필수, 프런트만 믿지 마세요

```python
# (간략) 아임포트 검증 스케치
import requests
IMP_HOST = "https://api.iamport.kr"

def get_access_token(key, secret):
    r = requests.post(f"{IMP_HOST}/users/getToken", json={"imp_key": key, "imp_secret": secret})
    r.raise_for_status()
    return r.json()["response"]["access_token"]

def verify_payment(access_token, imp_uid):
    r = requests.get(f"{IMP_HOST}/payments/{imp_uid}", headers={"Authorization": access_token})
    r.raise_for_status()
    return r.json()["response"]  # amount/status 등
```

> 운영 시 **서명처럼 위조 방지 수단**은 제공되지 않으므로, **백엔드에서 직접 검증 API 호출**로 정합성 확보가 핵심입니다.

---

### C-2. 소셜 로그인 — django-allauth (OAuth/OIDC)

```bash
pip install django-allauth
```

```python
# settings.py
INSTALLED_APPS += [
    "django.contrib.sites",
    "allauth", "allauth.account", "allauth.socialaccount",
    # 공급자
    "allauth.socialaccount.providers.google",
    "allauth.socialaccount.providers.github",
]
SITE_ID = 1
AUTHENTICATION_BACKENDS = [
    "django.contrib.auth.backends.ModelBackend",
    "allauth.account.auth_backends.AuthenticationBackend",
]
LOGIN_REDIRECT_URL = "/"
ACCOUNT_EMAIL_VERIFICATION = "optional"  # "mandatory" 권장
```

```python
# urls.py
from django.urls import path, include
urlpatterns += [
    path("accounts/", include("allauth.urls")),  # /accounts/login/ 등
]
```

- 각 공급자 콘솔에서 **Redirect URL** 등록: 예) `https://example.com/accounts/google/login/callback/`
- OIDC(기업 SSO) 환경에서는 **scopes/claim 매핑**, **이메일 도메인 제한**, **가입 승인 플로우**를 구현

> DRF/SPA 환경에서 소셜 로그인 → **백엔드에서 OAuth code 교환** 후 **백엔드가 발급한 토큰(JWT)** 을 프론트가 저장하는 패턴이 일반적입니다.

---

### C-3. 이메일 — SMTP / Transactional(발송 로그/템플릿)

#### SMTP(기본)
```python
# settings.py
EMAIL_BACKEND = "django.core.mail.backends.smtp.EmailBackend"
EMAIL_HOST = "smtp.gmail.com"
EMAIL_PORT = 587
EMAIL_USE_TLS = True
EMAIL_HOST_USER = env("EMAIL_USER")
EMAIL_HOST_PASSWORD = env("EMAIL_PASS")
DEFAULT_FROM_EMAIL = "MyApp <noreply@example.com>"
```

```python
# apps/notifications/mailers.py
from django.core.mail import send_mail
def send_welcome(to, name):
    subject = "환영합니다"
    message = f"{name}님, 가입을 환영합니다."
    send_mail(subject, message, None, [to], fail_silently=False)
```

#### 트랜잭셔널 서비스(예: SendGrid, Mailgun, Amazon SES)
- 장점: **발송 로그/반송/스팸/템플릿 관리**, **웹훅** 으로 **오픈/클릭/반송 이벤트** 처리 가능
- 단점: 서비스 의존 + 비용

```python
# 예: django-anymail + SendGrid
pip install django-anymail
```

```python
# settings.py
INSTALLED_APPS += ["anymail"]
EMAIL_BACKEND = "anymail.backends.sendgrid.EmailBackend"
ANYMAIL = {"SENDGRID_API_KEY": env("SENDGRID_API_KEY")}
```

```python
# templates 이메일 템플릿 + context 렌더
from django.template.loader import render_to_string
from anymail.message import AnymailMessage

def send_order_receipt(to, order):
    html = render_to_string("emails/receipt.html", {"order": order})
    msg = AnymailMessage(
        subject=f"주문 확인 #{order.pk}",
        body="HTML을 지원하지 않는 클라이언트를 위한 텍스트 본문",
        from_email="MyApp <noreply@example.com>",
        to=[to],
    )
    msg.attach_alternative(html, "text/html")
    msg.send()
```

**수신 웹훅 처리(반송/스팸 신고)**
- Anymail은 통합 뷰를 제공하지만, 직접 구현 시 이벤트의 **서명 검증**(서비스 제공) 사용

---

### C-4. 웹훅 보안/운영 체크리스트

- [ ] **서명 검증 / IP allowlist / mTLS 중 하나 이상**
- [ ] **Idempotency**: 이벤트 id 저장 후 **중복 수신 무시**
- [ ] **타임아웃/재시도**: 외부가 재시도할 수 있으니 **빠른 2xx** + 백그라운드 처리
- [ ] **감사 로그**: 원문 payload (민감정보 마스킹) + 처리 결과
- [ ] **장애 대응**: DLQ(실패 큐), 재처리 커맨드 관리
- [ ] **테스트 모드 분리**: 키/URL/DB 레코드 명시 구분

---

## D. 통합 실전 예제 — “주문 API + 결제 확정 웹훅 + 이메일 영수증 + 문서화”

### D-1. 주문 생성(REST)

```python
# apps/orders/serializers.py
from rest_framework import serializers
from .models import Order, OrderItem

class OrderItemWriteSerializer(serializers.ModelSerializer):
    class Meta:
        model = OrderItem
        fields = ["product_id", "quantity"]

class OrderWriteSerializer(serializers.ModelSerializer):
    items = OrderItemWriteSerializer(many=True)

    class Meta:
        model = Order
        fields = ["id", "email", "items"]
        read_only_fields = ["id"]

    def create(self, validated_data):
        items = validated_data.pop("items")
        order = Order.objects.create(**validated_data)
        # 가격 스냅샷/재고 확인은 여기 또는 서비스 계층에서
        bulk = [OrderItem(order=order, **it) for it in items]
        OrderItem.objects.bulk_create(bulk)
        return order
```

```python
# apps/orders/views.py
from rest_framework import viewsets, permissions
from .serializers import OrderWriteSerializer
from .models import Order

class OrderViewSet(viewsets.ModelViewSet):
    queryset = Order.objects.all()
    serializer_class = OrderWriteSerializer
    permission_classes = [permissions.AllowAny]  # 게스트 주문 허용 예시
```

```python
# apps/orders/urls.py
from rest_framework.routers import DefaultRouter
from .views import OrderViewSet

router = DefaultRouter()
router.register("orders", OrderViewSet, basename="order")
urlpatterns = router.urls
```

### D-2. 결제 진행(프런트) → Stripe PaymentIntent 생성(API) → 결제 완료 웹훅에서 확정

```python
# apps/payments/api.py
import stripe
from django.conf import settings
from rest_framework.decorators import api_view, permission_classes
from rest_framework.permissions import AllowAny
from rest_framework.response import Response
from apps.orders.models import Order

stripe.api_key = settings.STRIPE_API_KEY

@api_view(["POST"])
@permission_classes([AllowAny])
def create_payment_intent(request):
    order_id = request.data.get("order_id")
    order = Order.objects.get(pk=order_id)
    amount = order.total_amount_cents()  # 예: 센트 단위
    intent = stripe.PaymentIntent.create(
        amount=amount,
        currency="usd",
        metadata={"order_id": str(order.id)},
    )
    return Response({"client_secret": intent.client_secret})
```

프런트는 `client_secret` 으로 결제 완료 → Stripe가 **웹훅** 전송 → 앞서 구현한 `stripe_webhook` 이 주문 확정 + **이메일 영수증 발송** 트리거(비동기 권장).

### D-3. 이메일 영수증(비동기 권장: Celery)

```python
# apps/orders/tasks.py
from .models import Order
from apps.notifications.mailers import send_order_receipt

def send_receipt(order_id):
    order = Order.objects.get(pk=order_id)
    send_order_receipt(order.email, order)
```

```python
# apps/payments/webhooks.py (확정 후)
from apps.orders.tasks import send_receipt
...
if event["type"] == "payment_intent.succeeded":
    ...
    send_receipt(order.pk)  # Celery면 .delay(order.pk)
```

### D-4. OpenAPI 문서(스니펫)

```python
# config/urls.py
urlpatterns += [
    path("api/schema/", SpectacularAPIView.as_view(), name="schema"),
    path("api/docs/", SpectacularSwaggerView.as_view(url_name="schema"), name="docs"),
]
```

> 운영에서는 **테스트/스테이징** 환경에만 공개하거나, 접근 제어(로그인·IP) 후 노출하세요.

---

## E. 운영·보안·품질 체크리스트 (요약)

**DRF**
- [ ] 인증 체계 선택(JWT/세션) & 토큰 만료/폐기 전략
- [ ] 스로틀링, 페이징, 필터/검색/정렬 가드
- [ ] N+1 제거, 직렬화 비용 프로파일링
- [ ] OpenAPI 자동 문서 + 예제 payload

**GraphQL**
- [ ] DataLoader/프리페치로 N+1 방지
- [ ] 쿼리 복잡도/깊이 제한, 페이징/커서 표준화
- [ ] 스키마 변경 시 Deprecation/릴리스 노트
- [ ] 파일 업로드/구독 보안 검토

**웹훅 & 외부**
- [ ] 서명/허용 IP, TLS, 멱등성
- [ ] 콜백은 **빠른 2xx**, 실처리는 비동기
- [ ] 감사 로그 + 재처리 도구
- [ ] 결제는 항상 **서버측 검증**(금액/상태)

**이메일**
- [ ] SPF/DKIM/DMARC 정합
- [ ] 반송/스팸 이벤트 수신 → 구독 관리/평판 유지
- [ ] 템플릿/다국어/접근성 점검

---

## F. 추가 스니펫 모음

### F-1. DRF 예외 핸들러 통일
```python
# apps/api/exceptions.py
from rest_framework.views import exception_handler

def unified_exception_handler(exc, context):
    resp = exception_handler(exc, context)
    if resp is not None:
        resp.data = {
            "code": resp.status_code,
            "message": resp.data.get("detail", "error"),
            "errors": resp.data,
        }
    return resp

# settings.py
REST_FRAMEWORK["EXCEPTION_HANDLER"] = "apps.api.exceptions.unified_exception_handler"
```

### F-2. DRF 버저닝
```python
REST_FRAMEWORK["DEFAULT_VERSIONING_CLASS"] = "rest_framework.versioning.NamespaceVersioning"
# urls.py 에서 /api/v1/, /api/v2/ 네임스페이스로 분리
```

### F-3. GraphQL 쿼리 제한(예시: depth)
```python
# 미들웨어로 depth 검사(의사코드)
class MaxDepthMiddleware:
    def resolve(self, next_, root, info, **args):
        depth = calc_depth(info)  # 구현 필요
        if depth > 8:
            raise Exception("Query too deep")
        return next_(root, info, **args)
```

### F-4. 웹훅 재처리 커맨드
```python
# apps/payments/management/commands/replay_webhook.py
from django.core.management.base import BaseCommand
from apps.payments.models import WebhookLog

class Command(BaseCommand):
    def add_arguments(self, parser):
        parser.add_argument("event_id")
    def handle(self, *args, **opts):
        log = WebhookLog.objects.get(event_id=opts["event_id"])
        # 저장된 payload로 재처리 함수 호출
        process_webhook(log.payload)
```

### F-5. CORS 설정
```bash
pip install django-cors-headers
```

```python
INSTALLED_APPS += ["corsheaders"]
MIDDLEWARE = ["corsheaders.middleware.CorsMiddleware"] + MIDDLEWARE
CORS_ALLOWED_ORIGINS = ["https://app.example.com"]
CORS_ALLOW_CREDENTIALS = True
```

---

## 마무리

- **DRF** 로 RESTful API를 빠르고 안전하게 구축하고, **문서/스로틀/권한/N+1** 을 시스템 차원에서 관리하세요.
- **GraphQL** 은 DataLoader·프리페치로 N+1을 제압하고, 스키마/복잡도/버저닝을 운영 정책으로 다루세요.
- **외부 연동(결제·소셜·이메일)** 은 **서명/멱등성/검증/감사 로그** 를 기준선으로 삼고, **콜백은 빠르게 2xx** + 비동기 처리를 습관화하세요.
