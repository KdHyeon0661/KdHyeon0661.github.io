---
layout: post
title: 정보보안기사 - UNIX/Linux 기본 학습
date: 2025-11-05 21:25:23 +0900
category: 정보보안기사
---
# SECTION 01 시스템 기본 학습 — 02. UNIX/Linux 기본 학습

## 학습 목표와 실습 환경

- **사용자/커널 공간**, **프로세스·신호·파일 디스크립터**, **권한(UGO/SGID/SUID/Sticky)·ACL·Capabilities**, **패키지/서비스(systemd)**, **로깅(journald·syslog)·감사(auditd)**, **네트워킹(ip/ss/iptables/nftables)**, **스토리지(LVM/RAID/mount)**를 이해하고 실습한다.
- 보안 기본수칙: **최소권한**, **변경 이력/로그 가시성**, **구성의 코드화(IaC)**.

권장 환경: 최신 **Ubuntu Server 22.04+** 또는 **RHEL 8/9, Rocky/Alma**, **Debian 12**, 각자에 맞는 패키지 매니저(apt/dnf).  
루트 권한: `sudo -i` 또는 명령 앞에 `sudo`.

---

## UNIX/Linux 구조 한눈에

### 사용자 공간 vs 커널 공간
- 사용자 공간: 셸·앱·라이브러리(glibc), 시스템콜로 커널과 통신.
- 커널 공간: 프로세스 스케줄링, 메모리 관리, 파일시스템, 네트워킹, 보안(LSM: SELinux/AppArmor).

핸즈온(시스템콜 트레이스 미리보기):
```bash
strace -f -e trace=openat,execve -p $(pidof sshd) 2>&1 | head
```

### 프로세스/스레드/신호
- PID, PPID, 상태(R, S, D, Z), 우선순위/NI, cgroup.
- 신호: `SIGTERM(15)`, `SIGKILL(9)`, `SIGHUP(1)` 등.

핸즈온(프로세스/신호):
```bash
ps -eo pid,ppid,ni,stat,etime,cmd --sort=-etime | head
kill -TERM <pid>      # 정상 종료 시도
kill -KILL <pid>      # 강제 종료
```

---

## 파일·디렉터리·권한 모델

### 권한 비트와 소유(UGO)
- **소유자(user) / 그룹(group) / 기타(other)** 각각에 rwx(4,2,1).
- 숫자표현: `chmod 640 file` → 사용자(rw=6), 그룹(r=4), 기타(0).

수학적 표현(권한의 8진수 합계):
$$
\text{octal} = (4\cdot r_u + 2\cdot w_u + 1\cdot x_u)\,\|\, (4\cdot r_g + 2\cdot w_g + 1\cdot x_g)\,\|\, (4\cdot r_o + 2\cdot w_o + 1\cdot x_o)
$$

### 특수 비트: SUID·SGID·Sticky
- **SUID(4xxx)**: 실행 시 파일 소유자 권한으로 실행(예: `/usr/bin/passwd`).  
- **SGID(2xxx)**: 실행 시 파일 **그룹** 권한 적용, 디렉터리에 설정하면 **하위 파일의 그룹 상속**.  
- **Sticky(1xxx)**: 디렉터리에서 **소유자만 삭제 가능**(`/tmp`).

핸즈온(공유 디렉터리 설계: SGID+Sticky)
```bash
groupadd proj
mkdir -p /srv/proj
chgrp proj /srv/proj
chmod 2770 /srv/proj          # 2(SGID) + 770
chmod +t /srv/proj            # sticky 추가 → 3770
getfacl /srv/proj
```

### umask와 기본 권한
- `umask`는 **기본 권한에서 뺄 값**. 일반적으로 022(파일 644, 디렉터리 755), 보안환경은 027/077 권장.

핸즈온(강화된 umask 적용):
```bash
umask
echo 'umask 027' | sudo tee -a /etc/profile.d/hardening.sh
```

### POSIX ACL과 Capabilities
- ACL: `setfacl/getfacl`로 사용자/그룹별 세밀 제어(UGO를 보완).  
- Capabilities: 루트 전체 권한 대신 특정 권능만 부여(`cap_net_bind_service` 등).

핸즈온(ACL/Capabilities):
```bash
setfacl -m u:alice:rwx /srv/proj
getfacl /srv/proj

# 1024 미만 포트 바인딩 권한만 부여(루트 불필요)
setcap 'cap_net_bind_service=+ep' /usr/local/bin/myservice
getcap /usr/local/bin/myservice
```

체크리스트
- [ ] 민감 디렉터리는 **상속·SGID**로 그룹 일관성 유지.  
- [ ] `/tmp`와 같은 공유 디렉터리는 **sticky bit** 필수.  
- [ ] 기본 umask는 027/077 검토.  
- [ ] 루트 권한 대신 **capabilities** 사용 가능 여부 확인.

---

## 파일시스템·마운트·로그 구조(FHS)

### FHS(파일 시스템 계층 구조)
- `/etc`(설정), `/var/log`(로그), `/var/lib`(상태), `/usr`(읽기전용 앱/라이브러리), `/opt`(서드파티), `/home`(사용자 홈).

핸즈온(마운트/영구 설정):
```bash
lsblk -f
# 임시 마운트
mount /dev/vdb1 /data
# 영구 설정(fstab)
echo '/dev/vdb1  /data  ext4  defaults,noexec,nodev,nosuid  0  2' | sudo tee -a /etc/fstab
mount -a
```
보안 마운트 옵션:
- `noexec`(바이너리 실행 차단), `nodev`(디바이스 파일 무효), `nosuid`(SUID/SGID 무효).

핸즈온(로그 확인/회전):
```bash
# journald
journalctl -p warning -S -2h
# syslog/rsyslog
grep -E 'error|fail|denied' /var/log/syslog 2>/dev/null | tail
# logrotate 수동 테스트
logrotate -d /etc/logrotate.conf
```

체크리스트
- [ ] 데이터/임시/컨테이너 경로는 `noexec,nodev,nosuid` 검토.  
- [ ] logrotate 주기/보존/압축/권한 점검.  
- [ ] journald 영속(`/var/log/journal`) 및 크기 상한 설정.

---

## 사용자/그룹/PAM/암호 정책

### 사용자/그룹 관리
```bash
id alice
useradd -m -G proj alice
passwd alice
chage -l alice      # 암호 만료 정보
```

### PAM과 로그인 정책
- **PAM**(Pluggable Authentication Modules): `/etc/pam.d/` 설정.  
- **암호/락아웃**: `pam_pwquality`, `pam_faillock` 등으로 정책 적용.  
- 로그인 기본 정책: `/etc/login.defs`.

핸즈온(로그인 실패 잠금 예: RHEL8+/Rocky/Alma)
```bash
# faillock 구성(예시, 배포판 가이드에 맞춰 적용)
authselect select sssd with-faillock
# 확인
faillock --user alice
```

체크리스트
- [ ] 기본 계정(예: `root` 원격 로그인) 제한.  
- [ ] 실패 잠금/백오프, 비밀번호 복잡성, 만료 정책 적용.  
- [ ] `sudo` 최소권한 부여, TTY 제한/로깅.

---

## 셸·도구 체인과 텍스트 처리

### 셸과 환경
- Bash/Zsh, 프로파일: `/etc/profile`, `~/.profile`, `~/.bashrc`.  
- 경로/변수: `$PATH`, `export VAR=value`.

핸즈온(환경 변수/영구화):
```bash
export EDITOR=vim
echo 'export EDITOR=vim' >> ~/.bashrc
```

### 텍스트 도구: grep/sed/awk/find/xargs
```bash
# 에러 로그 추출
grep -iE 'error|fail' /var/log/syslog | tail -n 50

# sed로 설정 값 바꾸기(백업 생성)
sed -i.bak 's/^#?MaxAuthTries.*/MaxAuthTries 3/' /etc/ssh/sshd_config

# awk로 컬럼 집계
ss -tan | awk '/ESTAB/ {print $4}' | cut -d: -f1 | sort | uniq -c | sort -nr | head

# find + xargs: 최근 7일 수정/권한 점검
find /var/www -type f -mtime -7 -print0 | xargs -0 ls -l | awk '{print $1,$3,$4,$9}' | head
```

체크리스트
- [ ] 기본 텍스트 파이프라인(grep/sed/awk)으로 **로그를 요약**할 수 있다.  
- [ ] `find -print0 | xargs -0`로 **공백/특수문자 안전 처리**.

---

## 패키지·서비스(systemd)·타이머

### 패키지 매니저
```bash
# Debian/Ubuntu
apt update && apt install -y htop jq
# RHEL/Rocky/Alma
dnf makecache && dnf install -y htop jq
```

### systemd 서비스 관리
```bash
systemctl status sshd
systemctl enable --now sshd
systemctl disable --now avahi-daemon
systemctl list-unit-files --type=service | grep enabled
journalctl -u sshd -S -2h
```

### 사용자 서비스/타이머 예시(앱 헬스체크)
```bash
# /etc/systemd/system/app-health.service
[Unit]
Description=App health probe

[Service]
Type=oneshot
ExecStart=/usr/local/bin/app_health.sh

# /etc/systemd/system/app-health.timer
[Unit]
Description=Run app health every 1min

[Timer]
OnUnitActiveSec=60
Unit=app-health.service

[Install]
WantedBy=timers.target
```
```bash
systemctl daemon-reload
systemctl enable --now app-health.timer
systemctl list-timers --all | grep app-health
```

체크리스트
- [ ] 불필요 서비스 비활성화/제거.  
- [ ] 중요한 서비스는 `Restart=on-failure` 등 복구 전략 설정.  
- [ ] 타이머로 **정기 점검/백업/로테이션** 자동화.

---

## 네트워킹 기본: 인터페이스·주소·라우팅·세션

### 인터페이스/주소/라우팅
```bash
ip addr show
ip route
resolvectl status 2>/dev/null || cat /etc/resolv.conf
```

### 연결/포트 관찰: ss/lsof
```bash
ss -tulpn | head -n 20
lsof -i -nP | head -n 20
```

### 방화벽: nftables/iptables, ufw/firewalld
- **RHEL/Alma/Rocky** 기본: `firewalld`(nftables backend).  
- **Ubuntu**: `ufw` 또는 `nftables` 직접 사용 권장.

핸즈온(firewalld 예)
```bash
firewall-cmd --get-active-zones
firewall-cmd --zone=public --add-service=ssh --permanent
firewall-cmd --reload
firewall-cmd --list-all
```

핸즈온(ufw 예)
```bash
ufw default deny incoming
ufw default allow outgoing
ufw allow from 10.0.0.0/24 to any port 22 proto tcp
ufw enable
ufw status verbose
```

핸즈온(nftables 최소 정책)
```bash
cat >/etc/nftables.conf <<'EOF'
table inet filter {
  chain input {
    type filter hook input priority 0;
    ct state established,related accept
    iif lo accept
    tcp dport { 22 } accept
    ip protocol icmp accept
    ip6 nexthdr icmpv6 accept
    drop
  }
}
EOF
nft -f /etc/nftables.conf
systemctl enable --now nftables
```

체크리스트
- [ ] 인바운드 **기본 차단**, 필요한 서비스만 허용.  
- [ ] 관리 포트(SSH)는 **원천지 제한/MFA/키 인증**.  
- [ ] 불필요한 리스닝 포트 제거.

---

## SSH 보안과 운영

### 키 기반 인증/서버 하드닝
```bash
# 키 생성(클라이언트)
ssh-keygen -t ed25519 -C "ops@corp"
ssh-copy-id -i ~/.ssh/id_ed25519.pub alice@server

# 서버 구문(예: /etc/ssh/sshd_config)
PermitRootLogin no
PasswordAuthentication no
PubkeyAuthentication yes
MaxAuthTries 3
AllowUsers alice opssvc
```
```bash
systemctl reload sshd
```

체크리스트
- [ ] 루트 SSH 금지, 패스워드 인증 비활성.  
- [ ] 사용자 화이트리스트/개별 계정 키 관리.  
- [ ] 포트 포워딩/에이전트 포워딩 제한 필요 시 설정.

---

## 스토리지: 디스크·RAID·LVM·스냅샷

### 파티션/파일시스템
```bash
lsblk -f
parted -l
mkfs.ext4 /dev/vdb1
```

### LVM (물리→볼륨그룹→논리볼륨)
```bash
pvcreate /dev/vdb1
vgcreate vgdata /dev/vdb1
lvcreate -n lvlogs -L 10G vgdata
mkfs.xfs /dev/vgdata/lvlogs
mkdir -p /var/log2
echo '/dev/vgdata/lvlogs /var/log2 xfs defaults 0 2' >> /etc/fstab
mount -a
```

### 스냅샷(빠른 롤백/백업 시)
```bash
lvcreate -s -n snap_logs -L 1G /dev/vgdata/lvlogs
# 롤백 시:
umount /var/log2
lvconvert --merge /dev/vgdata/snap_logs
mount -a
```

체크리스트
- [ ] 데이터/로그 용량 모니터링, **LVM 스냅샷**으로 안전한 변경 테스트.  
- [ ] fstab 보안 옵션과 **정기 파일시스템 체크**.

---

## 로깅·감사(auditd)·문제 해결

### journald/syslog
```bash
journalctl -p err -S today
journalctl -u sshd --since "2025-11-11 00:00:00"
```

### auditd(보안 이벤트 추적)
```bash
# 특정 파일 접근 감사 룰
auditctl -w /etc/ssh/sshd_config -p wa -k sshcfg
# 검색
ausearch -k sshcfg --success no
```

### 성능/상태 진단
```bash
top -o %MEM
vmstat 1 5
iostat -xz 1 3
free -h
du -sh /var/* | sort -h | tail
dmesg | tail -n 50
```

체크리스트
- [ ] 서비스 단위 로그(`journalctl -u`)로 원인 근접.  
- [ ] audit 룰은 **명확한 키(-k)**로 검색용 태깅.  
- [ ] 디스크/메모리/IO 지표를 주기적으로 확인.

---

## 미니 랩 A — 팀 공유 디렉터리 보안(ACL/SGID/Sticky)

목표: `/srv/proj`에 팀원은 **읽기/쓰기**, 타인은 접근 불가. 하위는 **자동 그룹 상속**, 삭제는 **소유자만**.

절차:
```bash
groupadd proj
mkdir -p /srv/proj
chgrp proj /srv/proj
chmod 2770 /srv/proj         # SGID
chmod +t /srv/proj           # sticky
setfacl -m g:proj:rwx /srv/proj
```
검증:
```bash
sudo -u alice touch /srv/proj/x.txt
ls -l /srv/proj               # 그룹이 proj로 상속되는지 확인
```

---

## 미니 랩 B — SSH/방화벽/로그 묶음 구성

목표: SSH 키 인증, 포트 22은 내부망만, 실패 로그를 쿼리.

절차:
```bash
# sshd_config 강화(본문 예시)
systemctl reload sshd

# 방화벽(ufw 예)
ufw default deny incoming
ufw allow from 10.0.0.0/24 to any port 22 proto tcp
ufw enable

# 로그인 실패 분석
journalctl -u sshd -S -24h | grep -i "failed" | tail -n 50
```

---

## 미니 랩 C — systemd 타이머로 백업 자동화

목표: `/etc` 백업을 매일 로테이션.

스크립트:
```bash
install -d -m 750 /opt/backup
cat >/usr/local/bin/backup_etc.sh <<'EOF'
#!/usr/bin/env bash
set -euo pipefail
ts="$(date +%F_%H%M)"
tar -C / -czf /opt/backup/etc_${ts}.tar.gz etc
find /opt/backup -name 'etc_*.tar.gz' -mtime +14 -delete
EOF
chmod +x /usr/local/bin/backup_etc.sh
```
유닛/타이머:
```bash
cat >/etc/systemd/system/backup-etc.service <<'EOF'
[Unit]
Description=Backup /etc to /opt/backup

[Service]
Type=oneshot
ExecStart=/usr/local/bin/backup_etc.sh
EOF

cat >/etc/systemd/system/backup-etc.timer <<'EOF'
[Unit]
Description=Daily backup of /etc

[Timer]
OnCalendar=daily
Persistent=true

[Install]
WantedBy=timers.target
EOF

systemctl daemon-reload
systemctl enable --now backup-etc.timer
systemctl list-timers | grep backup-etc
```

---

## 흔한 실수와 방지

- **775/777 남발**: 기본 umask가 느슨해 내부 유출/변조 발생 → 027/077로 조정.  
- **/tmp 악용**: sticky 미설정 시 타인 파일 삭제 가능 → sticky 확인.  
- **SSH 패스워드 허용**: 무차별 대입 위험 → 키 인증·원천지 제한·Fail2ban.  
- **서비스 과다**: 불필요 데몬 상시 리스닝 → disable/remove, 포트 감시(ss/lsof).  
- **무로그/과로깅**: 사고 재현 불가 또는 디스크 포화 → logrotate/journald 한도 설정.  
- **루트 남용**: 루트 쉘 상시 사용 → `sudo` 최소권한·capabilities 사용.

---

## 최종 체크리스트

- [ ] **권한 모델**(UGO/SUID/SGID/Sticky/ACL/Capabilities)을 설명·구성할 수 있다.  
- [ ] **umask 027/077** 등 보안 기본권한을 적용했다.  
- [ ] **FHS**와 주요 경로(`/etc`, `/var/log`, `/usr`, `/opt`)의 역할을 구분한다.  
- [ ] **systemd**로 서비스/타이머/로그를 관리할 수 있다.  
- [ ] **SSH 하드닝**(키 인증, 루트 금지, 원천지 제한)을 수행했다.  
- [ ] **nftables/ufw/firewalld**로 인바운드 기본 차단 정책을 만든다.  
- [ ] **journalctl/auditd**로 근거 로그를 추출·보존할 수 있다.  
- [ ] **LVM 스냅샷**으로 변경 전후 롤백을 시연할 수 있다.  
- [ ] 텍스트 도구 체인으로 **로그 요약·필터**를 자동화했다.

---

## 예상문제(필기+실기형)

1) **SGID와 Sticky의 차이**와 적합한 사용 시나리오를 설명하라.  
2) `/srv/shared`에 대해 팀 그룹 `eng`에게만 rwx, 타인 접근 불가, 하위 그룹 자동 상속, 소유자만 삭제 가능하도록 **명령 4줄**을 작성하라.  
3) **SSH**에서 패스워드 인증을 금지하고 키만 허용하며, 루트 로그인을 막는 `sshd_config` 지시문 3개를 제시하라.  
4) `/data`를 `noexec,nodev,nosuid`로 마운트하려면 `/etc/fstab`에 어떤 행을 추가해야 하는가(파일시스템은 ext4, 장치는 `/dev/vdb1`)?  
5) 최근 2시간의 **sshd** 경고 이상 로그만 `journalctl`로 출력하는 명령을 작성하라.  
6) `/usr/local/bin/httpd-lite`가 80/TCP 바인딩이 필요하다. 루트 없이 이를 허용할 **capability 설정**과 검증 명령을 쓰라.

예시 답안 스니펫:
```bash
# Q2
groupadd eng
mkdir -p /srv/shared
chgrp eng /srv/shared
chmod 2770 /srv/shared
chmod +t /srv/shared
```
```text
# Q3 (sshd_config)
PermitRootLogin no
PasswordAuthentication no
PubkeyAuthentication yes
```
```bash
# Q4
echo '/dev/vdb1 /data ext4 defaults,noexec,nodev,nosuid 0 2' >> /etc/fstab
mount -a
```
```bash
# Q5
journalctl -u sshd -p warning -S -2h
```
```bash
# Q6
setcap 'cap_net_bind_service=+ep' /usr/local/bin/httpd-lite
getcap /usr/local/bin/httpd-lite
```

---

## 요약

본 장을 통해 다음을 달성했다.
- 리눅스의 **권한·파일시스템·서비스·로깅·네트워킹** 핵심을 **보안 우선** 관점으로 익혔다.  
- 실습으로 **SGID/Sticky/ACL/capabilities**, **systemd 타이머**, **SSH/방화벽 하드닝**을 구성했다.  
- 이 토대를 기반으로 다음 장(시스템 관리·서버 보안·취약점 점검)에서 **정책화·자동화**를 확장한다.