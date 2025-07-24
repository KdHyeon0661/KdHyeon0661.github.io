---
layout: post
title: Docker - Flask & Django 기반 커뮤니티 사이트 Docker 컨테이너화
date: 2025-02-27 19:20:23 +0900
category: Docker
---
# 🌐 Flask/Django 기반 커뮤니티 사이트의 Docker 컨테이너화

---

## 📌 목표

- Flask 또는 Django 웹 애플리케이션 컨테이너화
- PostgreSQL/MySQL 데이터베이스 연동
- 개발과 운영 환경을 Docker Compose로 통합
- 볼륨, 네트워크, 마이그레이션 등 통합 구성

---

## 📁 디렉터리 구조 예시 (Flask 기준)

```plaintext
community-app/
├── app/
│   ├── __init__.py
│   ├── models.py
│   ├── views.py
│   └── templates/
├── requirements.txt
├── Dockerfile
├── docker-compose.yml
└── .env
```

---

## 🐳 Dockerfile (공통 - Flask 또는 Django)

```Dockerfile
FROM python:3.11-slim

WORKDIR /app

COPY requirements.txt .
RUN pip install --no-cache-dir -r requirements.txt

COPY . .

CMD ["python", "-m", "flask", "run", "--host=0.0.0.0"]
```

> Django의 경우 `CMD`만 `python manage.py runserver 0.0.0.0:8000` 등으로 변경

---

## 📝 requirements.txt 예시

**Flask용**

```
Flask==2.3.3
Flask-SQLAlchemy
psycopg2-binary
```

**Django용**

```
Django==4.2
psycopg2-binary
```

---

## ⚙️ docker-compose.yml (Flask 예시)

```yaml
version: '3.9'

services:
  web:
    build: .
    ports:
      - "5000:5000"
    env_file:
      - .env
    volumes:
      - .:/app
    depends_on:
      - db
    networks:
      - app-net

  db:
    image: postgres:15
    environment:
      POSTGRES_DB: ${POSTGRES_DB}
      POSTGRES_USER: ${POSTGRES_USER}
      POSTGRES_PASSWORD: ${POSTGRES_PASSWORD}
    volumes:
      - db-data:/var/lib/postgresql/data
    networks:
      - app-net

volumes:
  db-data:

networks:
  app-net:
```

---

## 🔐 .env 파일 예시

```
POSTGRES_DB=community_db
POSTGRES_USER=admin
POSTGRES_PASSWORD=securepass
FLASK_APP=app
FLASK_ENV=development
```

---

## 🛠️ Flask 예제: 모델 + 뷰

### app/models.py

```python
from flask_sqlalchemy import SQLAlchemy
from flask import current_app

db = SQLAlchemy()

class Post(db.Model):
    id = db.Column(db.Integer, primary_key=True)
    title = db.Column(db.String(120))
    content = db.Column(db.Text)
```

### app/__init__.py

```python
from flask import Flask
from .models import db

def create_app():
    app = Flask(__name__)
    app.config['SQLALCHEMY_DATABASE_URI'] = 'postgresql://admin:securepass@db:5432/community_db'
    app.config['SQLALCHEMY_TRACK_MODIFICATIONS'] = False
    db.init_app(app)
    return app
```

---

## 🐍 마이그레이션 및 초기화 (Flask)

```bash
docker-compose run web python
>>> from app import create_app
>>> from app.models import db
>>> app = create_app()
>>> with app.app_context():
...     db.create_all()
```

---

## 🌐 Django 버전은 다음과 같이

- `python manage.py startproject community`
- `python manage.py migrate`
- `python manage.py createsuperuser`
- PostgreSQL 연동: `settings.py`에서 DB 설정 수정

```python
DATABASES = {
  'default': {
    'ENGINE': 'django.db.backends.postgresql',
    'NAME': os.getenv("POSTGRES_DB"),
    'USER': os.getenv("POSTGRES_USER"),
    'PASSWORD': os.getenv("POSTGRES_PASSWORD"),
    'HOST': 'db',
    'PORT': 5432,
  }
}
```

---

## ✅ 실제 운영을 위한 고려 사항

| 항목 | 설명 |
|------|------|
| Gunicorn | Flask/Django용 WSGI 서버로 배포 (CMD 대체) |
| nginx + certbot | HTTPS 적용 |
| static/media | 볼륨으로 따로 관리 |
| 환경변수 | `.env`로 민감 정보 분리 |
| healthcheck | DB 및 앱 상태 확인 |
| CI/CD | GitHub Actions 또는 GitLab CI 연동 |

---

## 🧪 개발 중 실시간 반영을 위한 설정

```yaml
volumes:
  - .:/app
```

- 코드 변경 시 컨테이너 재시작 없이 반영
- Django는 `runserver --reload`, Flask는 `FLASK_ENV=development` 필요

---

## 📚 참고 자료

- [Flask 공식 문서](https://flask.palletsprojects.com/)
- [Django 공식 문서](https://docs.djangoproject.com/)
- [Dockerizing Django with PostgreSQL](https://testdriven.io/blog/dockerizing-django-with-postgres/)
- [Docker Compose 문서](https://docs.docker.com/compose/)