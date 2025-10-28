---
layout: post
title: Django - REST 백엔드
date: 2025-10-06 23:30:23 +0900
category: Django
---
# REST 백엔드 — **DRF + JWT**, 문서화(Swagger/Redoc), 배포까지 (Django 5.x 기준)

> 이 문서는 **Django REST Framework(DRF)** 와 **JWT 인증(simplejwt)** 으로 **프로덕션급 REST API** 를 구축하고,  
> **OpenAPI 기반 문서화(스웨거/레닥)** 및 **배포(도커/NGINX/Gunicorn/Uvicorn, CI/CD)** 까지 **끝까지** 다룹니다.  
> 모든 소스 코드는 ``` 로 감싸고, 수학 표기는 없습니다.

---

## 0. 개요 & 목표

- **도메인**: 간단한 “프로젝트 & 작업(ToDo)” API
  - `Project` (소유자/이름/설명)  
  - `Task` (프로젝트/제목/설명/완료여부/마감일)
- **인증/권한**: JWT(Access/Refresh, 회전/블랙리스트), `IsAuthenticated`, `IsOwnerOrReadOnly`
- **기능**: CRUD, 필터/검색/정렬, 페이지네이션, 버저닝, 쓰로틀링, 업로드 예시
- **문서화**: drf-spectacular로 OpenAPI3 → Swagger UI & Redoc
- **운영**: 설정 분리(.env), 로깅/예외 처리, CORS, 배포(Docker, Nginx, Gunicorn/Uvicorn), 헬스체크
- **테스트**: pytest + DRF APIClient, curl/HTTPie 사용법

---

## 1. 초기 세팅

### 1.1 설치

```bash
python -m venv .venv && source .venv/bin/activate
pip install "Django>=5.0" "djangorestframework>=3.15" \
            "djangorestframework-simplejwt>=5.3" \
            "drf-spectacular>=0.27" "drf-spectacular-sidecar>=2024.7.1" \
            "django-filter>=24.2" "django-environ>=0.11" \
            "psycopg2-binary>=2.9" "django-cors-headers>=4.4" \
            "gunicorn>=22.0" "uvicorn[standard]>=0.30"
django-admin startproject config .
python manage.py startapp core      # 공용 유틸
python manage.py startapp projects  # 프로젝트/태스크 도메인
```

### 1.2 settings.py (핵심)

```python
# config/settings.py
from pathlib import Path
import environ, datetime

BASE_DIR = Path(__file__).resolve().parent.parent
env = environ.Env(
    DEBUG=(bool, True),
    SECRET_KEY=(str, "dev-secret"),
    ALLOWED_HOSTS=(list, ["*"]),
    DB_URL=(str, f"sqlite:///{BASE_DIR/'db.sqlite3'}"),
)
environ.Env.read_env(BASE_DIR / ".env")

SECRET_KEY = env("SECRET_KEY")
DEBUG = env("DEBUG")
ALLOWED_HOSTS = env("ALLOWED_HOSTS")

INSTALLED_APPS = [
    "django.contrib.admin","django.contrib.auth","django.contrib.contenttypes",
    "django.contrib.sessions","django.contrib.messages","django.contrib.staticfiles",
    # Third-party
    "rest_framework","django_filters","corsheaders",
    "drf_spectacular","drf_spectacular_sidecar",
    # Local
    "core","projects",
]

MIDDLEWARE = [
    "django.middleware.security.SecurityMiddleware",
    "django.contrib.sessions.middleware.SessionMiddleware",
    "corsheaders.middleware.CorsMiddleware",   # CORS
    "django.middleware.common.CommonMiddleware",
    "django.middleware.csrf.CsrfViewMiddleware", # 세션 기반 사용 시
    "django.contrib.auth.middleware.AuthenticationMiddleware",
    "django.contrib.messages.middleware.MessageMiddleware",
    "django.middleware.clickjacking.XFrameOptionsMiddleware",
]

ROOT_URLCONF = "config.urls"
TEMPLATES = [{
    "BACKEND": "django.template.backends.django.DjangoTemplates",
    "DIRS": [BASE_DIR/"templates"],
    "APP_DIRS": True,
    "OPTIONS": {"context_processors": [
        "django.template.context_processors.debug",
        "django.template.context_processors.request",
        "django.contrib.auth.context_processors.auth",
        "django.contrib.messages.context_processors.messages",
    ]},
}]
WSGI_APPLICATION = "config.wsgi.application"     # Gunicorn
ASGI_APPLICATION = "config.asgi.application"     # Uvicorn/ASGI (선택)

DATABASES = {"default": env.db("DB_URL")}

STATIC_URL = "static/"
STATIC_ROOT = BASE_DIR/"staticfiles"
DEFAULT_AUTO_FIELD = "django.db.models.BigAutoField"

# DRF 기본설정
REST_FRAMEWORK = {
    "DEFAULT_AUTHENTICATION_CLASSES": (
        "rest_framework_simplejwt.authentication.JWTAuthentication",
    ),
    "DEFAULT_PERMISSION_CLASSES": (
        "rest_framework.permissions.IsAuthenticated",
    ),
    "DEFAULT_PAGINATION_CLASS": "rest_framework.pagination.PageNumberPagination",
    "PAGE_SIZE": 20,
    "DEFAULT_FILTER_BACKENDS": (
        "django_filters.rest_framework.DjangoFilterBackend",
        "rest_framework.filters.SearchFilter",
        "rest_framework.filters.OrderingFilter",
    ),
    "DEFAULT_VERSIONING_CLASS": "rest_framework.versioning.NamespaceVersioning",
    "DEFAULT_SCHEMA_CLASS": "drf_spectacular.openapi.AutoSchema",
    "DEFAULT_THROTTLE_CLASSES": [
        "rest_framework.throttling.UserRateThrottle",
        "rest_framework.throttling.AnonRateThrottle",
    ],
    "DEFAULT_THROTTLE_RATES": {
        "user": "1000/day",
        "anon": "100/day",
    },
    "EXCEPTION_HANDLER": "core.exceptions.custom_exception_handler",
}

# SimpleJWT
from datetime import timedelta
SIMPLE_JWT = {
    "ACCESS_TOKEN_LIFETIME": timedelta(minutes=15),
    "REFRESH_TOKEN_LIFETIME": timedelta(days=7),
    "ROTATE_REFRESH_TOKENS": True,
    "BLACKLIST_AFTER_ROTATION": True,
    "UPDATE_LAST_LOGIN": True,
    "ALGORITHM": "HS256",
    "SIGNING_KEY": SECRET_KEY,
    "AUTH_HEADER_TYPES": ("Bearer",),
    "AUTH_TOKEN_CLASSES": ("rest_framework_simplejwt.tokens.AccessToken",),
    "USER_ID_FIELD": "id",
    "USER_ID_CLAIM": "uid",
}

# Spectacular
SPECTACULAR_SETTINGS = {
    "TITLE": "Projects API",
    "DESCRIPTION": "프로젝트/태스크 관리 REST API",
    "VERSION": "1.0.0",
    "SERVE_INCLUDE_SCHEMA": False,
    "SWAGGER_UI_SETTINGS": {"deepLinking": True, "persistAuthorization": True},
    "COMPONENT_SPLIT_REQUEST": True,
}

# CORS
CORS_ALLOW_ALL_ORIGINS = True  # 프로덕션에서 도메인 화이트리스트로 제한 권장

# 보안(배포 시)
SECURE_PROXY_SSL_HEADER = ("HTTP_X_FORWARDED_PROTO", "https")
SESSION_COOKIE_SECURE = not DEBUG
CSRF_COOKIE_SECURE = not DEBUG
```

`.env` 예시:

```bash
DEBUG=False
SECRET_KEY=change-me
ALLOWED_HOSTS=["api.example.com"]
DB_URL=postgres://user:pass@db:5432/app
```

---

## 2. 도메인 모델 & 마이그레이션

```python
# projects/models.py
from django.db import models
from django.conf import settings

class Project(models.Model):
    owner = models.ForeignKey(settings.AUTH_USER_MODEL, on_delete=models.CASCADE, related_name="projects")
    name = models.CharField(max_length=120)
    description = models.TextField(blank=True)
    is_archived = models.BooleanField(default=False)
    created_at = models.DateTimeField(auto_now_add=True)

    class Meta:
        indexes = [models.Index(fields=["owner","-created_at"])]

    def __str__(self): return self.name

class Task(models.Model):
    project = models.ForeignKey(Project, on_delete=models.CASCADE, related_name="tasks")
    title = models.CharField(max_length=180)
    detail = models.TextField(blank=True)
    done = models.BooleanField(default=False)
    due_date = models.DateField(null=True, blank=True)
    created_at = models.DateTimeField(auto_now_add=True)

    class Meta:
        indexes = [models.Index(fields=["project","-created_at"]), models.Index(fields=["done","due_date"])]

    def __str__(self): return self.title
```

마이그레이션:

```bash
python manage.py makemigrations
python manage.py migrate
```

---

## 3. 직렬화기(Serializer)

```python
# projects/serializers.py
from rest_framework import serializers
from .models import Project, Task

class TaskSerializer(serializers.ModelSerializer):
    class Meta:
        model = Task
        fields = ["id","project","title","detail","done","due_date","created_at"]
        read_only_fields = ["id","created_at"]

class ProjectSerializer(serializers.ModelSerializer):
    tasks = TaskSerializer(many=True, read_only=True)
    task_count = serializers.IntegerField(source="tasks.count", read_only=True)

    class Meta:
        model = Project
        fields = ["id","owner","name","description","is_archived","created_at","task_count","tasks"]
        read_only_fields = ["id","owner","created_at","task_count","tasks"]

class ProjectCreateSerializer(serializers.ModelSerializer):
    class Meta:
        model = Project
        fields = ["name","description"]

    def create(self, validated_data):
        user = self.context["request"].user
        return Project.objects.create(owner=user, **validated_data)
```

---

## 4. 권한/퍼미션 & 필터

```python
# projects/permissions.py
from rest_framework.permissions import BasePermission, SAFE_METHODS

class IsOwnerOrReadOnly(BasePermission):
    def has_object_permission(self, request, view, obj):
        if request.method in SAFE_METHODS: return obj.owner_id == request.user.id
        return obj.owner_id == request.user.id
```

```python
# projects/filters.py
import django_filters
from .models import Task

class TaskFilter(django_filters.FilterSet):
    min_due = django_filters.DateFilter(field_name="due_date", lookup_expr="gte")
    max_due = django_filters.DateFilter(field_name="due_date", lookup_expr="lte")
    done = django_filters.BooleanFilter(field_name="done")

    class Meta:
        model = Task
        fields = ["done","min_due","max_due","project"]
```

---

## 5. ViewSet & Router (버저닝 지원)

```python
# projects/views.py
from rest_framework import viewsets, mixins, permissions, filters
from rest_framework.decorators import action
from rest_framework.response import Response
from django.shortcuts import get_object_or_404
from .models import Project, Task
from .serializers import ProjectSerializer, ProjectCreateSerializer, TaskSerializer
from .permissions import IsOwnerOrReadOnly
from .filters import TaskFilter

class ProjectViewSet(viewsets.ModelViewSet):
    queryset = Project.objects.all()
    filter_backends = [filters.SearchFilter, filters.OrderingFilter]
    search_fields = ["name","description"]
    ordering_fields = ["created_at","name"]

    def get_permissions(self):
        if self.action in ("list","retrieve"):
            return [permissions.IsAuthenticated()]
        return [permissions.IsAuthenticated(), IsOwnerOrReadOnly()]

    def get_queryset(self):
        # 오너 소유만 노출(리스트)
        return Project.objects.filter(owner=self.request.user).order_by("-created_at")

    def get_serializer_class(self):
        if self.action in ("create","update","partial_update"):
            return ProjectCreateSerializer
        return ProjectSerializer

    @action(detail=True, methods=["post"], url_path="archive")
    def archive(self, request, pk=None):
        project = self.get_object()
        project.is_archived = True; project.save(update_fields=["is_archived"])
        return Response({"status":"archived"})

class TaskViewSet(viewsets.ModelViewSet):
    serializer_class = TaskSerializer
    filterset_class = TaskFilter
    filter_backends = [filters.SearchFilter, filters.OrderingFilter,]
    search_fields = ["title","detail"]
    ordering_fields = ["created_at","due_date","title"]

    def get_queryset(self):
        # 내 프로젝트의 태스크만
        return Task.objects.filter(project__owner=self.request.user).select_related("project").order_by("-created_at")

    def perform_create(self, serializer):
        project = get_object_or_404(Project, pk=self.request.data.get("project"), owner=self.request.user)
        serializer.save(project=project)
```

---

## 6. JWT 인증: 엔드포인트 & 커스텀 클레임

### 6.1 URLs

```python
# config/urls.py
from django.contrib import admin
from django.urls import path, include
from rest_framework import routers
from projects.views import ProjectViewSet, TaskViewSet
from drf_spectacular.views import SpectacularAPIView, SpectacularSwaggerView, SpectacularRedocView
from rest_framework_simplejwt.views import (
    TokenObtainPairView, TokenRefreshView, TokenVerifyView
)

router_v1 = routers.DefaultRouter()
router_v1.register(r"projects", ProjectViewSet, basename="project")
router_v1.register(r"tasks", TaskViewSet, basename="task")

urlpatterns = [
    path("admin/", admin.site.urls),
    # JWT
    path("api/token/", TokenObtainPairView.as_view(), name="token_obtain_pair"),
    path("api/token/refresh/", TokenRefreshView.as_view(), name="token_refresh"),
    path("api/token/verify/", TokenVerifyView.as_view(), name="token_verify"),

    # API v1
    path("api/v1/", include((router_v1.urls, "v1"), namespace="v1")),

    # OpenAPI schema & Docs
    path("api/schema/", SpectacularAPIView.as_view(), name="schema"),
    path("api/docs/", SpectacularSwaggerView.as_view(url_name="schema"), name="swagger-ui"),
    path("api/redoc/", SpectacularRedocView.as_view(url_name="schema"), name="redoc"),
]
```

### 6.2 커스텀 토큰(추가 클레임)

```python
# core/jwt.py
from rest_framework_simplejwt.serializers import TokenObtainPairSerializer

class MyTokenObtainPairSerializer(TokenObtainPairSerializer):
    @classmethod
    def get_token(cls, user):
        token = super().get_token(user)
        # 커스텀 클레임
        token["username"] = user.username
        token["is_staff"] = user.is_staff
        return token
```

```python
# core/views.py
from rest_framework_simplejwt.views import TokenObtainPairView
from .jwt import MyTokenObtainPairSerializer

class MyTokenObtainPairView(TokenObtainPairView):
    serializer_class = MyTokenObtainPairSerializer
```

```python
# config/urls.py (교체)
from core.views import MyTokenObtainPairView
urlpatterns[1:1] = [
    path("api/token/", MyTokenObtainPairView.as_view(), name="token_obtain_pair"),
]
```

> **리프레시 회전/블랙리스트**는 `SIMPLE_JWT` 설정으로 활성화됨. 블랙리스트 앱을 사용할 경우 `rest_framework_simplejwt.token_blacklist` 를 INSTALLED_APPS에 추가하세요.

---

## 7. 문서화(OpenAPI) — Swagger UI & Redoc

- `drf-spectacular` 가 **APIView/ViewSet/Serializer** 를 스캔해 OpenAPI 스키마를 생성
- `/api/docs/` (Swagger UI), `/api/redoc/` (Redoc) 자동 제공

커스텀 예시(서버/보안 스키마 추가):

```python
# settings.py 내 SPECTACULAR_SETTINGS 확장
SPECTACULAR_SETTINGS.update({
    "SERVERS": [{"url": "https://api.example.com", "description": "Prod"}],
    "SECURITY": [{"bearerAuth": []}],
    "COMPONENTS": {
        "securitySchemes": {
            "bearerAuth": {
                "type": "http", "scheme": "bearer", "bearerFormat": "JWT"
            }
        }
    },
})
```

> 문서에서 “Authorize” 버튼으로 **Bearer 토큰** 을 한 번 입력하면 이후 요청에 헤더 자동 첨부.

---

## 8. 공통 예외 처리 & 표준 응답

```python
# core/exceptions.py
from rest_framework.views import exception_handler
from rest_framework.response import Response
from rest_framework import status as st

def custom_exception_handler(exc, context):
    resp = exception_handler(exc, context)
    if resp is None:
        # 비DRF 예외
        return Response({"detail": "Server error", "error": str(exc)}, status=st.HTTP_500_INTERNAL_SERVER_ERROR)
    # 표준화 래핑
    return Response({
        "status_code": resp.status_code,
        "errors": resp.data,
    }, status=resp.status_code)
```

응답 포맷(예시):

```json
{
  "status_code": 400,
  "errors": {"name":["이 필드는 필수 항목입니다."]}
}
```

---

## 9. 업로드(파일) 예시 (간단)

```python
# projects/models.py (추가)
class Attachment(models.Model):
    project = models.ForeignKey(Project, on_delete=models.CASCADE, related_name="attachments")
    file = models.FileField(upload_to="attachments/%Y/%m/")
    created_at = models.DateTimeField(auto_now_add=True)
```

```python
# projects/serializers.py (추가)
class AttachmentSerializer(serializers.ModelSerializer):
    class Meta:
        model = Attachment
        fields = ["id","project","file","created_at"]
        read_only_fields = ["id","created_at"]
```

```python
# projects/views.py (추가)
from rest_framework.parsers import MultiPartParser, FormParser
from .models import Attachment
from .serializers import AttachmentSerializer

class AttachmentViewSet(viewsets.ModelViewSet):
    queryset = Attachment.objects.all()
    serializer_class = AttachmentSerializer
    parser_classes = [MultiPartParser, FormParser]

    def get_queryset(self):
        return Attachment.objects.filter(project__owner=self.request.user)

    def perform_create(self, serializer):
        project_id = self.request.data.get("project")
        project = get_object_or_404(Project, pk=project_id, owner=self.request.user)
        serializer.save(project=project)
```

```python
# config/urls.py (등록)
router_v1.register(r"attachments", AttachmentViewSet, basename="attachment")
```

> **주의**: 실제 운영에서는 **S3 서명 URL** 방식 권장(서버 부하 감소). 업로드 보안은 MIME/확장자/바이러스 스캔(다음 편 참조).

---

## 10. 쓰로틀/페이지/필터/정렬/검색 예시

- 이미 settings에서 전역 설정함.
- 쿼리 예:
  - `/api/v1/tasks/?done=true&min_due=2025-01-01&ordering=due_date`
  - `/api/v1/projects/?search=backend&ordering=-created_at`
- 페이지네이션: `/api/v1/tasks/?page=2&page_size=10` (커스텀 페이지네이터를 만들어 `PAGE_SIZE_QUERY_PARAM` 도 허용 가능)

---

## 11. 버전 전략

- `NamespaceVersioning` 으로 `api/v1/` 경로 네임스페이스 제공  
- 새로운 호환성 깨짐이 생기면 `router_v2` 로 분리, `path("api/v2/", include(...))` 추가  
- Spectacular는 버전별 스키마 분리도 가능(스키마 뷰를 버전 라우터마다 추가)

---

## 12. CORS/보안 헤더

- `django-cors-headers` 로 SPA/모바일 도메인 허용  
- 배포시:
  - `CORS_ALLOWED_ORIGINS=["https://app.example.com"]`
  - `SECURE_*` 쿠키/헤더, `X-Content-Type-Options: nosniff`, `Content-Security-Policy` (정적/문서 경로에 맞게)  
- JWT는 **기본적으로 Authorization 헤더** 사용. 브라우저 쿠키에 저장 시 `HttpOnly/SameSite=strict` 고려(권장: **메모리/secure storage**에 보관하고 헤더로 전송).

---

## 13. 테스트 (pytest + DRF APIClient)

### 13.1 설치 & 설정

```bash
pip install pytest pytest-django pytest-cov
```

`pytest.ini`:

```ini
[pytest]
DJANGO_SETTINGS_MODULE = config.settings
python_files = tests.py test_*.py *_tests.py
```

### 13.2 토큰 헬퍼 & API 테스트

```python
# projects/tests/test_api.py
import pytest
from django.contrib.auth import get_user_model
from rest_framework.test import APIClient
from django.urls import reverse

pytestmark = pytest.mark.django_db
User = get_user_model()

def auth_client(u: User):
    c = APIClient()
    from rest_framework_simplejwt.tokens import RefreshToken
    t = RefreshToken.for_user(u)
    c.credentials(HTTP_AUTHORIZATION=f"Bearer {str(t.access_token)}")
    return c

def test_project_crud():
    u = User.objects.create_user("u","u@x.com","pw")
    c = auth_client(u)

    # create
    res = c.post("/api/v1/projects/", {"name":"Demo","description":"d"}, format="json")
    assert res.status_code == 201
    pid = res.data["id"]

    # list
    res = c.get("/api/v1/projects/")
    assert res.status_code == 200 and len(res.data["results"]) == 1

    # update
    res = c.patch(f"/api/v1/projects/{pid}/", {"description":"changed"}, format="json")
    assert res.status_code == 200 and res.data["description"] == "changed"

    # create task
    res = c.post("/api/v1/tasks/", {"project": pid, "title":"T1"}, format="json")
    assert res.status_code == 201

    # filter/search/order
    res = c.get("/api/v1/tasks/?search=T1&ordering=-created_at")
    assert res.status_code == 200 and res.data["count"] == 1
```

---

## 14. 사용 예(curl/HTTPie)

```bash
# 회원가입(기본 Django auth 사용 시 DRF로 직접 뷰 작성하거나 admin에서 생성)
# 토큰 발급
http POST :8000/api/token/ username==admin password==admin
# => {"access":"...","refresh":"..."}

# 인증 붙여 프로젝트 생성
http POST :8000/api/v1/projects/ "Authorization:Bearer <ACCESS>" name="REST 백엔드" description="demo"

# 태스크 생성
http POST :8000/api/v1/tasks/ "Authorization:Bearer <ACCESS>" project:=1 title="API 문서 쓰기"

# 목록/검색/정렬
http GET  :8000/api/v1/tasks/?search=문서 "Authorization:Bearer <ACCESS>"
```

---

## 15. 어드민

```python
# projects/admin.py
from django.contrib import admin
from .models import Project, Task, Attachment

@admin.register(Project)
class ProjectAdmin(admin.ModelAdmin):
    list_display = ("id","name","owner","is_archived","created_at")
    search_fields = ("name","description","owner__username")
    list_filter = ("is_archived",)

@admin.register(Task)
class TaskAdmin(admin.ModelAdmin):
    list_display = ("id","project","title","done","due_date","created_at")
    list_filter = ("done","due_date")
    search_fields = ("title","detail","project__name")

@admin.register(Attachment)
class AttachmentAdmin(admin.ModelAdmin):
    list_display = ("id","project","file","created_at")
```

---

## 16. 로깅 & 헬스체크

```python
# settings.py (추가)
LOGGING = {
  "version": 1,
  "disable_existing_loggers": False,
  "formatters": {"json":{"()":"pythonjsonlogger.jsonlogger.JsonFormatter","fmt":"%(levelname)s %(name)s %(message)s"}},
  "handlers": {
    "console": {"class": "logging.StreamHandler", "formatter":"json"},
  },
  "root": {"handlers":["console"], "level":"INFO"},
}
```

헬스체크 뷰:

```python
# core/views.py (추가)
from django.http import JsonResponse
def health(request):
    return JsonResponse({"status":"ok"})
```

```python
# config/urls.py (추가)
from core.views import health
urlpatterns += [path("health/", health)]
```

---

## 17. 배포

### 17.1 Dockerfile (멀티스테이지)

```dockerfile
# Dockerfile
FROM python:3.12-slim AS base
ENV PYTHONDONTWRITEBYTECODE=1 PYTHONUNBUFFERED=1
WORKDIR /app
RUN apt-get update && apt-get install -y build-essential libpq-dev && rm -rf /var/lib/apt/lists/*
COPY pyproject.toml poetry.lock* requirements.txt* ./
# (pip 또는 poetry 중 하나 선택) 여기선 pip:
RUN pip install --no-cache-dir -r requirements.txt

FROM base AS runtime
COPY . .
ENV DJANGO_SETTINGS_MODULE=config.settings
RUN python manage.py collectstatic --noinput
CMD ["gunicorn","config.wsgi:application","-w","3","-b","0.0.0.0:8000","--access-logfile","-","--error-logfile","-"]
```

> **대안**: ASGI 성능 필요 시  
> `CMD ["uvicorn","config.asgi:application","--host","0.0.0.0","--port","8000","--workers","3"]`

### 17.2 docker-compose (Postgres + Nginx)

```yaml
# docker-compose.yml
services:
  db:
    image: postgres:16
    environment:
      POSTGRES_DB: app
      POSTGRES_USER: app
      POSTGRES_PASSWORD: app
    volumes: ["pg:/var/lib/postgresql/data"]
  web:
    build: .
    env_file: .env
    depends_on: [db]
    ports: ["8000:8000"]
  nginx:
    image: nginx:1.27
    depends_on: [web]
    ports: ["80:80"]
    volumes:
      - ./deploy/nginx.conf:/etc/nginx/conf.d/default.conf:ro
      - ./staticfiles:/static:ro
volumes: { pg: {} }
```

`deploy/nginx.conf`:

```nginx
server {
  listen 80;
  server_name _;
  client_max_body_size 20M;

  location /static/ {
    alias /static/;
  }

  location / {
    proxy_pass http://web:8000;
    proxy_set_header Host $host;
    proxy_set_header X-Forwarded-Proto $scheme;
    proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
    proxy_read_timeout 300s;
  }
}
```

> HTTPS는 **로드밸런서/프록시**(예: Nginx + certbot, Cloud LB)에서 종단.  
> HSTS/보안 헤더 설정 포함 권장.

---

## 18. CI/CD (GitHub Actions 예시)

```yaml
# .github/workflows/ci.yml
name: CI
on: [push, pull_request]
jobs:
  test:
    runs-on: ubuntu-latest
    services:
      postgres:
        image: postgres:16
        env:
          POSTGRES_DB: test
          POSTGRES_USER: user
          POSTGRES_PASSWORD: pass
        ports: ["5432:5432"]
        options: >-
          --health-cmd="pg_isready -U user -d test" --health-interval=10s --health-timeout=5s --health-retries=5
    env:
      DB_URL: postgres://user:pass@localhost:5432/test
      SECRET_KEY: ci
      DEBUG: "False"
      ALLOWED_HOSTS: '["*"]'
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-python@v5
        with: { python-version: "3.12" }
      - run: pip install -r requirements.txt
      - run: python manage.py migrate
      - run: pytest -q --maxfail=1 --disable-warnings
```

> 배포 잡은 **Docker build & push** 후 **Infra**(ECS/K8s/VM)에 배포 스텝 추가.

---

## 19. 성능 팁

- **쿼리 최적화**: `select_related`, `prefetch_related`, 인덱스 확인
- **읽기 집중**: Elasticsearch/캐시 읽기 모델(이전 장 참조)
- **Gunicorn**: `--workers` 는 CPU × 2 + 1 권장값에서 트래픽에 맞게 조정
- **Uvicorn/ASGI**: 장기 I/O(웹훅/외부 API) 많은 경우 유리
- **압축 & 캐싱**: Nginx gzip, `Cache-Control` 헤더(문서/정적)
- **레이트리밋/스로틀링**: DRF 전역 설정 외에 per-view 커스터마이즈

---

## 20. 흔한 이슈 & 해결

- **JWT 401**: Authorization 헤더 `Bearer <token>` 확인, 토큰 만료/서명키 불일치
- **CORS 오류**: `corsheaders` 순서/설정 확인(`CorsMiddleware` 는 `CommonMiddleware` 앞)
- **스웨거 빈 스키마**: `DEFAULT_SCHEMA_CLASS` 설정, URL 라우팅/네임스페이스 점검
- **파일 업로드 413**: Nginx `client_max_body_size` 증가
- **DB Deadlock**: 쿼리 순서 일관화, 재시도 로직(데코레이터) 적용

---

## 21. 확장 로드맵

- 소셜 로그인 → JWT 발급(로그인 후 백엔드에서 simplejwt 토큰 발급)
- RBAC/Permissions → `groups`, `permissions` 커스터마이즈 & `DjangoObjectPermissions`
- 이벤트 알림 → Outbox + Webhook → 비동기 작업 큐(Celery)
- 멱등 엔드포인트 → `Idempotency-Key` 헤더 지원
- 멀티테넌시 → `tenant_id` 스코핑/서브도메인/스키마 분리

---

## 22. 마무리

- 본 가이드는 **DRF + SimpleJWT** 로 **표준 REST 백엔드**를 구축하고  
  **OpenAPI(스웨거/레닥)**, **도커 배포** 까지 **프로덕션 관점**에서 다뤘습니다.  
- 핵심 체크:
  - [ ] JWT/권한/쓰로틀/페이지/필터
  - [ ] OpenAPI 문서 & Try-Out
  - [ ] 예외 포맷/로깅/헬스체크
  - [ ] 설정 분리(.env)/보안 헤더/CORS
  - [ ] Docker + Nginx + Gunicorn(Uvicorn)

행운을 빕니다! 🚀

```bash
# 빠른 스모크 테스트
python manage.py createsuperuser
python manage.py runserver
# 1) /api/token/ 로 토큰 발급
# 2) /api/v1/projects/ POST (Bearer 토큰)
# 3) /api/docs/ Swagger에서 Try it out
```