---
layout: post
title: Docker - Nginx + Markdown Docker 배포
date: 2025-02-27 20:20:23 +0900
category: Docker
---
# 📝 정적 블로그 (Nginx + Markdown) Docker 배포 가이드

---

## 📌 목표

- Markdown 문서를 정적으로 HTML로 변환
- Nginx로 정적 웹사이트 제공
- Docker Compose를 사용하여 배포 자동화
- 유지보수 쉽고 경량화된 기술 블로그 구축

---

## 🔧 아키텍처 개요

```
[Markdown 문서] → [정적 사이트 생성기] → [HTML 파일] → [Nginx로 서비스]
```

---

## 🧰 준비 도구

| 도구 | 설명 |
|------|------|
| Markdown | 블로그 콘텐츠 작성 형식 |
| Static Site Generator | 예: `Hugo`, `Jekyll`, `MkDocs`, `Zola` |
| Nginx | 정적 파일을 서비스하는 웹 서버 |
| Docker | 모든 구성 요소를 컨테이너로 실행 |

---

## 📁 디렉터리 구조 예시 (Hugo 기반)

```plaintext
my-blog/
├── content/            # Markdown 파일들
│   └── post1.md
├── public/             # 생성된 HTML 정적 사이트 (Nginx가 서비스할 폴더)
├── nginx/
│   └── default.conf    # Nginx 설정
├── Dockerfile          # Hugo 빌드용
├── docker-compose.yml
```

---

## ⚙️ 정적 사이트 생성기: Hugo 설치 및 빌드

### 📌 Dockerfile (Hugo)

```Dockerfile
FROM klakegg/hugo:0.126.1-ext-alpine

WORKDIR /src
COPY . .
RUN hugo --minify
```

- `hugo --minify`로 `public/` 폴더에 정적 HTML 생성
- 이 HTML을 Nginx에 전달할 예정

---

## 📃 Nginx 설정 (`nginx/default.conf`)

```nginx
server {
  listen 80;
  server_name localhost;

  root /usr/share/nginx/html;
  index index.html;

  location / {
    try_files $uri $uri/ =404;
  }
}
```

---

## 🐳 docker-compose.yml

```yaml
version: '3.9'

services:
  builder:
    build: .
    volumes:
      - ./public:/src/public

  web:
    image: nginx:alpine
    ports:
      - "8080:80"
    volumes:
      - ./public:/usr/share/nginx/html:ro
      - ./nginx/default.conf:/etc/nginx/conf.d/default.conf:ro
    depends_on:
      - builder
```

---

## 🚀 실행 방법

```bash
# 1. Markdown 글 작성 (content/ 디렉터리)
# 2. 빌드 및 배포 실행
docker-compose up --build
```

- `builder` 컨테이너가 Hugo로 빌드 후 `public/`에 HTML 출력
- `web` 컨테이너(Nginx)가 해당 정적 사이트를 서비스

---

## 🌐 접속 테스트

브라우저에서 다음 주소로 접속:

```
http://localhost:8080
```

정상적으로 HTML이 렌더링되면 성공입니다!

---

## ✅ Hugo 설치 없이 Markdown 빠르게 서비스하기 (기본 HTML 사용)

정적 변환 없이도 markdown을 HTML로 수동 변환해도 됩니다:

```bash
pandoc post1.md -o post1.html
```

→ `public/` 폴더에 넣고 Nginx에서 제공

---

## 🧪 추가 기능 확장

| 기능 | 방법 |
|------|------|
| 다국어 지원 | Hugo `i18n` 기능 |
| 댓글 시스템 | Disqus 연동 |
| GitHub Action CI/CD | 푸시 시 자동 배포 |
| HTTPS 적용 | Nginx + Certbot 도커 조합 |
| SEO 메타태그 자동 추가 | Hugo 테마 수정 |
| 마크다운에 수식, 차트 포함 | MathJax, Mermaid.js 포함 테마 사용 |

---

## 🛠️ 예시 Hugo 테마 적용

```bash
git submodule add https://github.com/adityatelange/hugo-PaperMod.git themes/PaperMod
echo 'theme = "PaperMod"' >> config.toml
```

---

## 📚 추천 SSG 도구 (선택사항)

| 도구 | 특징 |
|------|------|
| **Hugo** | Go 기반, 매우 빠름, 인기 많음 |
| **Jekyll** | Ruby 기반, GitHub Pages와 잘 맞음 |
| **MkDocs** | Python 기반 문서 사이트에 적합 |
| **Zola** | Rust 기반, 빠르고 간단함 |

---

## 📦 Docker 이미지 참고

- [klakegg/hugo Docker Hub](https://hub.docker.com/r/klakegg/hugo)
- [nginx 공식 이미지](https://hub.docker.com/_/nginx)