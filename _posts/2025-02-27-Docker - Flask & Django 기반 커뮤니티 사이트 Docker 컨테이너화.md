---
layout: post
title: Docker - Flask & Django 기반 커뮤니티 사이트 Docker 컨테이너화
date: 2025-02-27 19:20:23 +0900
category: Docker
---
# Flask/Django 기반 커뮤니티 사이트의 Docker 컨테이너화

## 최종 목표와 아키텍처 개요

- 웹 프레임워크: **Flask** 또는 **Django** (둘 다 예시 제공)
- 데이터베이스: **PostgreSQL 15** (필요 시 MySQL 전환 예시 포함)
- 컨테이너 오케스트레이션: **Docker Compose (v2)**
- 핵심 요구:
  - 개발/운영 공통 **컨테이너 이미지**로 일관성 확보
  - **.env** 기반 **환경설정 분리**(dev/prod)
  - **볼륨**으로 DB/정적파일 영속화
  - **헬스체크**와 **대기 스크립트(wait-for-db)**로 기동 안정성
  - 운영 시 **Gunicorn + Nginx + HTTPS(선택)**

---

## — 최소 구성

```plaintext
community-app/
├── app/
│   ├── __init__.py
│   ├── models.py
│   ├── views.py
│   ├── templates/
│   └── wsgi.py
├── requirements.txt
├── Dockerfile
├── docker-compose.yml
├── docker-compose.prod.yml
├── .env           # dev 기본값
├── .env.prod      # prod 값 (별도 보관)
└── docker/
    ├── wait-for-db.sh
    ├── nginx/
    │   ├── nginx.conf
    │   └── mime.types
    └── gunicorn.conf.py
```

> Django는 `manage.py`, 프로젝트 패키지(예: `community/`), `settings.py` 존재. 아래에서 Django 전용 파일도 제공.

---

## Python 요구사항 파일

### Flask용 `requirements.txt`

```
Flask==2.3.3
Flask_SQLAlchemy==3.1.1
psycopg2-binary==2.9.9
gunicorn==21.2.0
python-dotenv==1.0.1
```

> MySQL을 쓸 경우 `psycopg2-binary` 대신 `mysqlclient` 또는 `PyMySQL` 사용.

### Django용 `requirements.txt`

```
Django==4.2.16
psycopg2-binary==2.9.9
gunicorn==21.2.0
whitenoise==6.7.0
python-dotenv==1.0.1
```

---

## Dockerfile — 멀티스테이지 + 런타임 최적화

두 프레임워크 공용으로 쓸 수 있는 기본 이미지(엔트리 명령만 다르게 지정).

```Dockerfile
# Dockerfile

FROM python:3.11-slim AS base

ENV PYTHONDONTWRITEBYTECODE=1 \
    PYTHONUNBUFFERED=1

# 시스템 패키지 (빌드/런타임 공용 최소)

RUN apt-get update && apt-get install -y --no-install-recommends \
    build-essential curl netcat-openbsd ca-certificates \
 && rm -rf /var/lib/apt/lists/*

WORKDIR /app

# 의존성만 먼저 복사 → 캐시 최대로 활용

COPY requirements.txt .
RUN pip install --no-cache-dir -r requirements.txt

# 앱 소스 복사

COPY . .

# wait-for-db 스크립트 실행 권한

RUN chmod +x docker/wait-for-db.sh

# — 운영에서는 Gunicorn 사용

EXPOSE 5000

# 기본 CMD는 환경변수로 전환 가능(FLASK/DJANGO)
# 개발 디폴트 (Flask)

CMD ["python", "-m", "flask", "run", "--host=0.0.0.0", "--port=5000"]
```

> Django의 경우 Compose에서 `command`를 덮어써서 `gunicorn community.wsgi:application` 또는 `python manage.py runserver`를 실행.

---

## Flask 애플리케이션 샘플

### `app/models.py`

```python
from flask_sqlalchemy import SQLAlchemy

db = SQLAlchemy()

class Post(db.Model):
    __tablename__ = "posts"
    id = db.Column(db.Integer, primary_key=True)
    title = db.Column(db.String(120), nullable=False)
    content = db.Column(db.Text, nullable=False)
```

### `app/__init__.py`

```python
import os
from flask import Flask
from .models import db

def create_app():
    app = Flask(__name__)

    # 예: postgresql://USER:PASSWORD@HOST:PORT/DB
    db_user = os.getenv("POSTGRES_USER", "admin")
    db_pass = os.getenv("POSTGRES_PASSWORD", "securepass")
    db_name = os.getenv("POSTGRES_DB", "community_db")
    db_host = os.getenv("DB_HOST", "db")
    db_port = os.getenv("DB_PORT", "5432")

    app.config["SQLALCHEMY_DATABASE_URI"] = (
        f"postgresql://{db_user}:{db_pass}@{db_host}:{db_port}/{db_name}"
    )
    app.config["SQLALCHEMY_TRACK_MODIFICATIONS"] = False

    db.init_app(app)
    return app
```

### `app/views.py`

```python
from flask import Blueprint, jsonify, request
from .models import db, Post

bp = Blueprint("bp", __name__)

@bp.route("/")
def index():
    return jsonify({"message": "Flask + Postgres OK"})

@bp.route("/posts", methods=["GET"])
def list_posts():
    posts = Post.query.order_by(Post.id.desc()).all()
    return jsonify([{"id": p.id, "title": p.title, "content": p.content} for p in posts])

@bp.route("/posts", methods=["POST"])
def create_post():
    data = request.get_json(force=True)
    p = Post(title=data.get("title", ""), content=data.get("content", ""))
    db.session.add(p)
    db.session.commit()
    return jsonify({"id": p.id}), 201
```

### `app/wsgi.py`

```python
from . import create_app
from .views import bp

app = create_app()
app.register_blueprint(bp)
```

> 개발 모드에서는 `flask run`(환경변수 `FLASK_APP=app/wsgi.py`), 운영에서는 `gunicorn -c docker/gunicorn.conf.py app.wsgi:app` 사용.

---

## Django 애플리케이션 포인트(간략 예시)

### `community/settings.py` (핵심 DB/정적 설정만)

```python
import os
from pathlib import Path

BASE_DIR = Path(__file__).resolve().parent.parent

SECRET_KEY = os.getenv("DJANGO_SECRET_KEY", "dev-secret")
DEBUG = os.getenv("DJANGO_DEBUG", "true").lower() == "true"

ALLOWED_HOSTS = os.getenv("DJANGO_ALLOWED_HOSTS", "localhost,127.0.0.1").split(",")

INSTALLED_APPS = [
    "django.contrib.admin", "django.contrib.auth", "django.contrib.contenttypes",
    "django.contrib.sessions", "django.contrib.messages", "django.contrib.staticfiles",
    # 예: 커뮤니티 앱
    "posts",
    # Whitenoise
    "whitenoise.runserver_nostatic",
]

MIDDLEWARE = [
    "django.middleware.security.SecurityMiddleware",
    "whitenoise.middleware.WhiteNoiseMiddleware",  # 정적 파일 서빙
    "django.contrib.sessions.middleware.SessionMiddleware",
    "django.middleware.common.CommonMiddleware",
    "django.middleware.csrf.CsrfViewMiddleware",
    "django.contrib.auth.middleware.AuthenticationMiddleware",
    "django.contrib.messages.middleware.MessageMiddleware",
]

ROOT_URLCONF = "community.urls"
WSGI_APPLICATION = "community.wsgi.application"

DATABASES = {
    "default": {
        "ENGINE": "django.db.backends.postgresql",
        "NAME": os.getenv("POSTGRES_DB", "community_db"),
        "USER": os.getenv("POSTGRES_USER", "admin"),
        "PASSWORD": os.getenv("POSTGRES_PASSWORD", "securepass"),
        "HOST": os.getenv("DB_HOST", "db"),
        "PORT": int(os.getenv("DB_PORT", "5432")),
    }
}

STATIC_URL = "/static/"
STATIC_ROOT = BASE_DIR / "staticfiles"
# gzip/brotli 사용 가능 (설정 추가 가능)

```

### Django 운영 실행 예

- 개발: `python manage.py runserver 0.0.0.0:8000`
- 운영(Gunicorn): `gunicorn community.wsgi:application -c docker/gunicorn.conf.py`

---

## DB 준비 대기 스크립트 `docker/wait-for-db.sh`

DB가 뜨기 전에 앱이 연결을 시도하면 실패하므로, 간단한 대기 스크립트를 사용한다.

```bash
#!/usr/bin/env bash

set -euo pipefail

HOST="${DB_HOST:-db}"
PORT="${DB_PORT:-5432}"

echo "[wait-for-db] Waiting for ${HOST}:${PORT} …"
until nc -z "${HOST}" "${PORT}"; do
  sleep 1
done
echo "[wait-for-db] DB is up."
exec "$@"
```

---

## Compose — 개발용 `docker-compose.yml`

- 실시간 코드 반영을 위해 **바인드 마운트**
- **env_file** 주입
- **depends_on + healthcheck**로 최소한의 준비 확인
- 시작 시 `wait-for-db.sh`로 DB 준비 대기 후 앱 실행

```yaml
# docker-compose.yml (dev)

version: "3.9"

services:
  web:
    build: .
    container_name: community_web
    ports:
      - "${WEB_PORT:-5000}:5000"
    env_file:
      - .env
    environment:
      # Flask 전용. Django일 경우 command에서 runserver 지정.
      FLASK_APP: "app/wsgi.py"
      FLASK_ENV: "${FLASK_ENV:-development}"
      DB_HOST: "db"
      DB_PORT: "${DB_PORT:-5432}"
    volumes:
      - .:/app
    depends_on:
      db:
        condition: service_healthy
    command: [ "bash", "-lc", "docker/wait-for-db.sh && python -m flask run --host=0.0.0.0 --port=5000" ]
    networks: [ app-net ]

  db:
    image: postgres:15
    container_name: community_db
    environment:
      POSTGRES_DB: "${POSTGRES_DB:-community_db}"
      POSTGRES_USER: "${POSTGRES_USER:-admin}"
      POSTGRES_PASSWORD: "${POSTGRES_PASSWORD:-securepass}"
    volumes:
      - db-data:/var/lib/postgresql/data
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U $$POSTGRES_USER -d $$POSTGRES_DB -h 127.0.0.1"]
      interval: 5s
      timeout: 3s
      retries: 20
    networks: [ app-net ]

volumes:
  db-data:

networks:
  app-net:
```

> Django 개발로 바꾸려면 `web.command`를
> `bash -lc "docker/wait-for-db.sh && python manage.py migrate && python manage.py runserver 0.0.0.0:8000"`
> 로 교체하고, `ports`도 `"${WEB_PORT:-8000}:8000"`로 바꾼다.

---

## Compose — 운영용 `docker-compose.prod.yml`

- **Gunicorn**으로 WSGI 서비스
- **Nginx 리버스 프록시**(정적 캐싱 및 SSL 종단 선택)
- **볼륨 영속화**(DB, 정적/미디어)
- **리소스 제한**(예: 메모리/CPU, 필요 시 추가)

```yaml
# docker-compose.prod.yml

version: "3.9"

services:
  web:
    build: .
    container_name: community_web
    depends_on:
      db:
        condition: service_healthy
    env_file:
      - .env.prod
    environment:
      DB_HOST: "db"
      DB_PORT: "${DB_PORT:-5432}"
      DJANGO_DEBUG: "false"
    command: [ "bash", "-lc", "docker/wait-for-db.sh && gunicorn app.wsgi:app -c docker/gunicorn.conf.py" ]
    expose:
      - "8000"
    networks: [ app-net ]

  nginx:
    image: nginx:1.27-alpine
    container_name: community_nginx
    depends_on: [ web ]
    ports:
      - "80:80"
      # - "443:443"  # certbot 연동 시 오픈
    volumes:
      - ./docker/nginx/nginx.conf:/etc/nginx/nginx.conf:ro
      - ./docker/nginx/mime.types:/etc/nginx/mime.types:ro
      # Django일 경우 staticfiles/media 폴더 마운트 예:
      # - static-data:/app/staticfiles:ro
      # - media-data:/app/media:ro
    networks: [ app-net ]

  db:
    image: postgres:15
    container_name: community_db
    environment:
      POSTGRES_DB: "${POSTGRES_DB}"
      POSTGRES_USER: "${POSTGRES_USER}"
      POSTGRES_PASSWORD: "${POSTGRES_PASSWORD}"
    volumes:
      - db-data:/var/lib/postgresql/data
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U $$POSTGRES_USER -d $$POSTGRES_DB -h 127.0.0.1"]
      interval: 5s
      timeout: 3s
      retries: 20
    networks: [ app-net ]

volumes:
  db-data:
  # static-data:
  # media-data:

networks:
  app-net:
```

> Flask 운영 시 `command`의 `gunicorn app.wsgi:app` 경로 유지.
> Django 운영 시 `command`를 `gunicorn community.wsgi:application -c docker/gunicorn.conf.py`로 교체하고 Nginx에서 정적/미디어 경로를 매핑.

---

## Gunicorn & Nginx 설정

### `docker/gunicorn.conf.py`

```python
bind = "0.0.0.0:8000"
workers = 2           # CPU 코어 * 2 + 1 등으로 조정
worker_class = "gthread"
threads = 4
timeout = 60
accesslog = "-"
errorlog = "-"
loglevel = "info"
```

### `docker/nginx/nginx.conf` (정석 예시)

```nginx
user  nginx;
worker_processes auto;
pid /var/run/nginx.pid;

events { worker_connections 1024; }

http {
  include       /etc/nginx/mime.types;
  sendfile      on;
  tcp_nopush    on;
  tcp_nodelay   on;
  keepalive_timeout  65;

  upstream app_upstream {
    server web:8000;
  }

  server {
    listen 80;
    server_name _;

    # Django 정적/미디어 예시:
    # location /static/ { alias /app/staticfiles/; expires 7d; }
    # location /media/  { alias /app/media/;      expires 7d; }

    location / {
      proxy_pass         http://app_upstream;
      proxy_http_version 1.1;
      proxy_set_header   Host              $host;
      proxy_set_header   X-Real-IP         $remote_addr;
      proxy_set_header   X-Forwarded-For   $proxy_add_x_forwarded_for;
      proxy_set_header   X-Forwarded-Proto $scheme;
      proxy_read_timeout 60s;
    }
  }
}
```

---

## .env 샘플(개발/운영 분리)

### `.env` (dev)

```
WEB_PORT=5000
POSTGRES_DB=community_db
POSTGRES_USER=admin
POSTGRES_PASSWORD=securepass
FLASK_ENV=development
DB_PORT=5432
```

### `.env.prod` (prod)

```
POSTGRES_DB=community_db
POSTGRES_USER=admin
POSTGRES_PASSWORD=CHANGE_ME_STRONG
DB_PORT=5432
DJANGO_SECRET_KEY=replace_me
DJANGO_ALLOWED_HOSTS=yourdomain.com,localhost
```

> `.env*`와 인증서/비밀키는 반드시 `.gitignore`에 포함.

---

## 마이그레이션·초기화 절차

### — 간단 초기화

```bash
docker compose run --rm web python -c \
"from app import create_app; from app.models import db; app=create_app(); \
 from app.models import Post; \
 (print('Creating tables…'), db.init_app(app)); \
 [ (db.create_all(), print('Done')) for _ in [0] ]"
```

또는 Flask CLI를 구성한 뒤 `flask shell`로 실행.

> 장기 운영/버전업에는 **Alembic** 사용 권장:
> - `pip install alembic`
> - `alembic init migrations` → env.py에 SQLAlchemy 연결
> - 스키마 변경 시 `alembic revision --autogenerate -m "msg"` → `alembic upgrade head`

### Django — 마이그레이션

```bash
docker compose run --rm web bash -lc \
"python manage.py migrate && python manage.py collectstatic --noinput"
docker compose run --rm web bash -lc "python manage.py createsuperuser"
```

---

## MySQL로 전환하기(선택)

### Compose 변경

```yaml
db:
  image: mysql:8.0
  environment:
    MYSQL_DATABASE: "${MYSQL_DATABASE:-community_db}"
    MYSQL_USER: "${MYSQL_USER:-admin}"
    MYSQL_PASSWORD: "${MYSQL_PASSWORD:-securepass}"
    MYSQL_ROOT_PASSWORD: "${MYSQL_ROOT_PASSWORD:-rootpass}"
  command: --default-authentication-plugin=mysql_native_password
  volumes:
    - db-data:/var/lib/mysql
  healthcheck:
    test: ["CMD-SHELL", "mysqladmin ping -h 127.0.0.1 -u $$MYSQL_USER -p$$MYSQL_PASSWORD --silent"]
    interval: 5s
    timeout: 5s
    retries: 20
```

### 앱 연결 문자열

- Flask: `mysql+pymysql://admin:securepass@db:3306/community_db`
- Django: `ENGINE: django.db.backends.mysql` 및 포트 3306

> 패키지: `PyMySQL` 또는 `mysqlclient` 설치 필요.

---

## 헬스체크/레디니스 전략

- DB 컨테이너: `pg_isready` 또는 `mysqladmin ping`
- 웹 컨테이너: 어플리케이션 자체 헬스 엔드포인트(예: `/healthz`) 추가 후 Nginx/Gateway에서 사용

Flask 예시:
```python
@bp.route("/healthz")
def healthz():
    return "ok", 200
```

Nginx에서 /healthz는 `app_upstream` 프록시 그대로 통과시켜 상태 확인.

---

## 백업/복구 전략(핵심만)

### PostgreSQL

```bash
# 백업

docker compose exec db pg_dump -U "$POSTGRES_USER" -d "$POSTGRES_DB" > backup.sql

# 복구

cat backup.sql | docker compose exec -T db psql -U "$POSTGRES_USER" -d "$POSTGRES_DB"
```

### MySQL

```bash
# 백업

docker compose exec db sh -lc 'mysqldump -u"$MYSQL_USER" -p"$MYSQL_PASSWORD" "$MYSQL_DATABASE"' > backup.sql

# 복구

cat backup.sql | docker compose exec -T db sh -lc 'mysql -u"$MYSQL_USER" -p"$MYSQL_PASSWORD" "$MYSQL_DATABASE"'
```

> 주기적인 백업은 CI/CD 또는 크론 컨테이너로 자동화.

---

## 로깅/모니터링·리소스 제한

- **로깅**: 기본 json-file 드라이버 로테이션
- **리소스 제한**: 컨테이너별 `deploy.resources.limits`(Swarm) 또는 `mem_limit`, `cpus`(Compose 확장)

```yaml
services:
  web:
    logging:
      driver: "json-file"
      options:
        max-size: "10m"
        max-file: "3"
    # docker-compose만 사용할 때 예시
    deploy:
      resources:
        limits:
          cpus: "1.0"
          memory: "512M"
```

> 확장 모니터링: Prometheus + cAdvisor/Node Exporter, Sentry(애플리케이션 오류 추적) 연계.

---

## CI/CD 파이프라인(요점)

- `.env.prod`는 **CI Secret/Protected Var**로 관리
- 배포 전 `docker compose --env-file .env.prod config`로 **치환 검증**
- 이미지 태깅: `org/app:${GIT_SHA}` + `org/app:latest`
- 서버 측: `docker compose pull && docker compose --env-file .env.prod up -d`

GitHub Actions 예시 스텝(요약):
```yaml
- name: Build & Push
  run: |
    docker build -t ghcr.io/your/app:${GITHUB_SHA} .
    echo $CR_PAT | docker login ghcr.io -u USERNAME --password-stdin
    docker push ghcr.io/your/app:${GITHUB_SHA}

- name: Deploy (SSH)
  run: |
    ssh user@host "cd /srv/community && \
      docker compose pull && \
      docker compose --env-file .env.prod up -d --remove-orphans"
```

---

## 트러블슈팅 체크리스트

| 증상 | 점검 포인트 | 빠른 확인 |
|---|---|---|
| 웹 502/504 | Nginx→WSGI 업스트림 연결, 포트/헬스 | `docker compose logs nginx web` |
| DB 연결 실패 | `.env` 값(호스트/포트/패스워드), `wait-for-db` 실행 여부 | `pg_isready`/`mysqladmin ping` |
| 마이그 실패 | 마이그레이션 실행 순서, 권한 | `migrate` 로그/권한 |
| 정적 파일 미표시(Django) | `collectstatic` 미실행, Nginx 경로/권한 | `ls staticfiles`, Nginx conf |
| 변경 미반영(개발) | 바인드 마운트 경로, Flask/Django reload | `volumes`와 `DEBUG/FLASK_ENV` |

---

## 실행 순서(개발/운영)

### 개발(Flask)

```bash
docker compose up -d --build
# 초기 테이블 생성

docker compose exec web python -c "from app import create_app; from app.models import db; app=create_app(); from app.models import Post; \
                                   from flask import current_app; \
                                   ctx=app.app_context(); ctx.push(); db.create_all(); print('tables created')"
# 동작 확인

curl http://localhost:5000/
```

### 운영

```bash
docker compose -f docker-compose.yml -f docker-compose.prod.yml --env-file .env.prod up -d --build
# 최초 배포시

docker compose exec web bash -lc "python manage.py migrate && python manage.py collectstatic --noinput"
```

---

## 보안·권한·베스트 프랙티스

- 컨테이너 실행 사용자: 필요 시 `USER` 추가로 루트 회피
- 비밀 관리: 환경변수 대신 **Compose secrets** 또는 외부 Vault 권장
- DB 네트워크: 내부 브리지 네트워크만 노출, 외부 포트 미노출
- 자동화: `Makefile`/스크립트 제공 → 팀 표준 운영

예시 Makefile:
```makefile
up:
	docker compose up -d

up-prod:
	docker compose -f docker-compose.yml -f docker-compose.prod.yml --env-file .env.prod up -d

logs:
	docker compose logs -f

down:
	docker compose down
```

---

## 결론

- **하나의 이미지**로 개발/운영을 공통화하고, **Compose의 .env/프로파일/헬스체크**를 조합하면 **안정적이고 재현 가능한 스택**을 구성할 수 있다.
- 운영 시에는 **Gunicorn + Nginx + 헬스체크 + 백업/모니터링/리소스 제한**까지 포함해 **실서비스 요건**을 충족하라.
- 스키마 변경과 배포를 반복하는 커뮤니티 서비스 특성상, **마이그레이션(Alembic/Django migrate)** 체계화가 유지보수의 핵심이다.

이 문서의 예제를 그대로 따라 하면, Flask든 Django든 **Postgres(or MySQL) 기반 커뮤니티 사이트**를 컨테이너로 빠르게 띄우고 운영까지 확장할 수 있다.
