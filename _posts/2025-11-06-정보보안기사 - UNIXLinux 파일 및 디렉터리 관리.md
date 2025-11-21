---
layout: post
title: 정보보안기사 - UNIXLinux 파일 및 디렉터리 관리
date: 2025-11-06 16:25:23 +0900
category: 정보보안기사
---
# SECTION 02 UNIX/Linux 서버 취약점 — 02. 파일 및 디렉터리 관리

## 개요 — 왜 파일/디렉터리 관리가 취약점이 되는가

- **권한 과다**(world-writable, SUID/SGID 남발, 잘못된 umask) → 권한상승·데이터 변조.
- **특수비트/링크 취약**(sticky 부재, 하드링크/심링크 공격) → 임의 삭제/치환.
- **마운트 옵션 미설정**(noexec/nodev/nosuid 미적용) → 임의 실행·디바이스 악용.
- **ACL/xattr 미통제** → 비정형 권한 누수, 강제 접근통제 우회.
- **로그·백업 권한 부실** → 정보노출·DoS.
- **레이스(TOCTOU), /tmp 사용 오류** → 파일 탈취/자료 변조.
- **NFS/공유 디렉터리 실수** → root-squash 미설정, group 협업 디렉터리 오동작.

---

## 리마인드 — Unix 권한 모델과 특수 비트

### 기본 권한과 표기

- **소유자/그룹/기타** 3세트의 `r(4) w(2) x(1)`
- 8진수 표기: `chmod 750 file` → `rwx r-x ---`
- 디렉터리에서 `x`는 **디렉터리 진입·이름 탐색** 권한.

### 특수 비트

- **SUID(4xxx)**: 실행 시 소유자 권한 상속(파일만).
- **SGID(2xxx)**: 실행 시 그룹 상속(파일), 디렉터리에서 **기본 그룹 상속**.
- **Sticky(1xxx)**: `1777` 디렉터리에서 **소유자만 삭제 가능**(`/tmp` 등).

### umask와 최종 권한

파일 생성 요청 권한 \(R\)과 umask \(U\)의 **유효 권한** \(E\):
$$
E = R \ \&\ \sim U
$$
예: `umask 027`, 파일 기본 `666` → `640`, 디렉터리 기본 `777` → `750`.

---

## 취약 신호 자동 점검 스크립트(포터블)

```bash
#!/usr/bin/env bash
# fs_audit.sh — 파일/디렉터리 취약 구성 점검 (루트로 실행 권장)

set -euo pipefail

echo "[1] sticky 없는 world-writable 디렉터리"
find / -xdev -type d -perm -0002 ! -perm -1000 -printf '%m %u:%g %p\n' 2>/dev/null

echo -e "\n[2] world-writable 파일"
find / -xdev -type f -perm -0002 -printf '%m %u:%g %p\n' 2>/dev/null | head -100

echo -e "\n[3] SUID/SGID 파일"
find / -xdev -type f \( -perm -4000 -o -perm -2000 \) -printf '%m %u:%g %p\n' 2>/dev/null

echo -e "\n[4] 홈 디렉터리 권한(700/750 권장)"
awk -F: '{print $1":"$6}' /etc/passwd | while IFS=: read -r u h; do
  [ -d "$h" ] || continue
  p=$(stat -c '%a' "$h")
  echo "$u $p $h"
done | awk '$2 !~ /^(700|750|740|600|640)$/'

echo -e "\n[5] 위험 마운트 옵션 누락(/tmp, /var/tmp, /home, /data 등)"
mount | awk '/type (ext|xfs|btrfs|tmpfs)/ {print}'

echo -e "\n[6] cron/로그 디렉터리 권한"
ls -ld /etc/cron.* /var/log 2>/dev/null

echo -e "\n[7] 링크 보호 sysctl"
sysctl fs.protected_symlinks fs.protected_hardlinks fs.protected_fifos 2>/dev/null

echo -e "\n[8] setcap 적용 바이너리(특권 최소화 확인)"
getcap -r / 2>/dev/null | head -100
```

> 결과는 CSV/스크린샷 등으로 **증적화**하고, 항목별 **조치→검증**을 티켓으로 관리.

---

## 공유 디렉터리와 `/tmp` — Sticky 비트와 tmpfs + noexec

### 원칙

- **공유 쓰기 디렉터리**는 반드시 **Sticky(1xxx)**.
- `/tmp`,`/var/tmp`는 **`mode=1777,nodev,nosuid,noexec` tmpfs** 권장(서버 성격에 따라).

### 설정 예시

`/etc/fstab`:
```
tmpfs /tmp tmpfs defaults,rw,nosuid,nodev,noexec,relatime,mode=1777 0 0
tmpfs /var/tmp tmpfs defaults,rw,nosuid,nodev,noexec,relatime,mode=1777 0 0
```
적용:
```bash
mount -a
```

### 운영 점검

```bash
ls -ld /tmp /var/tmp    # drwxrwxrwt (t = sticky)
mount | grep -E '/tmp|/var/tmp'
```

**취약 사례**: Sticky 없는 `0777` 디렉터리 → 타 사용자 파일을 삭제/치환 가능.
**조치**:
```bash
chmod 1777 /shared_tmp
```

---

## SUID/SGID — 최소화와 대체(Capabilities)

### 위험성

- SUID root 바이너리는 **환경변수/경로/LD_PRELOAD/argv 처리** 결함과 결합 시 권한 상승 통로.

### 절차

1) 목록 산출 → 기능 확인 → 제거/업데이트/대체.
2) **가능하면 setcap로 대체**(예: 1024↓ 포트 바인드).

```bash
# SUID/SGID 목록

find / -xdev -type f -perm -4000 -o -perm -2000 -print 2>/dev/null

# SUID 제거(예시)

chmod u-s /usr/bin/at

# 이하 포트 바인딩이 필요한 비루트 서버

setcap 'cap_net_bind_service=+ep' /usr/local/bin/web-lite
getcap /usr/local/bin/web-lite
```

체크리스트
- [ ] SUID/SGID 최소화, 패키지 업데이트로 취약 바이너리 제거.
- [ ] Capabilities로 권한 최소 부여, 목록 **정기 리포트**.

---

## umask/UMask — 시스템·서비스 기본 권한 정책

### 시스템 기본

`/etc/profile` 또는 `/etc/profile.d/*`:
```bash
umask 027
```

### 서비스 단위(systemd)

`/etc/systemd/system/web.service`:
```ini
[Service]
User=web
Group=web
UMask=007      # 그룹 협업(770/660), 기타 접근 차단
```
적용:
```bash
systemctl daemon-reload && systemctl restart web
```

체크리스트
- [ ] 비공개 데이터는 **027/077** 범위, 협업 디렉터리는 **UMask=007** 조합.
- [ ] 배포 스크립트/컨테이너 진입점에도 일관 적용.

---

## 디렉터리 협업 설계 — SGID+기본 ACL

### 목표

- 팀 디렉터리에서 생성 파일이 **항상 팀 그룹**으로, 권한도 **일관**되게.

```bash
# 팀 그룹 생성

groupadd devteam
mkdir -p /srv/project
chgrp devteam /srv/project
chmod 2770 /srv/project    # SGID(2) + rwx for owner/group

# 기본 ACL로 파일/디렉터리 권한 고정

setfacl -d -m g::rwx /srv/project
setfacl -d -m o::--- /srv/project
# 사용자별 ACL 추가(필요 시)

setfacl -m u:alice:rwx /srv/project
getfacl /srv/project
```

체크리스트
- [ ] 팀 디렉터리는 **SGID + 기본 ACL**로 “권한 드리프트” 제거.
- [ ] 홈 디렉터리와 협업 디렉터리 정책 **명확 분리**.

---

## 마운트 옵션 하드닝 — noexec/nodev/nosuid/bind, ro

### 원칙

- **코드 실행이 필요 없는** 파티션: `noexec`.
- **디바이스 파일 불필요**: `nodev`.
- **SUID 필요 없음**: `nosuid`.
- 민감 경로는 **read-only bind mount** 또는 전용 파티션을 `ro`.

`/etc/fstab` 예:
```
/dev/vdb1  /data    ext4  defaults,noexec,nodev,nosuid  0 2
/dev/vdb2  /backups ext4  defaults,nodev,nosuid,ro      0 2
```

**bind + ro**:
```bash
mount --bind /srv/app/static /srv/app/static
mount -o remount,bind,ro /srv/app/static
```

체크리스트
- [ ] `/tmp`, `/var/tmp`, `/home`, `/data`, 백업/아카이브 경로에 적절한 옵션.
- [ ] **컨테이너 런타임 볼륨**에도 동일 정책(가능 범위 내).

---

## 로그·백업 — 권한/회전/유출 방지

### logrotate (권한/보존/압축)

`/etc/logrotate.d/web`:
```
/var/log/web/*.log {
  daily
  rotate 14
  compress
  missingok
  notifempty
  create 0640 web web
  postrotate
    systemctl kill -s HUP --kill-who=main web.service
  endscript
}
```

### rsync/tar — 권한 보존/유출 방지

```bash
# 권장 rsync: 권한/ACL/xattr/하드링크/소유자 보존

rsync -aHAX --numeric-ids --delete /srv/web/ backup:/backups/web/

# tar 아카이브(ACL/xattr 포함)

tar --acls --xattrs -cpf web_$(date +%F).tar /srv/web
```
**주의**: 외부 전송 시 **암호화 채널**(ssh, restic/borg) 사용, 백업 경로는 `nodev,nosuid,ro` 고려.

체크리스트
- [ ] 로그 파일은 **0640** 이상, 디렉터리 **0750**.
- [ ] 백업/아카이브 디렉터리는 **권한 제한 + ro 마운트**(필요 시).

---

## 확장 속성 — xattr/Immutable/Append-only

### chattr/lsattr (ext 계열)

```bash
# 중요 설정 파일을 변경 불가(Immutable)로

chattr +i /etc/resolv.conf
lsattr /etc/resolv.conf
# 로그 디렉터리는 append-only(+a)로 제한 가능(운영 영향 고려)

chattr +a /var/log/app.log
```

> Immutable/Append-only는 **root도 쓰기 불가**, 변경엔 `chattr -i/-a` 필요.
> 배포/업데이트 파이프라인과 충돌 가능 — **변경 관리에 문서화**할 것.

### xattr (getfattr/setfattr)

```bash
setfattr -n user.owner -v "web" /srv/web/README.txt
getfattr -d /srv/web/README.txt
```
> xattr는 메타데이터 누수에 유의(백업/전송 시 포함 여부 결정).

---

## 방어 — sysctl + 안전한 생성

### sysctl 보호(커널 레벨)

```bash
cat >/etc/sysctl.d/99-link-protect.conf <<'EOF'
fs.protected_hardlinks = 1
fs.protected_symlinks = 1
fs.protected_fifos = 2
EOF
sysctl --system
```

### 안전한 임시 파일 생성

- **mktemp 사용**(쉘) 또는 `open(O_CREAT|O_EXCL|O_NOFOLLOW)`(C/Python).
- **/tmp 대신 PrivateTmp**(systemd) 또는 앱별 전용 `0700` 디렉터리.

쉘 예:
```bash
tmp=$(mktemp -d -p /tmp app.XXXXXX)
install -m 600 /dev/null "$tmp/data"
# ... 사용 후

rm -rf "$tmp"
```

Python 예:
```python
import os, tempfile
with tempfile.NamedTemporaryFile(dir="/tmp", prefix="app_", delete=True) as f:
    os.write(f.fileno(), b"...")
```

systemd 격리:
```ini
[Service]
PrivateTmp=true
ProtectSystem=strict
```

체크리스트
- [ ] 링크 보호 sysctl = 1(또는 2) 확인.
- [ ] 임시파일은 **경쟁 상태 없는 생성** 사용, 서비스는 **PrivateTmp**.

---

## NFS/원격 마운트 — root-squash와 안전 옵션

### 위험 포인트

- `no_root_squash`: 클라이언트 root가 서버에서도 root 권한.
- `rw` 공유 + 권한 느슨함 → 임의 변조.

### 점검/설정

서버(`/etc/exports`):
```
/srv/share 10.0.0.0/24(rw,sync,no_subtree_check,root_squash)
```
클라이언트(`/etc/fstab`):
```
nfsserver:/srv/share /mnt/share nfs4 rw,nosuid,nodev,noexec 0 0
```

체크리스트
- [ ] **root_squash** 활성, 클라 마운트 `nosuid,nodev,noexec`.
- [ ] NFSv4 ACL과 로컬 ACL 상호작용 검토.

---

## 무결성/감사 — AIDE + auditd(디렉터리/파일)

### AIDE

```bash
apt install -y aide || dnf install -y aide
aideinit
mv /var/lib/aide/aide.db.new /var/lib/aide/aide.db
aide --check | tee /var/log/aide/last.txt
```
정책에서 `/etc`, `/usr/local/bin`, `/etc/systemd/system`, `/var/www` 등 지정.

### auditd — 디렉터리 변경 감시

```bash
auditctl -w /etc -p wa -k etcchg
auditctl -w /etc/sudoers -p wa -k sudoers
auditctl -w /var/www -p wa -k webroot
auditctl -w /etc/cron.d -p wa -k cronchg
augenrules --load
ausearch -k webroot -ts today
```

체크리스트
- [ ] **중요 경로 write/attr** 감시, AIDE 주기 검사.
- [ ] 결과는 **중앙 SIEM**으로 전송·보관.

---

## 웹 업로드/정적 컨텐츠 디렉터리 — 분리·noexec·캐릭터 디바이스 금지

- 업로드 경로를 **서버 실행 경로와 분리**, 별도 파티션/바인드 마운트에 `noexec,nodev,nosuid`.
- **정적만 제공**(NGINX `autoindex off`, 인덱스 생성 금지).
- 서버단에서 **확장자 화이트리스트/매직넘버 검사**.

NGINX 스니펫:
```nginx
location /uploads/ {
    alias /srv/uploads/;
    autoindex off;
    default_type application/octet-stream;
    add_header X-Content-Type-Options nosniff;
}
```

체크리스트
- [ ] 업로드/정적 디렉터리 권한(750/640), 실행 비활성.
- [ ] 스토리지 분리 및 **역경로 타기 금지**(경로 정규화).

---

## 하드링크/심링크 기반 공격 차단 — 운영 시나리오

### 시나리오: 로그 로테이션 심링크 공격

- 취약: `logrotate`가 보호되지 않은 경로에서 `create 0644 root root`로 파일 생성.
- 대응: **소유/권한/경로 고정**, `create 0640 app app` + 디렉터리 0750 + Sticky 경로 회피.

logrotate 안전 설정:
```
/var/log/app/*.log {
  daily
  create 0640 app app
  su app app
  compress
  missingok
  notifempty
  postrotate
    systemctl kill -s HUP --kill-who=main app.service
  endscript
}
```

---

## 파일 시스템별 주의(ext4/xfs/btrfs)

- **ext4**: `chattr +i/+a` 가능, 정합성 좋음.
- **xfs**: `chattr +i/+a` 지원, quota/프로젝트 ID로 **디렉터리 단위 제한** 가능.
- **btrfs**: 서브볼륨/스냅샷 편리, **압축/디듀프**(운영 정책과 성능 고려).

XFS 프로젝트 쿼터 예:
```bash
xfs_quota -x -c 'project -s proj1' /
xfs_quota -x -c 'limit -p bhard=100g proj1' /
```

---

## Ansible로 하드닝 선언(샘플 플레이북)

```yaml
- hosts: linux
  become: true
  tasks:
    - name: Ensure /tmp and /var/tmp are tmpfs with secure options
      mount:
        path: "{{ item.path }}"
        src: tmpfs
        fstype: tmpfs
        opts: "defaults,rw,nosuid,nodev,noexec,relatime,mode=1777"
        state: mounted
      loop:
        - { path: /tmp }
        - { path: /var/tmp }

    - name: Enforce umask globally
      copy:
        dest: /etc/profile.d/secure-umask.sh
        content: "umask 027\n"
        mode: '0644'

    - name: Remove SUID from known binaries
      file:
        path: "{{ item }}"
        mode: u-s
      loop:
        - /usr/bin/at
        - /usr/bin/newgrp

    - name: Apply default ACL on project dir
      command: >
        setfacl -d -m g::rwx /srv/project
      args: { creates: "/srv/project/.acl-default-tag" }
```

---

## 운영 체크리스트 — 파일/디렉터리

- [ ] **world-writable + sticky 없음** 디렉터리 0건.
- [ ] `/tmp`, `/var/tmp`는 **tmpfs + nodev/nosuid/noexec + 1777**.
- [ ] SUID/SGID **최소화**, Capabilities 대체 목록 갱신.
- [ ] **umask 027/077**(서비스는 `UMask=007` 등 정책적용).
- [ ] 협업 디렉터리 = **SGID + 기본 ACL**.
- [ ] `/data`,`/uploads` 등 **noexec,nodev,nosuid**.
- [ ] **링크 보호 sysctl=1/2**, 서비스 **PrivateTmp=true**.
- [ ] 로그/백업 권한(0640/0750) + **logrotate** + 중앙전송.
- [ ] AIDE 주기검사 + **auditd** 디렉터리 write/attr 감시.
- [ ] NFS `root_squash` + 클라 `nosuid,nodev,noexec`.

---

## 미니 랩 A — “/tmp 하드닝 & 링크 보호”

1) `/etc/fstab`로 `/tmp`/`/var/tmp` tmpfs 보안 옵션 적용 후 `mount -a`.
2) `sysctl`로 링크 보호 활성.
3) 심링크 공격 PoC(랩 전용) 시도 → 차단 확인.
4) 서비스 `PrivateTmp=true` 적용 비교.

---

## 미니 랩 B — “SUID 제거 + Capabilities 대체”

1) SUID 목록 산출 → 기능 확인.
2) **포트 바인드** 필요 바이너리를 `setcap cap_net_bind_service=+ep`로 대체.
3) AIDE로 변경 감지 확인.

---

## 권한 드리프트 제거”

1) `/srv/project`에 `chmod 2770`, **기본 ACL** 설정.
2) 서로 다른 사용자가 생성해도 그룹/권한이 일정한지 확인.
3) `UMask=007` 서비스에서 파일 생성 시 권한 검증.

---

## 예상문제(필기/실무)

1) **Sticky 비트**의 기능과 `/tmp`에서 필요한 이유를 설명하고, 권한 표기로 나타내라.
2) `noexec/nodev/nosuid`가 무엇을 의미하는지 각각 설명하고, 적용이 필요한 3개 경로를 쓰라.
3) **SUID/SGID** 파일의 위험성과 **Capabilities** 대체 예를 하나 쓰라.
4) **umask 027**일 때 기본 파일/디렉터리 **최종 권한**을 계산하라.
   - 힌트: 파일 기본 `666`, 디렉터리 기본 `777`, \(E=R\&\sim U\).
5) 링크 기반 공격을 차단하는 **sysctl** 3개를 쓰라.
6) 협업 디렉터리에서 **SGID**와 **기본 ACL**을 설정하는 이유를 설명하라.
7) logrotate에서 **심링크 공격**을 피하기 위한 주요 설정 3가지를 쓰라.

예시 스니펫:
```bash
# sticky 비트 확인

ls -ld /tmp        # drwxrwxrwt  (t)
```
```bash
# fstab 예

/dev/vdb1 /data ext4 defaults,noexec,nodev,nosuid 0 2
```
```bash
# sysctl

sysctl fs.protected_symlinks=1
sysctl fs.protected_hardlinks=1
sysctl fs.protected_fifos=2
```

---

## 결론

- 파일/디렉터리 보안은 **권한(기본/특수/ACL)**, **마운트옵션**, **링크/레이스 방어**, **무결성/감사**가 **동시에** 굴러갈 때 효과가 난다.
- 본 문서의 **점검 스크립트→수정 절차→운영 체크리스트**를 **IaC(Ansible)** 로 고정하면, “한 번 고치고 계속 유지”하는 **지속 가능한 보안 운영**을 달성할 수 있다.
