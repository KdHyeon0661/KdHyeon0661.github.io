---
layout: post
title: Docker - exec vs attach
date: 2025-01-14 19:20:23 +0900
category: Docker
---
# 🔌 Docker Volume Plugin: NFS, S3, Cloud Volume 완벽 가이드

---

## 📦 1. 왜 Volume Plugin이 필요한가?

기본적으로 Docker의 `volume`은 **호스트 로컬 디스크**를 사용합니다.  
하지만 다음과 같은 경우에는 **외부 스토리지**가 필요합니다:

- 여러 서버 간 데이터 공유 필요 (분산 시스템)
- 로컬 디스크 공간 한계
- 데이터 백업 및 클라우드 연동
- 고가용성, 마운트 지속성 필요

### ✅ 해결책: Docker Volume Plugin

> 외부 스토리지를 Docker에서 `volume`처럼 사용할 수 있게 해주는 플러그인 시스템

---

## 🗃️ 2. 지원되는 대표적인 외부 볼륨

| 유형 | 예시 | 용도 |
|------|------|------|
| 🔗 NFS (Network File System) | 사내 NAS, NFS 서버 | 다중 호스트 공유, 내부 시스템 |
| ☁️ Cloud Volume | AWS EBS, GCP PD, Azure Disk | 프로덕션 클라우드 환경 |
| ☁️ Object Storage | Amazon S3, MinIO | 정적 파일, 로그, 미디어 저장 등 |
| 기타 | CIFS, GlusterFS, iSCSI, Ceph | 고급 파일 시스템, 분산 스토리지 등 |

---

## 📡 3. NFS를 이용한 Docker Volume 예제

### ✅ 1. 전제 조건

- NFS 서버가 있어야 함 (`10.0.0.1:/data`)
- 클라이언트(도커 호스트)에 NFS 클라이언트 설치 필요

```bash
sudo apt install nfs-common
```

### ✅ 2. 볼륨 생성

```bash
docker volume create \
  --driver local \
  --opt type=nfs \
  --opt o=addr=10.0.0.1,rw \
  --opt device=:/data \
  nfs-volume
```

### ✅ 3. 컨테이너에서 사용

```bash
docker run -d -v nfs-volume:/app/data nginx
```

> 여러 Docker 호스트 간에 `nfs-volume`을 공유할 수 있음!

---

## ☁️ 4. Cloud Volume Plugin 예시

### ✅ 1. AWS EBS (Docker + EBS plugin)

Amazon에서 공식적으로 지원하지 않기 때문에, 커뮤니티 플러그인 사용:

```bash
docker plugin install rexray/ebs REXRAY_PREEMPT=true EBS_REGION=ap-northeast-2
```

#### 볼륨 생성

```bash
docker volume create --driver rexray/ebs --name myebsvol
```

#### 컨테이너에서 사용

```bash
docker run -d -v myebsvol:/data busybox
```

> EBS 볼륨이 자동으로 생성되어 컨테이너에 연결됨

---

## 🗃️ 5. Amazon S3 (s3fs or minio-fuse 등 이용)

### ❗ S3는 Object Storage라서 block device처럼 쓰려면 `FUSE` 필요

#### 전제: s3fs 설치 및 IAM 키 필요

```bash
# 설치 (Ubuntu)
sudo apt install s3fs

# AWS 키 등록
echo "AKIAXXX:SECRETXXX" > ~/.passwd-s3fs
chmod 600 ~/.passwd-s3fs
```

#### 디렉토리를 S3 버킷에 마운트

```bash
s3fs mybucket /mnt/s3 -o passwd_file=~/.passwd-s3fs -o url=https://s3.amazonaws.com
```

#### Bind Mount로 Docker 컨테이너에 연결

```bash
docker run -v /mnt/s3:/app/data nginx
```

> 실시간 반영은 느릴 수 있으므로 정적 파일 중심으로 사용

---

## 🧰 6. 기타 유명 Volume Plugins

| 플러그인 | 설명 |
|----------|------|
| `local-persist` | Docker 기본 local volume 확장 |
| `rexray/ebs`, `rexray/gce` | EBS, GCP 디스크 마운트 |
| `glusterfs`, `ceph` | 대규모 분산 스토리지용 |
| `docker-volume-netshare` | CIFS/NFS 지원 플러그인 |
| `minio`, `s3fs`, `goofys` | S3와 유사한 오브젝트 스토리지 플러그인 |

---

## ⚠️ 7. 주의사항 및 운영 팁

| 항목 | 주의 및 팁 |
|------|------------|
| 🔒 보안 | IAM, key 관리 철저히 (특히 S3) |
| 🔌 네트워크 | NFS/S3의 경우 네트워크 연결 필수 |
| 📦 자동 마운트 확인 | Docker 데몬 재시작 시 자동 마운트 보장 여부 확인 |
| 🐳 권한 문제 | 컨테이너 내부 권한과 외부 디스크 권한 일치 필요 |
| 📂 캐시 문제 | S3 연동 시에는 캐시 혹은 지연 발생 가능 |

---

## 📋 정리: 어떤 스토리지를 언제 쓸까?

| 용도 | 추천 스토리지 | 이유 |
|------|----------------|------|
| 사내 네트워크 공유 | NFS, CIFS | 쉽게 공유 가능, 설정 간단 |
| 클라우드 서버의 영구 저장 | AWS EBS, GCP PD | 성능 우수, 관리 편리 |
| 정적 파일, 이미지, 로그 | S3, MinIO | 비용 절감, 웹서버 최적 |
| 고성능 분산 시스템 | Ceph, GlusterFS | 확장성과 내결함성 제공 |

---

## 🧩 관련 명령어

```bash
# 설치된 플러그인 확인
docker plugin ls

# 플러그인 설치
docker plugin install <plugin-name>

# 플러그인 제거
docker plugin disable <plugin-name>
docker plugin rm <plugin-name>
```

---

## 📚 참고 자료

- [Docker Volume 공식 문서](https://docs.docker.com/storage/volumes/)
- [Docker Plugin 사용법](https://docs.docker.com/engine/extend/)
- [RexRay GitHub](https://github.com/rexray/rexray)
- [s3fs 프로젝트](https://github.com/s3fs-fuse/s3fs-fuse)