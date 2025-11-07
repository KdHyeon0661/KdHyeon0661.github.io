---
layout: post
title: Docker - Docker Volume Plugin
date: 2025-01-26 19:20:23 +0900
category: Docker
---
# Docker Volume Plugin: NFS, S3, Cloud Volume

- 왜 플러그인이 필요한가 → **멀티 호스트 공유/클라우드 영구 디스크/오브젝트 스토리지**  
- NFS/Cloud Volumes(EBS/PD/Azure Disk)/**오브젝트 스토리지(S3/MinIO) FUSE** 방식  
- `docker volume create --driver ...` / `--mount` / `compose` 정석 패턴  
- **보안(자격증명/네트워크/SELinux/Kerberos)**, **성능(IOPS/캐시/마운트옵션)**, **HA/잠금**  
- 백업/복구/스냅샷/마이그레이션 + 트러블슈팅 표/체크리스트

> 주의: 오브젝트 스토리지(S3)는 **POSIX 비호환**이며 DB/트랜잭션 파일 저장에는 적합하지 않습니다. 정적 자산/로그/백업 대상에 권장됩니다.

---

## 1) 왜 Volume Plugin인가?

기본 `local` 볼륨은 도커 호스트의 로컬 디스크에 저장됩니다. 다음 상황에서는 **외부 스토리지**가 필요합니다.

- 다수의 Docker 호스트(스웜/수동 롤링)에서 **공유 데이터** 접근
- 로컬 디스크 용량/내구성 한계 → **클라우드 블록 스토리지**(EBS/PD/Azure Disk)
- 정적 자산/로그/백업을 **오브젝트 스토리지(S3/MinIO)** 로 집약
- DR(재해복구)/스냅샷/스케일아웃 등 **스토리지 레벨 관리** 필요

효용의 직관(가중합):
$$
U \approx \alpha\cdot \text{Availability} + \beta\cdot \text{Scalability} + \gamma\cdot \text{Operational\,Simplicity}
$$
플러그인은 위 요소를 로컬 디스크 대비 크게 끌어올립니다.

---

## 2) 개념/아키텍처: Docker Volume Driver vs Plugin

- **Volume Driver**: Docker가 특정 스토리지를 **볼륨처럼** 취급하도록 하는 드라이버.  
- **Docker Plugin**: 드라이버/네트워크 등 기능을 **외부 플러그인** 프로세스로 제공(수명/설정/권한을 엔진과 분리).

흐름:
```
docker CLI → dockerd(Volume API) → Volume Driver/Plugin → 외부 스토리지 마운트/프로비저닝
```

핵심 명령:
```bash
docker plugin ls
docker plugin install <vendor/plugin> [key=value ...]
docker plugin disable <name>
docker plugin rm <name>

docker volume create --driver <driver> --name <vol> [--opt k=v ...]
docker run --mount type=volume,source=<vol>,target=/path ...
```

---

## 3) 스토리지 유형과 선택 기준(요약)

| 유형 | 예시 | 장점 | 주의 |
|---|---|---|---|
| **NFS** | 온프레 NAS, 리눅스 NFS 서버 | 손쉬운 멀티호스트 공유 | 네트워크/잠금/권한/보안(Kerberos) |
| **클라우드 블록** | AWS **EBS**, GCP **PD**, Azure **Disk** | 고성능/스냅샷/운영툴 풍부 | AZ 종속/단일 호스트 어태치 기본 |
| **오브젝트** | **S3**, MinIO | 비용효율/버전닝/내구성 | POSIX 미호환, FUSE 지연/일관성 |

---

## 4) NFS with Docker — 가장 간단한 멀티호스트 공유

### 4.1 전제: NFS 서버
- 서버 예: `10.0.0.1:/exports/data`
- 클라이언트(도커 호스트)에 NFS 유틸 설치
```bash
# Ubuntu/Debian
sudo apt update && sudo apt install -y nfs-common
# RHEL/CentOS
sudo yum install -y nfs-utils
```

### 4.2 local 드라이버로 NFS 마운트(권장: --mount 옵션 이해)
```bash
docker volume create \
  --driver local \
  --opt type=nfs \
  --opt o=addr=10.0.0.1,rw,nfsvers=4.1,timeo=600,retrans=2 \
  --opt device=:/exports/data \
  nfs-volume
```

### 4.3 컨테이너에서 사용
```bash
docker run -d --name web \
  --mount type=volume,source=nfs-volume,target=/var/www/html \
  -p 8080:80 nginx:alpine
```

#### 옵션 팁
- `nfsvers=4.1`(또는 4.2): 최신 프로토콜, 성능/보안 유리  
- `timeo/retrans`: 네트워크 지연/재전송 튜닝  
- SELinux 환경은 `context` 라벨 정책(CentOS/RHEL) 고려  
- 보안 민감 시 **NFSv4 + Kerberos(krb5/krb5i/krb5p)** 구성

### 4.4 Compose 패턴
```yaml
version: "3.9"
services:
  web:
    image: nginx:alpine
    ports: ["8080:80"]
    volumes:
      - nfs-volume:/usr/share/nginx/html
volumes:
  nfs-volume:
    driver: local
    driver_opts:
      type: nfs
      o: "addr=10.0.0.1,rw,nfsvers=4.1,timeo=600,retrans=2"
      device: ":/exports/data"
```

---

## 5) 클라우드 블록 스토리지 — EBS / PD / Azure Disk

> 원칙: **블록 디바이스는 기본적으로 단일 EC2/VM에 어태치**(읽기/읽기쓰기 모드 제약). 멀티 노드 공유가 필요하면 파일시스템 레이어(NFS/FSx for Lustre/Filestore/ANF 등)를 별도로 구성.

### 5.1 AWS EBS (예: rexray/ebs 플러그인)
플러그인 설치:
```bash
docker plugin install rexray/ebs \
  REXRAY_PREEMPT=true \
  EBS_REGION=ap-northeast-2
```

볼륨 프로비저닝/사용:
```bash
docker volume create --driver rexray/ebs --name myebs
docker run -d --name app \
  --mount type=volume,source=myebs,target=/data \
  busybox sleep 3600
```

옵션(예시):
```bash
docker volume create \
  --driver rexray/ebs \
  --name db-ebs \
  -o size=200 \
  -o volumetype=gp3 \
  -o iops=6000 \
  -o throughput=250
```
- **AZ 일치** 필수(EC2 인스턴스와 같은 AZ).  
- 스냅샷 기반 생성: `-o snapshotid=snap-xxxxxxxx`.

### 5.2 GCP PD (rexray/gcepd 등)
```bash
docker plugin install rexray/gcepd GCEPD_PROJECT=<project-id> GCEPD_TAGS="docker"
docker volume create --driver rexray/gcepd --name gpd-1 -o size=100
docker run -d --mount type=volume,source=gpd-1,target=/data busybox sleep 3600
```

### 5.3 Azure Disk (예: rexray/azureud 등 플러그인)
- 설치/인증은 서비스 프린시펄 또는 MSI 방식.  
- 동일하게 `docker volume create --driver ...` 패턴으로 사용.

#### 실전 팁
- DB/큐 등 랜덤 IO 워크로드 → **IOPS/스루풋 지표** 기준 선택(gp3/io2, pd-ssd 등)  
- 스냅샷/백업은 **클라우드 네이티브 방법**(EBS Snapshot, GCP Snapshot) 사용 권장  
- **AZ 이동**은 스냅샷 → 재생성 경로로 수행

---

## 6) 오브젝트 스토리지(S3/MinIO) — FUSE로 “마운트처럼” 쓰기

> 오브젝트 스토리지는 **디렉터리/원자성/락/퍼미션**이 파일시스템과 다릅니다.  
> DB/트랜잭션/빈번한 작은 쓰기에는 부적합. 정적 파일/백업/로그 적합.

### 6.1 s3fs (FUSE) 설치/자격증명
```bash
sudo apt update && sudo apt install -y s3fs
echo "AKIAXXX:SECRETXXX" > ~/.passwd-s3fs
chmod 600 ~/.passwd-s3fs
```

### 6.2 마운트
```bash
mkdir -p /mnt/s3
s3fs mybucket /mnt/s3 \
  -o passwd_file=~/.passwd-s3fs \
  -o url=https://s3.amazonaws.com \
  -o use_path_request_style \
  -o allow_other \
  -o umask=0022
```

### 6.3 Docker에 연결(Bind Mount)
```bash
docker run -d --name web \
  --mount type=bind,source=/mnt/s3,target=/usr/share/nginx/html,readonly \
  -p 8080:80 nginx:alpine
```

#### 대안/고급
- **goofys**(지연쓰기/메타데이터 빠름)  
- **MinIO Gateway** 또는 MinIO 자체를 **NFS/포지식 파일시스템 뒤에 올려** 제공  
- 성능/일관성 요구가 높다면 **파일시스템 계층(NAS/FSx/Filestore/ANF)** 고려

---

## 7) 보안 — 자격증명/네트워크/SELinux/Kerberos

- 자격증명:  
  - 클라우드 플러그인은 **인스턴스 롤/서비스 계정** 활용(키 저장 최소화)  
  - s3fs는 **IAM 사용자 최소 권한** + **버킷 정책** + **버전닝/암호화**  
- 네트워크:  
  - NFS는 **보안 도메인 제한** + 방화벽 + NFSv4 + **Kerberos(krb5p)** 권장  
  - EBS/PD/Azure Disk는 **해당 클라우드 네트워크/권한 범위**에서만 접근  
- Linux 보안:  
  - SELinux/AppArmor 정책, `:Z/:z` 또는 `--mount ...,z`(라벨)  
  - 컨테이너 **비루트 USER**, `--read-only` 루트FS, 필요 디렉터리는 `tmpfs` 부여

예시(비루트 + 읽기전용 + tmpfs):
```bash
docker run -d \
  --user 65532:65532 \
  --read-only \
  --tmpfs /tmp --tmpfs /run \
  --mount type=volume,source=cfg,target=/app/cfg,readonly \
  myimg
```

---

## 8) 성능 — 마운트 옵션/캐시/IOPS/스루풋

- **NFS**: `nfsvers=4.1/4.2`, `rsize/wsize`, `timeo`, `retrans`, `noatime` 고려.  
- **블록**: 스토리지 클래스(gp3/io2, pd-ssd 등)와 **크기→IOPS/Throughput** 관계 숙지.  
- **오브젝트 FUSE**: 지연/일관성/메타데이터 호출 비용 큼. 대용량 순차 읽기/쓰기 위주로.

성능 예산의 간단 모델:
$$
\text{Latency} \approx \text{Network RTT} + \text{Server FS overhead} + \text{Client FS overhead}
$$
$$
\text{Throughput} \lesssim \min(\text{Link BW}, \text{Server Limit}, \text{Disk Limit})
$$

---

## 9) 백업/복구/스냅샷/마이그레이션

### 9.1 Volume 내용 tar 백업/복구
```bash
# 백업
docker run --rm \
  -v myvol:/src \
  -v "$(pwd)":/backup \
  alpine sh -c "cd /src && tar czf /backup/myvol-$(date +%F).tgz ."

# 복구
docker run --rm \
  -v myvol:/dest \
  -v "$(pwd)":/backup \
  alpine sh -c "cd /dest && tar xzf /backup/myvol-2025-11-06.tgz"
```

### 9.2 클라우드 스냅샷
- EBS/PD/Azure Disk는 **스냅샷 → 새 볼륨 생성** 루트로 손쉬운 마이그레이션.  
- DB는 **애플리케이션 일관 시점** 확보 후 스냅샷/덤프 권장.

### 9.3 오브젝트
- `rclone sync`, S3 버전닝/수명주기 정책으로 보존/아카이빙.

---

## 10) Compose 패턴 총정리

### 10.1 NFS
```yaml
volumes:
  webdata:
    driver: local
    driver_opts:
      type: nfs
      o: "addr=10.0.0.1,rw,nfsvers=4.1"
      device: ":/exports/web"
```

### 10.2 EBS(예: rexray/ebs 사용 가정)
```yaml
volumes:
  ebsdata:
    driver: rexray/ebs
    driver_opts:
      volumetype: gp3
      size: "200"
```

### 10.3 S3(FUSE 마운트 후 바인드)
```yaml
services:
  app:
    image: nginx:alpine
    volumes:
      - /mnt/s3:/usr/share/nginx/html:ro
```

---

## 11) 트러블슈팅 표

| 증상 | 원인 | 진단 | 해결 |
|---|---|---|---|
| NFS 권한 에러 | UID/GID/Export 옵션/SELinux | `id`, `ls -l`, 서버 `/etc/exports` | export 수정, `:Z/:z`, `--user` 정렬 |
| 느린 I/O(NFS/S3) | 네트워크/마운트옵션/메타데이터 과다 | `iostat`, `nfsstat`, `iotop` | rsize/wsize/캐시 조정, 워크로드 재설계 |
| EBS 어태치 실패 | AZ 불일치/권한 | 클라우드 콘솔/플러그인 로그 | 인스턴스·EBS AZ 일치, 역할 권한 확인 |
| 플러그인 안 보임 | 설치/활성화 문제 | `docker plugin ls` | `docker plugin install/enable` 재시도 |
| S3 일관성 문제 | 오브젝트 지연/리스트 지연 | 앱 로그/버킷 설정 | 버전닝/캐시 정책, 설계(최종 일관성) 반영 |
| 데이터 손상 | 다중 쓰기/락 미사용 | 앱 로그/FS 락 | DB는 단일 인스턴스 전담, NFS 락/프로토콜 준수 |

---

## 12) 실습 시나리오 묶음

### 12.1 “멀티 호스트 공유 정적 자산” — NFS
```bash
# 볼륨 만들기
docker volume create \
  --driver local \
  --opt type=nfs \
  --opt o=addr=10.0.0.1,rw,nfsvers=4.1 \
  --opt device=:/exports/web \
  web-nfs

# 컨텐츠 주입(호스트 A)
docker run --rm \
  -v web-nfs:/dst \
  -v "$(pwd)"/public:/src:ro \
  alpine sh -c "cp -r /src/* /dst/"

# 어느 호스트에서나 서비스
docker run -d -p 8080:80 \
  --mount type=volume,source=web-nfs,target=/usr/share/nginx/html,readonly \
  nginx:alpine
```

### 12.2 “고IO DB 단일 인스턴스” — EBS
```bash
docker plugin install rexray/ebs EBS_REGION=ap-northeast-2 REXRAY_PREEMPT=true

docker volume create --driver rexray/ebs \
  --name pg-ebs \
  -o volumetype=gp3 -o size=200 -o iops=6000 -o throughput=250

docker run -d --name pg \
  -e POSTGRES_PASSWORD=secret \
  --mount type=volume,source=pg-ebs,target=/var/lib/postgresql/data \
  -p 5432:5432 postgres:16
```

### 12.3 “정적 파일 배포/로그 보관” — S3(FUSE)
```bash
# s3fs 마운트
s3fs mybucket /mnt/s3 -o passwd_file=~/.passwd-s3fs -o url=https://s3.amazonaws.com -o allow_other

# 읽기 전용 서빙
docker run -d -p 8081:80 \
  --mount type=bind,source=/mnt/s3,target=/usr/share/nginx/html,readonly \
  nginx:alpine
```

---

## 13) 운영 체크리스트

- [ ] 데이터 유형별 스토리지 선택(NFS/블록/S3)  
- [ ] DB/트랜잭션 파일은 **POSIX 파일시스템**(NFS/블록) 사용  
- [ ] 클라우드 블록은 **AZ/권한/IOPS/Throughput** 설계  
- [ ] NFS는 **버전/옵션/락/보안(Kerberos)** 확인  
- [ ] S3는 **정적/백업** 용도 중심, 캐시·일관성 고려  
- [ ] 컨테이너 **비루트/읽기전용** + SELinux/AppArmor  
- [ ] 정기 **스냅샷/백업** + 체크섬/암호화  
- [ ] `docker plugin/volume` 상태 점검 및 로테이션  
- [ ] Compose/인프라 코드(IaC)로 **버전관리**

---

## 14) 명령 요약

```bash
# 플러그인
docker plugin ls
docker plugin install <name> [KEY=VALUE ...]
docker plugin disable <name>
docker plugin rm <name>

# 볼륨
docker volume create --driver <driver> --name <vol> [--opt k=v ...]
docker volume ls
docker volume inspect <vol>
docker volume rm <vol>

# 컨테이너 마운트(권장 --mount)
docker run --mount type=volume,source=<vol>,target=/path image
docker run --mount type=bind,source=/host/path,target=/path image
```

---

## 15) 참고(개념적 구분)
- Docker Volume Plugin/Driver는 **Docker 엔진 생태계**의 확장 메커니즘  
- Kubernetes 환경에서는 **CSI(컨테이너 스토리지 인터페이스)** 가 표준(개념은 유사, 구현·리소스/정책은 다름)

---

## 부록) 비용/성능 직관 수식

스토리지 선택 시 단순화된 의사결정:
$$
\text{Cost\_month} \approx \text{CapEx/OpEx} + c_1\cdot \text{GB} + c_2\cdot \text{IOPS} + c_3\cdot \text{Throughput}
$$

목표는
$$
\max\ U = w_P P + w_A A + w_S S - w_C \text{Cost},
$$
여기서 \(P\): 성능, \(A\): 가용성, \(S\): 보안/운영성, \(w_\*\): 업무 비중.

---

## 참고 자료
- Docker Storage Volumes / Plugins / Engine API 문서
- 각 클라우드(EBS/PD/Azure Disk) 스냅샷/성능 가이드
- s3fs-fuse / goofys / MinIO(게이트웨이/디스크 백엔드)
- NFSv4/Kerberos/락/튜닝 베스트 프랙티스

이상으로 **NFS/클라우드 볼륨/S3(FUSE)** 를 아우르는 **Docker Volume Plugin 실전 가이드**를 마칩니다. 본 문서를 토대로, 서비스 특성(일관성·성능·가용성·비용)에 맞는 스토리지를 선택하고, 보안/백업/운영 절차를 표준화하시기 바랍니다.