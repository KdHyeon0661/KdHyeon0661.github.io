---
layout: post
title: Docker - Docker Compose 예제(Flask + MongoDB 연동)
date: 2025-02-05 19:20:23 +0900
category: Docker
---
# Flask와 MongoDB를 Docker Compose로 연동하는 실전 가이드

이번 가이드에서는 Flask 웹 애플리케이션과 MongoDB 데이터베이스를 Docker Compose로 효과적으로 연동하는 방법을 단계별로 살펴보겠습니다. 실제 운영 환경에서 요구되는 보안, 성능, 안정성 요소를 모두 고려한 실용적인 예제를 중심으로 설명합니다.

## 프로젝트 개요와 목표

이 프로젝트의 목표는 Flask 애플리케이션이 MongoDB와 안정적으로 통신하는 완전한 컨테이너 기반 환경을 구축하는 것입니다. 주요 목표는 다음과 같습니다:

1. Flask 애플리케이션이 MongoDB에 연결하여 데이터를 생성하고 조회할 수 있도록 합니다.
2. Docker Compose를 사용하여 웹 서비스와 데이터베이스 서비스를 함께 관리합니다.
3. 애플리케이션 시작 시 MongoDB가 완전히 준비된 상태인지 확인한 후 Flask가 시작되도록 보장합니다.
4. 초기 데이터베이스 설정(사용자 생성, 시드 데이터 삽입)을 자동화합니다.
5. 개발 환경과 운영 환경의 차이점을 명확히 분리하여 관리합니다.

## 프로젝트 구조 설계

프로젝트는 다음과 같은 구조로 구성됩니다:

```
flask-mongo-app/
├── app/                    # Flask 애플리케이션 소스 코드
│   ├── app.py             # 메인 애플리케이션 파일
│   ├── requirements.txt   # Python 의존성 목록
│   └── init_db.py        # 데이터베이스 초기화 스크립트 (선택 사항)
├── mongo-init/            # MongoDB 초기화 스크립트
│   ├── 01-create-user.js # 데이터베이스 사용자 생성
│   └── 02-seed-data.js   # 초기 데이터 삽입
├── Dockerfile             # Flask 애플리케이션 빌드 정의
├── docker-compose.yml     # 기본 Docker Compose 구성
├── docker-compose.override.yml # 개발 환경 오버라이드
├── .env                   # 환경 변수 설정
└── Makefile               # 자주 사용하는 명령어 모음 (선택 사항)
```

이 구조는 관심사의 분리 원칙을 따르며, 각 구성 요소의 역할이 명확하게 구분됩니다.

## Flask 애플리케이션 개발

### Dockerfile 작성

Flask 애플리케이션을 위한 Dockerfile은 성능과 보안을 모두 고려하여 작성합니다:

```dockerfile
FROM python:3.11-slim

# 필수 시스템 패키지 설치 (최소한의 런타임만 포함)
RUN apt-get update && apt-get install -y --no-install-recommends \
    curl ca-certificates tzdata \
 && rm -rf /var/lib/apt/lists/*

# 보안을 위한 비루트 사용자 생성
RUN useradd -m -u 10001 appuser

WORKDIR /app

# 의존성 파일 먼저 복사하여 캐시 효율성 향상
COPY app/requirements.txt .
RUN pip install --no-cache-dir -r requirements.txt

# 애플리케이션 코드 복사
COPY app/ .

# 환경 변수 설정
ENV FLASK_RUN_HOST=0.0.0.0 \
    FLASK_RUN_PORT=5000

# 비루트 사용자로 전환하여 보안 강화
USER appuser

EXPOSE 5000
CMD ["python", "app.py"]
```

이 Dockerfile의 핵심 특징은 다음과 같습니다:
- 캐시 효율성을 위해 의존성 파일을 먼저 복사하고 설치합니다.
- 보안을 위해 비루트 사용자를 생성하고 애플리케이션을 해당 사용자로 실행합니다.
- 불필요한 시스템 패키지를 제거하여 이미지 크기를 최소화합니다.

### Flask 애플리케이션 구현

Flask 애플리케이션은 MongoDB와의 안정적인 연결, 데이터 유효성 검증, 헬스체크 기능을 포함합니다:

```python
from flask import Flask, jsonify, request
from pymongo import MongoClient, ASCENDING
from pymongo.errors import ServerSelectionTimeoutError
import os
import time

app = Flask(__name__)

# 환경 변수에서 MongoDB 연결 설정 읽기
MONGO_URI = os.environ.get("MONGO_URI", "mongodb://localhost:27017/mydb")
CONNECT_TIMEOUT_MS = int(os.environ.get("CONNECT_TIMEOUT_MS", "5000"))
READ_TIMEOUT_MS = int(os.environ.get("READ_TIMEOUT_MS", "5000"))

def create_mongo_client(uri, retries=10, wait_sec=2):
    """
    MongoDB 클라이언트 생성 시 재시도 로직을 포함한 함수
    컨테이너 시작 시 MongoDB가 준비되기까지 시간이 걸릴 수 있으므로
    여러 번 재시도하여 연결 안정성을 보장합니다.
    """
    for i in range(retries):
        try:
            client = MongoClient(
                uri,
                serverSelectionTimeoutMS=CONNECT_TIMEOUT_MS,
                socketTimeoutMS=READ_TIMEOUT_MS
            )
            # 연결 테스트
            client.admin.command("ping")
            print(f"MongoDB 연결 성공 ({i+1}번째 시도)")
            return client
        except ServerSelectionTimeoutError as e:
            print(f"MongoDB 연결 실패 ({i+1}/{retries}): {e}")
            time.sleep(wait_sec)
    
    raise RuntimeError("MongoDB 연결 실패: 최대 재시도 횟수 초과")

# MongoDB 클라이언트 생성
client = create_mongo_client(MONGO_URI)
db = client.get_database()  # URI에서 지정된 데이터베이스 사용

# 컬렉션 참조
items_collection = db.get_collection("items")

# 컬렉션 유효성 검사기 설정 (MongoDB 6.0 이상)
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
    print("컬렉션 유효성 검사기 설정 완료")
except Exception as e:
    # 컬렉션이 이미 존재하는 경우 무시
    print(f"컬렉션 설정 (무시됨): {e}")

# 검색 성능을 위한 인덱스 생성
items_collection.create_index([("name", ASCENDING)], name="idx_name")
print("이름 필드 인덱스 생성 완료")

@app.get("/healthz")
def health_check():
    """
    애플리케이션 상태 확인을 위한 헬스체크 엔드포인트
    MongoDB 연결 상태를 포함한 전체 시스템 상태를 확인합니다.
    """
    try:
        client.admin.command("ping")
        return jsonify({"status": "healthy", "message": "서버와 데이터베이스 정상"}), 200
    except Exception as e:
        return jsonify({"status": "unhealthy", "error": str(e)}), 500

@app.get("/")
def index():
    """기본 정보 페이지"""
    return jsonify({
        "message": "Flask와 MongoDB 연동 성공",
        "database": db.name,
        "collection": "items"
    })

@app.post("/items")
def add_item():
    """
    새 항목 추가 엔드포인트
    클라이언트로부터 JSON 데이터를 받아 MongoDB에 저장합니다.
    """
    try:
        data = request.get_json(force=True, silent=False)
        
        # 입력 데이터 유효성 검사
        if not isinstance(data, dict):
            return jsonify({"error": "JSON 객체가 필요합니다"}), 400
        
        name = data.get("name")
        price = data.get("price")
        
        # 필수 필드 검증
        if not isinstance(name, str) or not name.strip():
            return jsonify({"error": "유효한 이름(문자열)이 필요합니다"}), 400
        
        if not isinstance(price, (int, float)) or price < 0:
            return jsonify({"error": "가격은 0 이상의 숫자여야 합니다"}), 400
        
        # MongoDB에 데이터 저장
        result = items_collection.insert_one({
            "name": name.strip(),
            "price": float(price),
            "created_at": time.time()
        })
        
        return jsonify({
            "message": "항목이 성공적으로 추가되었습니다",
            "id": str(result.inserted_id)
        }), 201
        
    except Exception as e:
        return jsonify({"error": f"서버 오류: {str(e)}"}), 500

@app.get("/items")
def get_items():
    """
    모든 항목 조회 엔드포인트
    MongoDB에서 저장된 모든 항목을 조회합니다.
    """
    try:
        # MongoDB에서 데이터 조회 (_id 필드는 제외)
        items = list(items_collection.find({}, {"_id": 0}))
        return jsonify({"items": items, "count": len(items)})
    except Exception as e:
        return jsonify({"error": f"데이터 조회 실패: {str(e)}"}), 500

if __name__ == "__main__":
    # 개발 환경 설정
    debug_mode = os.environ.get("FLASK_DEBUG", "false").lower() == "true"
    
    app.run(
        host=os.environ.get("FLASK_RUN_HOST", "0.0.0.0"),
        port=int(os.environ.get("FLASK_RUN_PORT", "5000")),
        debug=debug_mode
    )
```

이 Flask 애플리케이션의 주요 특징은 다음과 같습니다:

1. **연결 안정성**: MongoDB 연결 시 재시도 로직을 포함하여 컨테이너 시작 시 발생할 수 있는 타이밍 문제를 해결합니다.
2. **데이터 무결성**: 애플리케이션 수준과 데이터베이스 수준에서 모두 데이터 유효성 검증을 수행합니다.
3. **상태 모니터링**: `/healthz` 엔드포인트를 통해 애플리케이션과 데이터베이스의 상태를 확인할 수 있습니다.
4. **오류 처리**: 모든 예외 상황에 대해 적절한 오류 응답을 반환합니다.

### Python 의존성 관리

`requirements.txt` 파일은 프로젝트의 Python 패키지 의존성을 명시합니다:

```
Flask==2.3.3
pymongo==4.7.1
```

## MongoDB 초기화 설정

MongoDB 컨테이너가 처음 시작될 때 실행될 초기화 스크립트를 준비합니다.

### 데이터베이스 사용자 생성 스크립트

```javascript
// mongo-init/01-create-user.js
// MongoDB 초기 사용자 및 데이터베이스 생성

// 애플리케이션 데이터베이스 생성 및 사용자 설정
db = db.getSiblingDB('mydb');

// 애플리케이션 전용 사용자 생성
db.createUser({
  user: "appuser",
  pwd: "apppass",
  roles: [
    { role: "readWrite", db: "mydb" },
    { role: "dbAdmin", db: "mydb" }
  ]
});

print("애플리케이션 사용자 생성 완료");
```

### 초기 데이터 삽입 스크립트

```javascript
// mongo-init/02-seed-data.js
// 초기 테스트 데이터 삽입

db = db.getSiblingDB('mydb');

// 초기 데이터 삽입 (이미 존재하는 경우 중복 방지)
if (db.items.countDocuments() === 0) {
  db.items.insertMany([
    { 
      name: "orange", 
      price: 900,
      category: "fruit",
      created_at: new Date()
    },
    { 
      name: "banana", 
      price: 700,
      category: "fruit", 
      created_at: new Date()
    },
    { 
      name: "apple", 
      price: 1200,
      category: "fruit",
      created_at: new Date()
    }
  ]);
  print("초기 데이터 삽입 완료");
} else {
  print("데이터가 이미 존재합니다. 초기 데이터를 건너뜁니다.");
}
```

이 초기화 스크립트들은 MongoDB 공식 이미지의 `/docker-entrypoint-initdb.d` 디렉토리 메커니즘을 활용합니다. 이 디렉토리에 `.js` 또는 `.sh` 파일을 배치하면 MongoDB가 처음 시작될 때 자동으로 실행됩니다.

## Docker Compose 구성

### 기본 Docker Compose 구성 (docker-compose.yml)

이 파일은 프로덕션 환경에 적합한 기본 구성을 정의합니다:

```yaml
version: '3.9'

services:
  mongo:
    image: mongo:6
    container_name: mongodb
    command: ["--auth"]  # 인증 모드 활성화
    environment:
      - MONGO_INITDB_ROOT_USERNAME=root
      - MONGO_INITDB_ROOT_PASSWORD=secretroot
    ports:
      - "27017:27017"  # 개발 편의를 위한 포트 노출
    volumes:
      - mongo-data:/data/db
      - ./mongo-init:/docker-entrypoint-initdb.d:ro
    healthcheck:
      test: ["CMD", "mongosh", "--quiet", "--eval", "db.adminCommand('ping').ok", "localhost:27017/admin"]
      interval: 5s
      timeout: 3s
      retries: 20
      start_period: 10s
    networks:
      - backend

  web:
    build: .
    ports:
      - "5000:5000"
    environment:
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

이 구성의 주요 특징은 다음과 같습니다:

1. **서비스 의존성 관리**: Flask 애플리케이션이 MongoDB의 헬스체크를 통과한 후에만 시작되도록 보장합니다.
2. **데이터 영속성**: MongoDB 데이터를 호스트 볼륨에 저장하여 컨테이너 재시작 시에도 데이터가 유지됩니다.
3. **네트워크 격리**: 백엔드 네트워크를 사용하여 서비스 간 통신을 보호합니다.
4. **인증 강화**: MongoDB 인증 모드를 활성화하고 애플리케이션 전용 사용자를 사용합니다.

### 환경 변수 파일 (.env)

환경별 설정을 분리하기 위해 `.env` 파일을 사용합니다:

```env
# MongoDB 연결 설정
MONGO_HOST=mongo
MONGO_DB=mydb
MONGO_USER=appuser
MONGO_PASS=apppass

# Flask 애플리케이션 설정
FLASK_DEBUG=true
FLASK_RUN_HOST=0.0.0.0
FLASK_RUN_PORT=5000

# MongoDB 연결 타임아웃 설정
CONNECT_TIMEOUT_MS=5000
READ_TIMEOUT_MS=5000
```

### 개발 환경 오버라이드 (docker-compose.override.yml)

개발 환경에서는 추가적인 편의 기능이 필요합니다. `docker-compose.override.yml` 파일은 개발 시에만 적용되는 설정을 정의합니다:

```yaml
version: '3.9'
services:
  web:
    environment:
      - FLASK_DEBUG=${FLASK_DEBUG}
      - MONGO_URI=mongodb://${MONGO_USER}:${MONGO_PASS}@${MONGO_HOST}:27017/${MONGO_DB}?authSource=${MONGO_DB}
    volumes:
      - ./app:/app  # 코드 변경 즉시 반영을 위한 바인드 마운트
```

이 오버라이드 구성은 다음과 같은 개발자 편의 기능을 제공합니다:
- Flask 디버그 모드 활성화
- 환경 변수를 사용한 동적 MongoDB URI 구성
- 호스트의 소스 코드 디렉토리를 컨테이너에 마운트하여 코드 변경 시 즉시 반영

## 프로젝트 실행과 테스트

### Makefile을 통한 명령어 간소화

자주 사용하는 Docker Compose 명령어를 Makefile로 간소화할 수 있습니다:

```makefile
.PHONY: up down logs ps test clean

# 애플리케이션 시작
up:
	docker-compose up --build -d

# 애플리케이션 중지
down:
	docker-compose down

# 로그 확인
logs:
	docker-compose logs -f --tail=200

# 서비스 상태 확인
ps:
	docker-compose ps

# 테스트 데이터 추가
seed:
	@echo "테스트 데이터 추가..."
	@curl -s -X POST http://localhost:5000/items \
		-H 'Content-Type: application/json' \
		-d '{"name":"grape","price":1500}' | jq
	@curl -s -X POST http://localhost:5000/items \
		-H 'Content-Type: application/json' \
		-d '{"name":"watermelon","price":5000}' | jq

# 시스템 테스트
test:
	@echo "헬스체크 테스트..."
	@curl -s http://localhost:5000/healthz | jq
	@echo "\n기본 엔드포인트 테스트..."
	@curl -s http://localhost:5000 | jq
	@echo "\n데이터 조회 테스트..."
	@curl -s http://localhost:5000/items | jq '.count'

# 시스템 정리
clean:
	docker-compose down -v
	docker system prune -f
```

### 애플리케이션 실행

터미널에서 다음 명령어를 실행하여 애플리케이션을 시작합니다:

```bash
# 애플리케이션 빌드 및 시작
docker-compose up --build -d

# 서비스 상태 확인
docker-compose ps

# 로그 모니터링
docker-compose logs -f
```

### API 테스트

애플리케이션이 정상적으로 실행된 후, 다양한 API 엔드포인트를 테스트할 수 있습니다:

```bash
# 헬스체크 엔드포인트 테스트
curl http://localhost:5000/healthz

# 기본 정보 엔드포인트 테스트
curl http://localhost:5000/

# 데이터 추가 테스트
curl -X POST http://localhost:5000/items \
  -H "Content-Type: application/json" \
  -d '{"name":"strawberry","price":2000}'

# 데이터 조회 테스트
curl http://localhost:5000/items
```

## 운영 환경을 위한 고급 구성

### 연결 풀 최적화

프로덕션 환경에서는 MongoDB 연결 풀 설정을 최적화해야 합니다. Flask 애플리케이션의 MongoDB 클라이언트 생성 부분을 다음과 같이 수정할 수 있습니다:

```python
client = MongoClient(
    MONGO_URI,
    serverSelectionTimeoutMS=CONNECT_TIMEOUT_MS,
    socketTimeoutMS=READ_TIMEOUT_MS,
    maxPoolSize=50,        # 최대 연결 풀 크기
    minPoolSize=5,         # 최소 연결 풀 크기
    maxIdleTimeMS=30000,   # 유휴 연결 최대 시간
    waitQueueTimeoutMS=10000  # 연결 대기 시간 제한
)
```

### 복합 인덱스 설계

데이터베이스 성능을 최적화하기 위해 자주 함께 조회되는 필드에 복합 인덱스를 생성합니다:

```python
# 이름과 가격으로 검색하는 쿼리가 빈번한 경우
items_collection.create_index(
    [("name", ASCENDING), ("price", ASCENDING)],
    name="idx_name_price"
)

# 생성일자 기준 정렬이 필요한 경우
items_collection.create_index(
    [("created_at", DESCENDING)],
    name="idx_created_at_desc"
)
```

### 보안 강화

운영 환경에서는 다음과 같은 추가 보안 조치를 고려해야 합니다:

1. **포트 노출 제한**: MongoDB 포트(27017)를 외부에 노출하지 않고 내부 네트워크에서만 접근하도록 제한합니다.
2. **비밀 정보 관리**: `.env` 파일 대신 Docker Swarm의 Secrets나 외부 비밀 관리 시스템을 사용합니다.
3. **네트워크 격리**: 프로덕션 네트워크를 추가로 구성하여 서비스 간 통신을 제한합니다.

```yaml
# docker-compose.prod.yml (운영 환경 전용)
version: '3.9'

services:
  mongo:
    ports: []  # 외부 포트 노출 제거
    networks:
      - internal-network  # 내부 전용 네트워크

  web:
    networks:
      - internal-network
      - public-network  # 웹 트래픽을 위한 공개 네트워크

networks:
  internal-network:
    internal: true  # 내부 전용 네트워크
  public-network:
    # 공개 네트워크 구성
```

## 문제 해결 가이드

### 일반적인 문제와 해결 방법

**문제 1: Flask 애플리케이션이 MongoDB에 연결하지 못함**

증상: Flask 컨테이너 로그에 "MongoDB 연결 실패" 메시지가 반복적으로 표시됨

원인 및 해결:
- MongoDB가 아직 준비되지 않은 상태에서 Flask가 시작되었을 수 있습니다. `depends_on`의 `condition: service_healthy` 설정이 올바른지 확인하세요.
- MongoDB URI에 오타가 있을 수 있습니다. 환경 변수와 연결 문자열을 다시 확인하세요.
- 네트워크 설정 문제일 수 있습니다. `docker network inspect` 명령으로 네트워크 구성을 확인하세요.

**문제 2: 초기화 스크립트가 실행되지 않음**

증상: MongoDB 컨테이너는 시작되지만 초기 사용자나 데이터가 생성되지 않음

원인 및 해결:
- 볼륨이 이미 존재하는 경우 초기화 스크립트는 실행되지 않습니다. 새로운 볼륨을 사용하거나 기존 볼륨을 삭제한 후 다시 시작하세요.
- 스크립트 파일 권한 문제일 수 있습니다. 스크립트 파일이 실행 가능한지 확인하세요.
- MongoDB 로그에서 초기화 과정에 대한 메시지를 확인하세요.

**문제 3: 인증 오류 발생**

증상: "Authentication failed" 또는 유사한 오류 메시지

원인 및 해결:
- MongoDB URI의 사용자 이름, 비밀번호, 데이터베이스 이름이 일치하는지 확인하세요.
- `authSource` 매개변수가 올바른지 확인하세요. 일반적으로 인증 데이터베이스와 애플리케이션 데이터베이스가 동일해야 합니다.
- MongoDB 컨테이너가 `--auth` 옵션으로 시작되었는지 확인하세요.

### 진단 명령어

문제 해결을 위한 유용한 Docker 명령어:

```bash
# 모든 컨테이너 상태 확인
docker-compose ps

# 특정 서비스 로그 확인
docker-compose logs web
docker-compose logs mongo

# 컨테이너 내부에서 테스트
docker-compose exec web python -c "import app; print('모듈 로드 성공')"

# 네트워크 연결 테스트
docker-compose exec web curl -s http://mongo:27017

# 볼륨 상태 확인
docker volume ls
docker volume inspect flask-mongo-app_mongo-data

# 시스템 리소스 사용량 확인
docker stats
```

## 확장 가능성과 향후 개선 방향

### 애플리케이션 아키텍처 발전

현재의 단일 Flask 애플리케이션 구조에서 더 확장 가능한 아키텍처로 발전시킬 수 있습니다:

1. **마이크로서비스 분리**: 사용자 관리, 상품 관리, 주문 관리 등의 기능을 별도의 서비스로 분리
2. **API 게이트웨이 도입**: Kong 또는 Traefik과 같은 API 게이트웨이를 추가하여 라우팅, 인증, 로깅 통합
3. **메시지 큐 도입**: RabbitMQ 또는 Kafka를 사용하여 비동기 작업 처리

### 데이터베이스 고급 구성

프로덕션 환경을 위해 MongoDB 구성을 강화할 수 있습니다:

1. **복제본 세트 구성**: 데이터 가용성과 읽기 성능 향상을 위해 복제본 세트 구성
2. **샤딩 구성**: 대용량 데이터 처리를 위한 수평적 확장
3. **백업 자동화**: 정기적인 백업 및 복구 프로세스 구현

### 모니터링과 로깅

운영 가시성을 높이기 위한 모니터링 시스템 구축:

1. **메트릭 수집**: Prometheus를 사용하여 애플리케이션 및 데이터베이스 메트릭 수집
2. **분산 로깅**: ELK 스택 또는 Loki를 사용하여 중앙화된 로그 관리
3. **분산 추적**: Jaeger 또는 Zipkin을 사용하여 요청 흐름 추적

## 결론

이 가이드에서는 Flask와 MongoDB를 Docker Compose로 효과적으로 연동하는 방법을 상세히 살펴보았습니다. 핵심적인 학습 내용을 정리하면 다음과 같습니다:

**컨테이너 기반 개발의 장점**: Docker Compose를 사용하면 복잡한 다중 컨테이너 환경도 단일 명령어로 관리할 수 있습니다. 개발, 테스트, 프로덕션 환경의 일관성을 보장하며, 새로운 팀원의 온보딩 시간을 크게 단축할 수 있습니다.

**안정적인 서비스 운영**: 헬스체크와 의존성 관리를 통해 서비스 간의 시작 순서를 제어함으로써 시스템 안정성을 높일 수 있습니다. MongoDB의 초기화 스크립트 기능을 활용하면 데이터베이스 설정의 재현성을 보장할 수 있습니다.

**보안 모범 사례**: 비루트 사용자로 애플리케이션 실행, 데이터베이스 인증 활성화, 네트워크 격리 등의 기본적인 보안 조치를 적용하는 것이 중요합니다. 환경별 설정 분리를 통해 개발 편의성과 운영 보안을 모두 확보할 수 있습니다.

**확장 가능한 아키텍처**: 현재의 기본 구성을 바탕으로 필요에 따라 마이크로서비스 아키텍처, 고가용성 데이터베이스 구성, 고급 모니터링 시스템으로 발전시킬 수 있습니다.

이 예제는 실제 프로젝트에서 바로 사용할 수 있을 뿐만 아니라, 더 복잡한 시스템을 구축하기 위한 기반으로도 활용될 수 있습니다. 각 조직의 특정 요구사항에 맞게 이 구성을 수정하고 확장하여 효과적인 컨테이너 기반 애플리케이션을 구축하시기 바랍니다.