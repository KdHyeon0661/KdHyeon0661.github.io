---
layout: post
title: Docker - Docker Secrets
date: 2025-03-13 20:20:23 +0900
category: Docker
---
# 🔐 Docker Secrets 활용하기
> 민감 정보를 안전하게 저장하고 사용하는 보안 메커니즘

---

## 📌 왜 Docker Secrets가 필요한가?

| 방법 | 보안 수준 | 특징 |
|------|------------|------|
| `.env` 파일 | ❌ 낮음 | Git에 커밋될 위험 |
| `docker run -e` | ⚠️ 중간 | `ps`, `docker inspect`에서 노출 |
| 🔐 `Docker Secrets` | ✅ 높음 | 메모리 기반 노출 차단, 암호화 전달 |

---

## 🔧 Docker Secrets의 핵심 원리

- **Swarm 모드**에서만 동작
- Secret은 **암호화되어 클러스터 노드에 저장**
- 컨테이너 내부에서는 `/run/secrets/<secret-name>` 파일로 접근
- 읽기 전용, 자동 마운트, 로그나 환경변수에 남지 않음

---

## 🧪 사용 흐름 요약

1. **Docker Swarm 초기화**
2. **Secret 생성**
3. **서비스에 Secret 할당**
4. **컨테이너 내부에서 `/run/secrets/<name>`로 읽기**

---

## 🛠️ 1. Swarm 초기화

```bash
docker swarm init
```

> 이미 활성화된 경우 생략 가능

---

## 📄 2. Secret 생성

```bash
echo "my-secret-password" | docker secret create db_password -
```

```bash
docker secret ls
```

| ID | NAME |
|----|------|
| abcd123 | db_password |

---

## 🔗 3. Secret 사용해서 서비스 생성

```bash
docker service create \
  --name myapp \
  --secret db_password \
  myapp-image
```

> `--secret`으로 연결하면 컨테이너 안에서 `/run/secrets/db_password`로 자동 마운트됩니다.

---

## 🔍 4. 컨테이너 내부에서 확인

```bash
docker exec -it <container-id> cat /run/secrets/db_password
```

출력:
```
my-secret-password
```

---

## 📦 Docker Compose 예시

### 🔒 secrets 폴더 구조

```plaintext
.
├── docker-compose.yml
└── secrets/
    └── db_password.txt
```

### ✅ docker-compose.yml

```yaml
version: "3.9"

services:
  web:
    image: myapp
    secrets:
      - db_password

secrets:
  db_password:
    file: ./secrets/db_password.txt
```

---

## 📖 앱 코드 내에서는?

```python
with open("/run/secrets/db_password", "r") as f:
    db_pass = f.read().strip()
```

---

## ✅ 장점 정리

| 항목 | 장점 |
|------|------|
| 안전한 전달 | 이미지, 로그, 환경변수에 남지 않음 |
| 읽기 전용 마운트 | 컨테이너 내부 수정 불가 |
| 메모리 제한 | 디스크 저장 안 함 |
| 유출 위험 감소 | 일반 명령어(`ps`, `inspect`)에 표시 안 됨 |

---

## ⚠️ 주의 사항

- 일반 `docker run`으로는 사용 불가 → 반드시 Swarm 모드 (`docker service`)에서만 가능
- `docker-compose`도 Secrets는 Swarm 모드에서만 fully 지원됨 (`docker stack deploy`)

---

## 🧰 고급 기능

| 기능 | 설명 |
|------|------|
| Secret update | 불가능 → 삭제 후 재생성 필요 |
| Secret scope 제한 | 특정 서비스만 접근 가능 |
| 여러 Secret 연결 | `--secret` 옵션을 반복 사용 |
| `target` 이름 변경 | 마운트 경로 이름 지정 가능 (`target:`)

### 예시

```yaml
secrets:
  - source: db_password
    target: mysql_password
```

→ `/run/secrets/mysql_password` 로 접근됨

---

## 🧪 Secret 삭제

```bash
docker secret rm db_password
```

---

## 📚 참고 링크

- [Docker 공식 Secrets 문서](https://docs.docker.com/engine/swarm/secrets/)
- [Compose Secrets 문서](https://docs.docker.com/compose/compose-file/compose-file-v3/#secrets)
- [Best practices for secrets](https://docs.docker.com/engine/swarm/secrets/#tips-for-using-secrets)

---

## ✅ 요약

| 기능 | 설명 |
|------|------|
| 생성 | `docker secret create` |
| 할당 | `docker service create --secret` |
| 접근 | `/run/secrets/<name>` 경로로 읽기 |
| 보안성 | 로그/환경변수 노출 차단, 자동 암호화 |
| 제약 | Swarm 모드 필요, 수정 불가 |

---

## 🔐 관련 보안 연계 주제

- Vault / AWS Secrets Manager / GCP Secret Manager 연동
- GitHub Actions 내 secret과 Docker 연계
- Kubernetes Secrets vs Docker Secrets 비교