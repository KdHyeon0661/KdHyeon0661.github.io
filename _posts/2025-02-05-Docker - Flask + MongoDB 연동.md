---
layout: post
title: Docker - Docker Compose 예제: Flask + MongoDB 연동
date: 2025-02-05 19:20:23 +0900
category: Docker
---
# 🛠️ Docker Compose 예제 : Flask + MongoDB 연동

---

## 📌 목표

- Flask 앱에서 MongoDB에 데이터 삽입 및 조회
- Docker Compose로 두 컨테이너를 구성 및 연결
- Flask → MongoDB 접속 확인

---

## 📁 프로젝트 구조

```plaintext
flask-mongo-app/
├── app/
│   ├── app.py
│   └── requirements.txt
├── docker-compose.yml
├── Dockerfile
```

---

## 🐳 docker-compose.yml

```yaml
version: '3.9'

services:
  web:
    build: ./app
    ports:
      - "5000:5000"
    environment:
      - MONGO_URI=mongodb://mongo:27017/mydb
    depends_on:
      - mongo
    networks:
      - backend

  mongo:
    image: mongo:6
    container_name: mongodb
    ports:
      - "27017:27017"
    volumes:
      - mongo-data:/data/db
    networks:
      - backend

volumes:
  mongo-data:

networks:
  backend:
```

### 💡 포인트
- `mongo` 서비스는 MongoDB 공식 이미지 사용
- Flask는 `MONGO_URI` 환경변수를 통해 MongoDB 접속
- 두 컨테이너는 `backend` 네트워크로 묶임
- `mongo`라는 컨테이너 이름으로 접근 가능 (`mongo:27017`)

---

## 🐍 app/requirements.txt

```text
Flask==2.3.3
pymongo==4.7.1
```

---

## 🐳 Dockerfile

```Dockerfile
FROM python:3.11-slim

WORKDIR /app

COPY requirements.txt .
RUN pip install --no-cache-dir -r requirements.txt

COPY . .

CMD ["python", "app.py"]
```

---

## 🐍 app/app.py (Flask 코드)

```python
from flask import Flask, jsonify, request
from pymongo import MongoClient
import os

app = Flask(__name__)

# MongoDB 연결
mongo_uri = os.environ.get('MONGO_URI', 'mongodb://localhost:27017/mydb')
client = MongoClient(mongo_uri)
db = client.get_database()

@app.route('/')
def index():
    return jsonify({"message": "Flask + MongoDB 연결 성공!"})

@app.route('/add', methods=['POST'])
def add_data():
    data = request.json
    db.items.insert_one(data)
    return jsonify({"message": "데이터 추가 완료!"})

@app.route('/items', methods=['GET'])
def get_items():
    items = list(db.items.find({}, {'_id': 0}))
    return jsonify(items)

if __name__ == '__main__':
    app.run(host='0.0.0.0', port=5000)
```

---

## 🚀 실행 방법

```bash
# 1. 현재 디렉터리로 이동
cd flask-mongo-app

# 2. 컨테이너 실행
docker-compose up --build -d

# 3. 상태 확인
docker-compose ps
```

---

## 🔍 테스트 방법

### 1. 접속 확인

```bash
curl http://localhost:5000
# → {"message":"Flask + MongoDB 연결 성공!"}
```

### 2. 데이터 추가

```bash
curl -X POST http://localhost:5000/add \
  -H "Content-Type: application/json" \
  -d '{"name": "apple", "price": 1000}'
```

### 3. 데이터 조회

```bash
curl http://localhost:5000/items
# → [{"name": "apple", "price": 1000}]
```

---

## ✅ 주요 포인트 요약

| 구성 요소 | 설명 |
|-----------|------|
| Flask | Python 웹 서버 |
| pymongo | MongoDB 드라이버 |
| mongo | NoSQL DB 서버 |
| depends_on | MongoDB가 먼저 실행되도록 보장 |
| 환경변수 | Flask에서 Mongo URI 전달 |
| 볼륨 | Mongo 데이터 영속화 |

---

## 🧪 확장 아이디어

- Flask → FastAPI로 전환
- MongoDB 초기 데이터 seed 삽입
- Mongo Express UI 연동
- `docker-compose.override.yml`로 개발/운영 분리

---

## 📚 참고 자료

- [MongoDB 공식 Docker 이미지](https://hub.docker.com/_/mongo)
- [Flask 공식 문서](https://flask.palletsprojects.com/)
- [pymongo 공식 문서](https://pymongo.readthedocs.io/)
- [Compose 공식 문서](https://docs.docker.com/compose/)