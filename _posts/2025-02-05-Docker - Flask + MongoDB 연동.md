---
layout: post
title: Docker - Docker Compose 예제(Flask + MongoDB 연동)
date: 2025-02-05 19:20:23 +0900
category: Docker
---
# Docker Compose 예제: Flask + MongoDB 연동

## 1. 목표 및 시나리오

- Flask 앱이 **MongoDB에 접속**하여 문서를 **삽입/조회**한다.
- Docker Compose로 **두 컨테이너(web, mongo)** 를 올리고 네트워크를 묶는다.
- 기동 시 **MongoDB 준비 완료(healthcheck)** 를 확인하고 Flask가 시작된다.
- 첫 기동 시 **컬렉션 인덱스/검증 규칙(schema validation)/시드 데이터**가 들어간다.
- 개발/운영 차이를 **`.env`+`docker-compose.override.yml`** 로 분리한다.

---

## 2. 프로젝트 구조

```
flask-mongo-app/
├─ app/
│  ├─ app.py
│  ├─ requirements.txt
│  └─ init_db.py              # (선택) 앱 내 초기화 스크립트
├─ mongo-init/
│  ├─ 01-create-user.js       # 초기 사용자/DB 생성
│  └─ 02-seed-data.js         # 시드 데이터
├─ Dockerfile
├─ docker-compose.yml
├─ docker-compose.override.yml
├─ .env
└─ Makefile                   # (선택) 편의 스크립트
```

> `mongo-init`는 MongoDB 공식 이미지의 `/docker-entrypoint-initdb.d` 메커니즘을 사용합니다.

---

## 3. Dockerfile(Flask 이미지)

- 캐시 효율을 높이고, 런타임 이미지를 슬림하게 유지합니다.
- 비루트 사용자로 실행해 **기초 보안**을 확보합니다.

```Dockerfile
FROM python:3.11-slim

# 필수 OS 패키지(런타임 최소화)
RUN apt-get update && apt-get install -y --no-install-recommends \
    curl ca-certificates tzdata \
 && rm -rf /var/lib/apt/lists/*

# 앱 사용자 생성(보안상 root 회피)
RUN useradd -m -u 10001 appuser

WORKDIR /app

# 의존성 먼저 복사 → 캐시 효율
COPY app/requirements.txt .
RUN pip install --no-cache-dir -r requirements.txt

# 앱 복사
COPY app/ .

# 포트 및 헬스엔드포인트를 위한 기본 설정
ENV FLASK_RUN_HOST=0.0.0.0 \
    FLASK_RUN_PORT=5000

# 비루트로 실행
USER appuser

EXPOSE 5000
CMD ["python", "app.py"]
```

---

## 4. Flask 앱 코드(app/app.py)

- `MONGO_URI` 환경변수를 읽어 접속
- 기동 시 ping으로 연결 확인 + 인덱스 보증
- 간단한 **스키마 검증**(애플리케이션 레벨) + **컬렉션 validation**(데이터베이스 레벨)
- `GET /healthz` 헬스엔드포인트 제공

```python
# app/app.py
from flask import Flask, jsonify, request
from pymongo import MongoClient, ASCENDING
from pymongo.errors import ServerSelectionTimeoutError
import os, time

app = Flask(__name__)

MONGO_URI = os.environ.get("MONGO_URI", "mongodb://localhost:27017/mydb")
CONNECT_TIMEOUT_MS = int(os.environ.get("CONNECT_TIMEOUT_MS", "5000"))
READ_TIMEOUT_MS = int(os.environ.get("READ_TIMEOUT_MS", "5000"))

# 연결에 재시도 로직(초기 Mongo 준비 시간 고려)
def create_mongo_client(uri, retries=10, wait_sec=2):
    for i in range(retries):
        try:
            client = MongoClient(
                uri,
                serverSelectionTimeoutMS=CONNECT_TIMEOUT_MS,
                socketTimeoutMS=READ_TIMEOUT_MS
            )
            client.admin.command("ping")
            return client
        except ServerSelectionTimeoutError as e:
            print(f"[mongo] ping 실패({i+1}/{retries})... {e}")
            time.sleep(wait_sec)
    raise RuntimeError("MongoDB 접속 실패: 재시도 초과")

client = create_mongo_client(MONGO_URI)
db = client.get_database()  # URI의 마지막 DB명 사용

# 컬렉션 참조
items = db.get_collection("items")

# 컬렉션 validation(처음에만 적용됨; 이미 있으면 skip)
# MongoDB 6.x에서 validator를 설정해 간단한 스키마 제약
try:
    db.create_collection("items", validator={
        "$jsonSchema": {
            "bsonType": "object",
            "required": ["name", "price"],
            "properties": {
                "name": {"bsonType": "string"},
                "price": {"bsonType": ["int", "double"], "minimum": 0}
            }
        }
    })
except Exception:
    # 이미 있으면 무시
    pass

# 인덱스 보증
items.create_index([("name", ASCENDING)], name="idx_name")

@app.get("/healthz")
def healthz():
    try:
        client.admin.command("ping")
        return jsonify(status="ok"), 200
    except Exception as e:
        return jsonify(status="fail", detail=str(e)), 500

@app.get("/")
def index():
    return jsonify({"message": "Flask + MongoDB 연결 성공"})

@app.post("/add")
def add_data():
    data = request.get_json(force=True, silent=False)
    # 애플리케이션 단에서도 가벼운 밸리데이션
    if not isinstance(data, dict):
        return jsonify({"error": "JSON body required"}), 400
    name = data.get("name")
    price = data.get("price")
    if not isinstance(name, str) or name.strip() == "":
        return jsonify({"error": "name(string) required"}), 400
    if not isinstance(price, (int, float)) or price < 0:
        return jsonify({"error": "price >= 0 required"}), 400

    items.insert_one({"name": name.strip(), "price": float(price)})
    return jsonify({"message": "데이터 추가 완료"})

@app.get("/items")
def get_items():
    docs = list(items.find({}, {"_id": 0}))
    return jsonify(docs)

if __name__ == "__main__":
    # 개발용: reloader off(컨테이너 2중 실행 방지)
    app.run(host=os.environ.get("FLASK_RUN_HOST", "0.0.0.0"),
            port=int(os.environ.get("FLASK_RUN_PORT", "5000")),
            debug=os.environ.get("FLASK_DEBUG", "false").lower() == "true")
```

---

## 5. Mongo 초기 스크립트(mongo-init)

- `/docker-entrypoint-initdb.d` 경로에 배치하면 **초기 기동 시 자동 실행**됩니다.
- 01: 사용자/DB/권한 생성
- 02: 샘플 데이터 시드

```javascript
// mongo-init/01-create-user.js
// 기본 root 없이도 사용 가능한 사용자 생성(파일은 최초 기동 시 1회 실행)
db = db.getSiblingDB('mydb');
db.createUser({
  user: "appuser",
  pwd:  "apppass",
  roles: [ { role: "readWrite", db: "mydb" } ]
});
```

```javascript
// mongo-init/02-seed-data.js
db = db.getSiblingDB('mydb');
db.items.insertMany([
  { name: "orange", price: 900 },
  { name: "banana", price: 700 }
]);
```

> 이미 초기화가 이루어진 볼륨에서는 **재실행되지 않습니다**.

---

## 6. requirements.txt

```text
Flask==2.3.3
pymongo==4.7.1
```

---

## 7. Compose(운영 가능한 버전): `docker-compose.yml`

- `healthcheck` 를 통해 Mongo 준비완료를 판단하고, `depends_on: condition: service_healthy` 로 Flask 기동 순서를 제어합니다.
- Mongo는 **초기 사용자/DB/시드**를 `mongo-init`로 주입합니다.
- 네트워크는 `backend` 하나로 단순화하고, 명시적으로 묶습니다.

```yaml
version: '3.9'

services:
  mongo:
    image: mongo:6
    container_name: mongodb
    command: ["--auth"]   # 인증 활성화(초기 스크립트에서 생성한 사용자 사용)
    environment:
      - MONGO_INITDB_ROOT_USERNAME=root
      - MONGO_INITDB_ROOT_PASSWORD=secretroot
    ports:
      - "27017:27017"     # 개발 편의용(운영에서 불필요하면 제거)
    volumes:
      - mongo-data:/data/db
      - ./mongo-init:/docker-entrypoint-initdb.d:ro
    healthcheck:
      # mongosh가 없을 수 있으므로 기본 이미지의 mongosh/ mongo 연결 확인
      test: ["CMD", "mongosh", "--quiet", "--eval", "db.adminCommand('ping').ok", "localhost:27017/admin"]
      interval: 5s
      timeout: 3s
      retries: 20
      start_period: 10s
    networks:
      - backend

  web:
    build: ./                      # 상위의 Dockerfile 사용
    ports:
      - "5000:5000"
    environment:
      # 앱은 root가 아닌 appuser 계정으로 접속(초기 스크립트 참조)
      - MONGO_URI=mongodb://appuser:apppass@mongo:27017/mydb?authSource=mydb
      - CONNECT_TIMEOUT_MS=5000
      - READ_TIMEOUT_MS=5000
      - FLASK_DEBUG=false
    depends_on:
      mongo:
        condition: service_healthy
    networks:
      - backend

volumes:
  mongo-data:

networks:
  backend:
```

> `--auth`를 활성화했으므로, Flask의 `MONGO_URI`는 **사용자/비밀번호+authSource** 를 포함해야 합니다.

---

## 8. 개발/운영 분리: `.env` + `docker-compose.override.yml`

### `.env`
```env
# 공통/개발 기본값
MONGO_HOST=mongo
MONGO_DB=mydb
MONGO_USER=appuser
MONGO_PASS=apppass

FLASK_DEBUG=true
```

### `docker-compose.override.yml` (개발 시 자동 병합)
- 개발에서는 소스코드를 **바인드 마운트**해 핫리로드(옵션)
- 운영에서는 볼륨/이미지만 사용

```yaml
version: '3.9'
services:
  web:
    environment:
      - FLASK_DEBUG=${FLASK_DEBUG}
      - MONGO_URI=mongodb://${MONGO_USER}:${MONGO_PASS}@${MONGO_HOST}:27017/${MONGO_DB}?authSource=${MONGO_DB}
    volumes:
      - ./app:/app  # 개발 편의용(코드 변경 즉시 반영)
```

---

## 9. 실행/중지/로그 ― Makefile(선택)

```Makefile
up:
\tdocker-compose up --build -d

down:
\tdocker-compose down

logs:
\tdocker-compose logs -f --tail=200

ps:
\tdocker-compose ps

seed:
\t# 컨테이너 안에서 HTTP 요청으로 시드 추가 예시
\tcurl -s -X POST http://localhost:5000/add -H 'Content-Type: application/json' -d '{"name":"apple","price":1000}' | jq
```

---

## 10. 기동 및 상태 확인

```bash
# 1. 최초 기동
docker-compose up --build -d

# 2. 상태
docker-compose ps

# 3. 로그 추적
docker-compose logs -f --tail=200
```

---

## 11. 테스트 시나리오

### 11.1 헬스체크/기본 연결
```bash
curl -s http://localhost:5000/healthz
# {"status":"ok"}
```

### 11.2 루트 엔드포인트
```bash
curl -s http://localhost:5000
# {"message":"Flask + MongoDB 연결 성공"}
```

### 11.3 추가/조회
```bash
# insert
curl -s -X POST http://localhost:5000/add \
  -H "Content-Type: application/json" \
  -d '{"name":"grape","price":1200}'

# find
curl -s http://localhost:5000/items
# [{"name":"orange","price":900},{"name":"banana","price":700},{"name":"grape","price":1200}, ...]
```

> 앱 레벨과 DB 레벨의 검증이 모두 동작하므로, 잘못된 필드는 400 또는 Mongo validation 에러가 납니다.

---

## 12. 보강 포인트(현업형)

### 12.1 접속 재시도/타임아웃
- 이미 코드에 **서버 선택 타임아웃**과 **재시도**를 넣었습니다.
- 대규모 트래픽 환경에서는 **커넥션 풀 크기**(e.g. `maxPoolSize`)를 조절하세요.

```python
MongoClient(MONGO_URI, maxPoolSize=50, minPoolSize=5, serverSelectionTimeoutMS=5000)
```

### 12.2 인덱스/스키마 전략
- 자주 조회하는 필드에 **복합 인덱스** 설계 (예: `name + price`)
- 컬렉션 validation은 **강한 스키마 보증**이 필요할 때만(성능·유연성 트레이드오프)

### 12.3 운영 보안
- 운영에서는 DB 포트 외부 공개(`27017:27017`)를 제거하고, **내부 네트워크에서만** 접근.
- 비밀값은 `.env` 대신 **Docker secrets**(Swarm) 또는 **외부 비밀관리**(Vault/Parameter Store)를 고려.
- Flask 컨테이너는 `USER appuser`로 실행했고, 루트 권한을 피함.

### 12.4 백업/복구
- `mongo-data` 볼륨 스냅샷 또는 `mongodump/mongorestore` 파이프라인을 주기화.
- 운영 장애 대비: 데이터 파일 손상 시 복구 가이드라인과 RPO/RTO 정의.

### 12.5 트랜잭션(선택)
- 단일 노드에서는 **레플리카셋**이 아니면 다문서 트랜잭션이 제한됩니다. 필요 시 **single-node replica set** 구성 후 앱에서 세션/트랜잭션 사용.

---

## 13. 트러블슈팅

| 증상 | 원인 후보 | 해결 |
|------|----------|------|
| Flask가 시작했지만 `/healthz`가 실패 | Mongo 준비 미완료 | Compose의 healthcheck/depends_on 구성 확인, Flask 재시도 로직/타임아웃 상향 |
| 초기 스크립트가 재실행되지 않음 | 엔진 동작 특성(최초 1회) | 새 볼륨으로 재기동하거나 수동으로 적용 |
| 인증 실패 | URI/사용자/DB/`authSource` 불일치 | `mongodb://user:pass@host:port/db?authSource=db` 재확인 |
| Docker Desktop에서 포트 충돌 | 다른 프로세스 사용 중 | `lsof -i :27017` 등으로 충돌 확인, 포트 변경 |
| 느린 응답 | 인덱스 부재/풀 과소 | 인덱스 설계/`maxPoolSize` 조정, 도커 리소스 할당 상향 |

---

## 14. 대체/확장 아이디어

- **FastAPI 변환**: Uvicorn+Gunicorn 다중 워커로 처리량 향상
- **Mongo Express UI**:
  ```yaml
  mongo-express:
    image: mongo-express
    ports: ["8081:8081"]
    environment:
      - ME_CONFIG_MONGODB_ADMINUSERNAME=root
      - ME_CONFIG_MONGODB_ADMINPASSWORD=secretroot
      - ME_CONFIG_MONGODB_SERVER=mongo
    depends_on:
      mongo:
        condition: service_healthy
    networks: [backend]
  ```
- **테스트 자동화**: `pytest`로 컨테이너 기동 후 e2e 검증
- **프로파일링**: `Flask` → `Gunicorn` + 프런트 리버스 프록시(`nginx`) 구조

---

## 15. 원문 대비 핵심 확장 요약

- **헬스체크/depends_on 조건**으로 **안정적 기동 순서** 확보
- **초기 스크립트**(사용자/시드) 자동화로 재현성 보장
- **비루트 사용자**로 컨테이너 실행(보안)
- `.env`+`override` 로 **개발/운영 분리**
- 커넥션 풀/타임아웃/인덱스/밸리데이션 등 **운영형 튜닝 포인트** 제시

---

## 부록: 원형(간소 버전)도 유지하고 싶다면

아래는 최소 구성(초기 제공 예제 스타일)을 유지한 채, URI만 인증형으로 수정한 버전입니다.

```yaml
version: '3.9'
services:
  web:
    build: ./app
    ports: ["5000:5000"]
    environment:
      - MONGO_URI=mongodb://appuser:apppass@mongo:27017/mydb?authSource=mydb
    depends_on:
      - mongo
    networks: [backend]

  mongo:
    image: mongo:6
    container_name: mongodb
    ports: ["27017:27017"]
    volumes:
      - mongo-data:/data/db
      - ./mongo-init:/docker-entrypoint-initdb.d:ro
    networks: [backend]

volumes:
  mongo-data:

networks:
  backend:
```

---

## 참고

- MongoDB Docker Hub: https://hub.docker.com/_/mongo
- Flask: https://flask.palletsprojects.com/
- PyMongo: https://pymongo.readthedocs.io/
- Docker Compose: https://docs.docker.com/compose/
