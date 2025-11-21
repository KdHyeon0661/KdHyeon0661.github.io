---
layout: post
title: Django - Django란
date: 2025-10-01 14:25:23 +0900
category: Django
---
# 시작하기 — Django란? 환경 설정, 첫 프로젝트/앱 생성 (Django 5.x 기준)

> 이 글은 **Django 초심자~중급자**가 **실전 프로젝트**를 바로 시작할 수 있도록,
> “무엇을 왜 그렇게 하는지”까지 **맥락 + 예제 + 체크리스트**로 빠짐없이 정리한다.
> 모든 코드는 **```**로 감싼다. 수학식이 나오면 **$$**로 감싼다(본 글은 수학식이 없다).

---

## 0-1. Django란? — 역사·철학(MTV)·Django 5.x 관점 핵심

### Django의 탄생 배경과 철학

- **신문사 콘텐츠 CMS**에서 출발(빠른 개발 속도에 초점)
- **“배터리 포함(batteries-included)”**: 인증, 관리자, ORM, 국제화, 보안 프레임워크 등 **웹 서비스 필수 기능**을 광범위하게 기본 제공
- **일관된 설계 철학**: 단순함, 명시성, DRY(Don’t Repeat Yourself), 보안 우선, 확장성
- **장점**
  - 공통 과제를 해결하는 **표준 부품**과 **Best Practice** 내장
  - 강력한 **ORM**으로 RDB 연동·마이그레이션 수월
  - **Admin**으로 초기 운영 도구 즉시 확보
  - **보안 기본값**(CSRF, XSS, Clickjacking 방어 옵션 등)
  - **테스트/국제화/배포**까지 **엔드투엔드 관점** 지원
- **유스케이스**
  - 기업 정보시스템(내부·B2B)
  - 커뮤니티/커머스/콘텐츠 플랫폼
  - 백오피스/관리도구·데이터 포털
  - API 백엔드(DRF 추가 시)

### MTV 아키텍처(= Django의 MVC 해석)

- **M(Model)**: 데이터/도메인 규칙 → Django ORM의 `models.Model`
- **T(Template)**: 프론트 표현 → DTL(Django Template Language)
- **V(View)**: 요청 처리·비즈니스 규칙 → 함수형 뷰/클래스형 뷰(CBV)
- **URLconf**가 **컨트롤러 역할** 일부를 담당(라우팅을 통해 View 호출)

**요청 흐름(고수준)**
1. 브라우저 → **URLconf** 매칭
2. **View** 진입(권한·폼 검증·서비스 로직·ORM 호출)
3. **Template** 렌더링(HTML) 또는 **JSON 응답** 반환
4. **Middleware**는 요청/응답의 전후 훅에서 횡단 관심사 처리(보안 헤더, 로깅 등)

### Django 5.x 관점(핵심 포인트 위주)

- **ASGI(비동기) 서포트 성숙**: `async def` 뷰, 일부 ORM I/O 최적화 지향(단, ORM은 전면 async 완성 전환이 아닌, 제한적/선택적 활용이 일반적이었음)
- **Form/템플릿/유틸리티** 등 점진 개선, 오래된 API 정리·폐기 가속(장기 지원 정책에 따라 주기적 디프리케이션)
- **보안 디폴트 강화** 추세(보안 헤더/쿠키 속성/CSRF 개선 등)
> 주: 5.x의 **세부 변경점 버전별 리스트**는 공식 문서 릴리스 노트를 확인. 실무 관점에서는 **비동기 대응, 보안 기본값, 테스트/타이핑/정적분석 친화성**을 체크리스트에 반영하면 된다.

---

## 0-2. 개발 환경 설정 — Python/venv/Poetry, VS Code, 프로젝트 구조

### 파이썬 설치 & 가상 환경(venv)

- **권장 버전**: Django 5.x 라인과 호환되는 최신 안정 파이썬(3.10+ 권장)
- **venv 생성/활성화**
```bash
# Windows (PowerShell)

py -3.11 -m venv .venv
.venv\Scripts\Activate.ps1

# macOS/Linux (bash/zsh)

python3.11 -m venv .venv
source .venv/bin/activate
```
- **업그레이드/기본 패키지**
```bash
python -m pip install --upgrade pip wheel setuptools
```

### Poetry로 의존성/빌드 관리(선택)

- **Poetry 설치**
```bash
# 권장: pipx 사용

pipx install poetry
poetry --version
```
- **프로젝트 초기화**
```bash
poetry init  # 질의응답으로 pyproject.toml 생성
poetry add django python-dotenv django-environ
# 개발 전용

poetry add -D black isort ruff pytest pytest-django
```
- **Poetry 가상환경 진입**
```bash
poetry shell
```
- **pyproject.toml 샘플**
```toml
[tool.poetry]
name = "my-django-app"
version = "0.1.0"
description = "Example Django project"
authors = ["You <you@example.com>"]
packages = [{ include = "src" }]

[tool.poetry.dependencies]
python = "^3.11"
django = "^5.0"
django-environ = "^0.11.2"
python-dotenv = "^1.0.1"

[tool.poetry.group.dev.dependencies]
black = "^24.3.0"
isort = "^5.13.2"
ruff = "^0.5.0"
pytest = "^8.0.0"
pytest-django = "^4.8.0"

[build-system]
requires = ["poetry-core"]
build-backend = "poetry.core.masonry.api"

# 선택: 포맷터/린터 설정

[tool.black]
line-length = 100
target-version = ["py311"]

[tool.isort]
profile = "black"

[tool.ruff]
line-length = 100
select = ["E", "F", "I"]
```

### VS Code 설정 포인트

- **익스텐션**: Python, Pylance, Django Template, GitLens, dotenv
- **Interpreter 선택**: `Ctrl/Cmd + Shift + P` → *Python: Select Interpreter* → `.venv`
- **디버그 설정**: `.vscode/launch.json` 예시
```json
{
  "version": "0.2.0",
  "configurations": [
    {
      "name": "Django Server",
      "type": "python",
      "request": "launch",
      "program": "${workspaceFolder}/manage.py",
      "args": ["runserver", "0.0.0.0:8000"],
      "django": true,
      "justMyCode": true,
      "envFile": "${workspaceFolder}/.env"
    }
  ]
}
```
- **포맷/린트 자동화**: `.vscode/settings.json`
```json
{
  "python.formatting.provider": "black",
  "editor.formatOnSave": true,
  "python.linting.enabled": true,
  "python.analysis.typeCheckingMode": "basic",
  "ruff.lint.args": ["--fix"]
}
```

### 디렉터리 구조 베스트프랙티스

- **권장 구조(단일 레포 내 다중 앱)**
```
myproject/
├─ .env                # 비밀/환경 변수
├─ .env.example        # 공유 가능한 샘플(민감값은 공란/더미)
├─ .gitignore
├─ pyproject.toml / requirements.txt
├─ manage.py
├─ config/             # 프로젝트(사이트) 설정 패키지
│  ├─ __init__.py
│  ├─ asgi.py
│  ├─ wsgi.py
│  ├─ urls.py
│  └─ settings/
│     ├─ __init__.py
│     ├─ base.py      # 공통 설정
│     ├─ dev.py       # 개발 설정
│     └─ prod.py      # 운영 설정
├─ apps/
│  └─ core/            # 첫 앱(예: 공용/홈)
│     ├─ __init__.py
│     ├─ admin.py
│     ├─ apps.py
│     ├─ models.py
│     ├─ urls.py
│     ├─ views.py
│     ├─ forms.py
│     ├─ tests/
│     │  └─ test_home.py
│     └─ templates/
│        └─ core/
│           └─ home.html
└─ static/             # 정적 파일(선택)
```

---

## 0-3. 첫 프로젝트/앱 생성 — `django-admin` vs `manage.py`, 설정 분리(dev/prod)

### django-admin과 manage.py의 차이

- `django-admin`: **글로벌 진입점**(가상환경 PATH에 노출). 어떤 프로젝트 맥락 없이도 명령 실행 가능
- `manage.py`: **프로젝트 로컬 진입점**. 내부에서 `DJANGO_SETTINGS_MODULE`을 지정해 **해당 프로젝트 설정**을 자동 로드
- 실무에서는 **프로젝트 하위에서 `python manage.py <cmd>`** 사용이 안전하고 명확하다.

간단 실험:
```bash
# 프로젝트 외부에서 가능 (django-admin)

django-admin startproject config .

# 프로젝트 내부에서 권장 (manage.py)

python manage.py runserver
python manage.py makemigrations
python manage.py migrate
```

### 프로젝트 생성 & 기본 기동

```bash
# 작업 루트 생성

mkdir myproject && cd myproject

# 또는 poetry shell 활성화 후

pip install "django>=5.0,<6.0" django-environ

# 프로젝트 생성 (현재 폴더에 생성하려면 끝의 '.')

django-admin startproject config .

# 서버 기동

python manage.py runserver 0.0.0.0:8000
```

폴더 상태 확인:
```
myproject/
├─ manage.py
└─ config/
   ├─ __init__.py
   ├─ asgi.py
   ├─ settings.py  # 초기에는 단일 파일
   ├─ urls.py
   └─ wsgi.py
```

### 앱 생성 & URL 연결

```bash
python manage.py startapp core  # apps/core 생성
```

`apps/core/views.py`:
```python
from django.http import HttpResponse

def home(request):
    return HttpResponse("Hello, Django!")
```

`apps/core/urls.py`:
```python
from django.urls import path
from . import views

app_name = "core"

urlpatterns = [
    path("", views.home, name="home"),
]
```

`config/urls.py`에 연결:
```python
from django.contrib import admin
from django.urls import path, include

urlpatterns = [
    path("admin/", admin.site.urls),
    path("", include("apps.core.urls")),
]
```

`config/settings.py`에 앱 등록:
```python
INSTALLED_APPS = [
    "django.contrib.admin",
    "django.contrib.auth",
    "django.contrib.contenttypes",
    "django.contrib.sessions",
    "django.contrib.messages",
    "django.contrib.staticfiles",
    "apps.core",  # 추가
]
```

브라우저 `http://127.0.0.1:8000/` → “Hello, Django!” 표시.

### 템플릿(DTL)로 전환

`apps/core/templates/core/home.html`:
{% raw %}
```html
<!doctype html>
<html>
  <head>
    <meta charset="utf-8" />
    <title>Home</title>
  </head>
  <body>
    <h1>{{ title }}</h1>
    <p>Welcome to Django starter.</p>
  </body>
</html>
```
{% endraw %}

`apps/core/views.py`:
```python
from django.shortcuts import render

def home(request):
    ctx = {"title": "Django Starter Home"}
    return render(request, "core/home.html", ctx)
```

템플릿 검색 경로 설정(`config/settings.py`):
```python
import os
from pathlib import Path

BASE_DIR = Path(__file__).resolve().parent.parent

TEMPLATES = [
    {
        "BACKEND": "django.template.backends.django.DjangoTemplates",
        "DIRS": [BASE_DIR / "apps" / "core" / "templates"],  # 필요 시 전역 templates 디렉터리 추가 가능
        "APP_DIRS": True,
        "OPTIONS": {
            "context_processors": [
                "django.template.context_processors.debug",
                "django.template.context_processors.request",
                "django.contrib.auth.context_processors.auth",
                "django.contrib.messages.context_processors.messages",
            ],
        },
    },
]
```
> 팁: 규모가 커지면 `templates/`를 최상위에 만들고 각 앱에서 **네임스페이스(폴더명)**로 분리해 충돌을 방지한다.

### 정적 파일 셋업(개발 기본)

`config/settings.py`:
```python
STATIC_URL = "static/"
STATICFILES_DIRS = [
    BASE_DIR / "static",
]
```
개발 중에는 `runserver`가 자동 서빙. 운영에서는 Nginx 등 별도 서빙 권장.

### 데이터베이스 & 마이그레이션 기본

- 기본 SQLite 사용(시작에 충분)
- 모델 작성 → `makemigrations` → `migrate`

`apps/core/models.py`:
```python
from django.db import models

class Article(models.Model):
    title = models.CharField(max_length=200)
    body = models.TextField()
    is_published = models.BooleanField(default=False)
    created_at = models.DateTimeField(auto_now_add=True)

    def __str__(self):
        return f"[{self.id}] {self.title}"
```

마이그레이션:
```bash
python manage.py makemigrations
python manage.py migrate
```

Django Admin 등록:
```python
# apps/core/admin.py

from django.contrib import admin
from .models import Article

@admin.register(Article)
class ArticleAdmin(admin.ModelAdmin):
    list_display = ("id", "title", "is_published", "created_at")
    list_filter = ("is_published",)
    search_fields = ("title",)
```

슈퍼유저 생성:
```bash
python manage.py createsuperuser
```
관리자: `http://127.0.0.1:8000/admin/`

---

## — django-environ + 다중 설정 모듈

### 왜 분리하나?

- **12-Factor** 원칙: 코드와 설정 분리, 환경변수로 비밀/환경값 관리
- 개발/운영 **동일 코드**로 **환경만 바꿔** 배포
- 실수 방지: 디버그, 로깅 레벨, DB/캐시/도메인 등 **환경별 차이**를 명시적으로 분기

### django-environ으로 .env 읽기

설치:
```bash
pip install django-environ
```

`.env` (레포에 커밋 금지!):
```
DJANGO_DEBUG=True
DJANGO_SECRET_KEY=change-me-please
DJANGO_ALLOWED_HOSTS=127.0.0.1,localhost
DJANGO_DB_URL=sqlite:///db.sqlite3
```

`.env.example` (공유용 템플릿):
```
DJANGO_DEBUG=
DJANGO_SECRET_KEY=
DJANGO_ALLOWED_HOSTS=
DJANGO_DB_URL=
```

### settings 디렉터리 분리

```
config/settings/
├─ __init__.py
├─ base.py
├─ dev.py
└─ prod.py
```

`config/settings/base.py`:
```python
from pathlib import Path
import environ

BASE_DIR = Path(__file__).resolve().parent.parent.parent

env = environ.Env(
    DJANGO_DEBUG=(bool, False),
)
env.read_env(BASE_DIR / ".env")  # .env 파일 로드

DEBUG = env("DJANGO_DEBUG")

SECRET_KEY = env("DJANGO_SECRET_KEY", default="!!replace-this!!")
ALLOWED_HOSTS = env.list("DJANGO_ALLOWED_HOSTS", default=["127.0.0.1", "localhost"])

INSTALLED_APPS = [
    "django.contrib.admin",
    "django.contrib.auth",
    "django.contrib.contenttypes",
    "django.contrib.sessions",
    "django.contrib.messages",
    "django.contrib.staticfiles",
    "apps.core",
]

MIDDLEWARE = [
    "django.middleware.security.SecurityMiddleware",
    "django.contrib.sessions.middleware.SessionMiddleware",
    "django.middleware.common.CommonMiddleware",
    "django.middleware.csrf.CsrfViewMiddleware",
    "django.contrib.auth.middleware.AuthenticationMiddleware",
    "django.contrib.messages.middleware.MessageMiddleware",
    "django.middleware.clickjacking.XFrameOptionsMiddleware",
]

ROOT_URLCONF = "config.urls"
WSGI_APPLICATION = "config.wsgi.application"
ASGI_APPLICATION = "config.asgi.application"

# DB: DATABASE_URL 형식 (sqlite:///db.sqlite3, postgres://user:pass@host:port/db)

DATABASES = {
    "default": env.db("DJANGO_DB_URL", default=f"sqlite:///{BASE_DIR / 'db.sqlite3'}")
}

# Templates

TEMPLATES = [
    {
        "BACKEND": "django.template.backends.django.DjangoTemplates",
        "DIRS": [BASE_DIR / "templates"],  # 전역 템플릿 디렉터리(선택)
        "APP_DIRS": True,
        "OPTIONS": {
            "context_processors": [
                "django.template.context_processors.debug",
                "django.template.context_processors.request",
                "django.contrib.auth.context_processors.auth",
                "django.contrib.messages.context_processors.messages",
            ],
        },
    },
]

# Static & Media

STATIC_URL = "static/"
STATIC_ROOT = BASE_DIR / "staticfiles"  # collectstatic 대상(운영)
STATICFILES_DIRS = [BASE_DIR / "static"]  # 개발용

MEDIA_URL = "media/"
MEDIA_ROOT = BASE_DIR / "media"

# 국제화

LANGUAGE_CODE = "ko-kr"
TIME_ZONE = "Asia/Seoul"
USE_I18N = True
USE_TZ = True

DEFAULT_AUTO_FIELD = "django.db.models.BigAutoField"
```

`config/settings/dev.py`:
```python
from .base import *

DEBUG = True

# 개발 편의

INTERNAL_IPS = ["127.0.0.1"]

# 이메일(콘솔로 출력)

EMAIL_BACKEND = "django.core.mail.backends.console.EmailBackend"

# 보안(개발 완화)

CSRF_COOKIE_SECURE = False
SESSION_COOKIE_SECURE = False
SECURE_SSL_REDIRECT = False
```

`config/settings/prod.py`:
```python
from .base import *

DEBUG = False

# 보안 헤더/HTTPS(리버스 프록시 환경 고려)

SECURE_SSL_REDIRECT = True
CSRF_COOKIE_SECURE = True
SESSION_COOKIE_SECURE = True
SECURE_HSTS_SECONDS = 60 * 60 * 24 * 30  # 30일
SECURE_HSTS_INCLUDE_SUBDOMAINS = True
SECURE_HSTS_PRELOAD = True

# 로깅(예: JSON 구조화 로깅으로 교체 가능)

LOGGING = {
    "version": 1,
    "disable_existing_loggers": False,
    "handlers": {
        "console": {"class": "logging.StreamHandler"},
    },
    "root": {"handlers": ["console"], "level": "INFO"},
}
```

**실행 시 설정 모듈 지정**
- `manage.py`는 기본 `DJANGO_SETTINGS_MODULE="config.settings"`를 가정(단일 파일일 때)
- 다중 파일로 바꿨다면 **명시**:
```bash
# 개발

DJANGO_SETTINGS_MODULE=config.settings.dev python manage.py runserver

# 운영 (gunicorn/uvicorn 서비스 환경 변수에 지정)

DJANGO_SETTINGS_MODULE=config.settings.prod gunicorn config.wsgi:application
```

윈도우 PowerShell:
```powershell
$env:DJANGO_SETTINGS_MODULE="config.settings.dev"
python manage.py runserver
```

또는 `.env`에 설정:
```
DJANGO_SETTINGS_MODULE=config.settings.dev
```
`manage.py`/`wsgi.py`/`asgi.py`에서 `os.environ.setdefault(...)`로 읽는다.

`manage.py` 확인:
```python
#!/usr/bin/env python

import os
import sys

def main():
    os.environ.setdefault("DJANGO_SETTINGS_MODULE", "config.settings.dev")  # 기본 dev로
    from django.core.management import execute_from_command_line
    execute_from_command_line(sys.argv)

if __name__ == "__main__":
    main()
```

> 주: 팀 합의에 따라 `dev`를 기본으로 두고, 배포 파이프라인에서 `prod`로 주입하거나, **항상 환경변수 우선**으로 강제하는 패턴 중 하나를 채택.

---

## 0-5. 개발 품질 기본 세팅(선택 강추)

### pre-commit 훅으로 포맷/린트

`.pre-commit-config.yaml`:
```yaml
repos:
  - repo: https://github.com/psf/black
    rev: 24.3.0
    hooks:
      - id: black
  - repo: https://github.com/charliermarsh/ruff-pre-commit
    rev: v0.5.0
    hooks:
      - id: ruff
  - repo: https://github.com/pycqa/isort
    rev: 5.13.2
    hooks:
      - id: isort
```
설치:
```bash
pip install pre-commit
pre-commit install
```

### pytest/pytest-django 기본

`pytest.ini`:
```ini
[pytest]
DJANGO_SETTINGS_MODULE = config.settings.dev
python_files = tests.py test_*.py *_tests.py
```

샘플 테스트 `apps/core/tests/test_home.py`:
```python
import pytest
from django.urls import reverse

@pytest.mark.django_db
def test_home(client):
    resp = client.get(reverse("core:home"))
    assert resp.status_code == 200
    assert b"Django" in resp.content
```

---

## 0-6. 실무 체크리스트(시작 단계)

**프로젝트 기동 전**
- [ ] Python 3.10+ / venv(or Poetry)
- [ ] `pip install django django-environ`
- [ ] `django-admin startproject config .`
- [ ] `python manage.py startapp core`
- [ ] `apps.core`를 `INSTALLED_APPS`에 등록
- [ ] `core.urls`를 `config.urls`에 include
- [ ] 기본 `home` 뷰 + 템플릿 연결

**설정 분리/보안**
- [ ] `config/settings/base|dev|prod.py` 구성
- [ ] `.env`로 `SECRET_KEY`, DB URL, ALLOWED_HOSTS 관리
- [ ] `DEBUG=False`(운영), HSTS/SSL 리다이렉트/보안 쿠키 설정
- [ ] `collectstatic` 경로·Nginx 정적 서빙 설계

**운영 진입 전**
- [ ] Admin 계정/권한/감사 로그 전략
- [ ] 기본 로깅(JSON/구조화) 또는 APM/Sentry 연동 설계
- [ ] DB 마이그레이션/백업 절차
- [ ] 헬스체크 엔드포인트(예: `/healthz`)
- [ ] 간단한 e2e 테스트(CI에서 `manage.py test`)

---

## 0-7. 확장 예제 — 간단한 CRUD 스켈레톤(Article)

### URL

```python
# apps/core/urls.py

from django.urls import path
from . import views

app_name = "core"

urlpatterns = [
    path("", views.home, name="home"),
    path("articles/", views.article_list, name="article_list"),
    path("articles/<int:pk>/", views.article_detail, name="article_detail"),
    path("articles/new/", views.article_new, name="article_new"),
]
```

### Views

```python
# apps/core/views.py

from django.shortcuts import render, get_object_or_404, redirect
from .models import Article
from .forms import ArticleForm

def home(request):
    return render(request, "core/home.html", {"title": "Home"})

def article_list(request):
    qs = Article.objects.order_by("-created_at")
    return render(request, "core/article_list.html", {"articles": qs})

def article_detail(request, pk: int):
    article = get_object_or_404(Article, pk=pk)
    return render(request, "core/article_detail.html", {"a": article})

def article_new(request):
    if request.method == "POST":
        form = ArticleForm(request.POST)
        if form.is_valid():
            a = form.save()
            return redirect("core:article_detail", pk=a.pk)
    else:
        form = ArticleForm()
    return render(request, "core/article_form.html", {"form": form})
```

### Forms

```python
# apps/core/forms.py

from django import forms
from .models import Article

class ArticleForm(forms.ModelForm):
    class Meta:
        model = Article
        fields = ["title", "body", "is_published"]
```

### Templates

`apps/core/templates/core/article_list.html`:
{% raw %}
```html
{% extends "base.html" %}
{% block content %}
<h1>Articles</h1>
<ul>
  {% for a in articles %}
  <li>
    <a href="{% url 'core:article_detail' a.pk %}">{{ a.title }}</a>
    <small>{{ a.created_at }}</small>
  </li>
  {% empty %}
  <li>No articles.</li>
  {% endfor %}
</ul>
<p><a href="{% url 'core:article_new' %}">New Article</a></p>
{% endblock %}
```
{% endraw %}

`apps/core/templates/core/article_detail.html`:
{% raw %}
```html
{% extends "base.html" %}
{% block content %}
<h1>{{ a.title }}</h1>
<p>{{ a.body|linebreaks }}</p>
<p>Published: {{ a.is_published }}</p>
<p><a href="{% url 'core:article_list' %}">Back</a></p>
{% endblock %}
```
{% endraw %}

`apps/core/templates/core/article_form.html`:
{% raw %}
```html
{% extends "base.html" %}
{% block content %}
<h1>New Article</h1>
<form method="post">
  {% csrf_token %}
  {{ form.as_p }}
  <button type="submit">Save</button>
</form>
{% endblock %}
```
{% endraw %}

전역 베이스(`templates/base.html`, 선택):
{% raw %}
```html
<!doctype html>
<html>
  <head>
    <meta charset="utf-8" />
    <title>{% block title %}My Django{% endblock %}</title>
  </head>
  <body>
    <nav>
      <a href="{% url 'core:home' %}">Home</a> |
      <a href="{% url 'core:article_list' %}">Articles</a>
    </nav>
    <hr />
    {% block content %}{% endblock %}
  </body>
</html>
```
{% endraw %}

---

## 0-8. 운영을 미리 생각하는 습관(아주 기초)

- **비밀 관리**: `.env`는 절대 커밋하지 말 것. 배포 환경은 **환경변수**로 주입.
- **ALLOWED_HOSTS/CSRF_TRUSTED_ORIGINS** 정확히 지정.
- **정적/미디어 파일**: `collectstatic` → Nginx/CloudFront로 서빙 설계.
- **DB 마이그레이션**: 배포 파이프라인에서 자동화(`python manage.py migrate`).
- **건강 체크/장애 대응**: 로깅 레벨, 타임아웃, 리버스 프록시 타임아웃, 5xx 가시성.
- **테스트 최소한**: 뷰/모델/URL 역방향/폼 검증 1~2개만 있어도 실수 크게 줄어든다.

---

## 부록 A. 자주 만나는 에러/함정 Top 10

1. `ALLOWED_HOSTS` 미설정 → 운영에서 400.
2. `DEBUG=True`로 배포 → 내부 정보 노출 위험.
3. 시크릿/DB 자격증명 커밋 → **레포 이력 영구 오염**. 즉시 폐기·재발급.
4. `TIME_ZONE`/`USE_TZ` 혼동 → 날짜가 어긋남. `Asia/Seoul` + `USE_TZ=True` 권장.
5. 마이그레이션 충돌 → 브랜치 간 모델 변경 병합 시 `--merge`와 주석 확인.
6. 정적/미디어 혼동 → 개발에서 보이는데 운영에서 404. `collectstatic`·Nginx 설정 점검.
7. 템플릿 경로/네임스페이스 충돌 → 앱별 폴더 네임스페이스로 분리.
8. 폼 검증 누락 → 빈 값·비정상 입력 저장. `form.is_valid()` 확인 및 에러 메시지 렌더링.
9. 권한 체크 누락 → Admin/Login 보호 필요.
10. 메모리 캐시 오용 → 프로세스 재기동 시 휘발. Redis 등 외부 캐시 사용 고려.

---

## 부록 B. 명령어 치트시트

```bash
# 새 프로젝트/앱

django-admin startproject config .
python manage.py startapp core

# 서버/DB

python manage.py runserver
python manage.py makemigrations
python manage.py migrate
python manage.py createsuperuser

# 정적파일 수집(운영)

python manage.py collectstatic

# 테스트

python manage.py test
pytest -q  # pytest 사용 시
```

---

## 마무리

이 글 하나로 **Django의 철학(MTV)**, **환경 구성(venv/Poetry/VS Code)**, **프로젝트/앱 생성**, **설정 분리(dev/prod)**, 그리고 **간단 CRUD**까지 **처음부터 끝**을 실행할 수 있다.
