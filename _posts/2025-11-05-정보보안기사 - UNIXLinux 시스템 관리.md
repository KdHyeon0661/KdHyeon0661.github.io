---
layout: post
title: 정보보안기사 - UNIX/Linux 시스템 관리
date: 2025-11-05 22:25:23 +0900
category: 정보보안기사
---
# SECTION 01 시스템 기본 학습 — 03. UNIX/Linux 시스템 관리

## 운영자 베이스라인(핵심 요약)

- 사용자는 **최소권한 원칙**, 역할별 그룹/`sudoers` 분리, `pam_faillock`·`pam_pwquality`.
- 서비스는 **systemd 유닛**으로 관리하고, **오버라이드**·`Restart=`·`Sandboxing` 적용.
- 파일시스템은 **LVM/RAID**로 유연히 운영하고, **noexec/nodev/nosuid**·`logrotate`·쿼터를 적용.
- 네트워크는 **기본 차단**(ufw/firewalld/nftables), SSH 키 인증, 원천지 제한.
- 로깅은 **journald + rsyslog(선택)**, 보안은 **auditd**로 근거 확보.
- **패치 자동화**와 **백업/스냅샷**은 운영의 생명선.
- 장애는 **관측→격리→가설→검증** 루프와 **복구 절차 문서화**로 빠르게 해결.

---

## 사용자/그룹/권한 관리

### 계정 수명주기

```bash
# 생성과 기본 그룹 지정

useradd -m -s /bin/bash -G dev alice
passwd alice

# 만료/비활성화/잠금

chage -l alice
chage -M 90 -W 14 -I 7 alice   # 최대 90일, 만료 14일 전 경고, 유예 7일
usermod -L alice               # 잠금
```

### sudo 정책(권한 위임)

```bash
visudo   # /etc/sudoers 혹은 /etc/sudoers.d/*

# 예: dev 그룹은 특정 명령만 비암호로 허용

%dev ALL=(root) NOPASSWD: /usr/bin/systemctl restart web.service,/usr/bin/journalctl -u web
```

검증과 점검
```bash
id alice
sudo -l -U alice
```

체크리스트
- [ ] 운영 계정은 **개인계정 + 그룹**으로 추적 가능.
- [ ] `sudoers`는 **최소 명령 단위**로 허용, 와일드카드 남용 금지.
- [ ] 계정 **만료/퇴사 처리** 자동화(타이머/Ansible).

---

## PAM과 로그인 정책(보안·추적)

### 암호 품질/락아웃

Debian/Ubuntu:
```bash
apt install -y libpam-pwquality
# /etc/pam.d/common-password 내 예시

password requisite pam_pwquality.so retry=3 minlen=12 ucredit=-1 lcredit=-1 dcredit=-1 ocredit=-1
```

RHEL/Rocky/Alma(락아웃):
```bash
# 권장 프로파일 선택

authselect select sssd with-faillock --force
# 확인

faillock --user alice
```

`/etc/login.defs` 기본 정책
```bash
PASS_MAX_DAYS   90
PASS_MIN_DAYS   1
PASS_WARN_AGE   14
```

체크리스트
- [ ] 최소 길이/문자종류/재사용 방지 설정.
- [ ] 연속 실패 **락아웃** 및 **백오프**.
- [ ] `last`, `lastb`, `journalctl -u sshd`로 로그인 이력 점검.

---

## 패키지/리포지토리/패치 관리

### apt/dnf 기본

```bash
# Debian/Ubuntu

apt update && apt upgrade -y
apt install -y htop jq curl

# RHEL 계열

dnf makecache && dnf upgrade -y
dnf install -y htop jq curl
```

### 자동 업데이트

Ubuntu:
```bash
apt install -y unattended-upgrades
dpkg-reconfigure -plow unattended-upgrades
```
RHEL:
```bash
dnf install -y dnf-automatic
sed -i 's/^apply_updates = .*/apply_updates = yes/' /etc/dnf/automatic.conf
systemctl enable --now dnf-automatic.timer
```

체크리스트
- [ ] **보안 업데이트**는 자동, 커널/대규모는 **점검 창** 내 수동 롤링 적용.
- [ ] 내부 **미러/리포지토리 핀닝**(검증된 출처) 사용.

---

## systemd: 서비스·타이머·오버라이드·샌드박스

### 서비스 관리

```bash
systemctl status nginx
systemctl enable --now nginx
systemctl restart nginx
journalctl -u nginx -S -2h
```

### 사용자 정의 유닛

`/etc/systemd/system/web.service`:
```ini
[Unit]
Description=Flask Web
After=network-online.target
Wants=network-online.target

[Service]
Type=simple
User=web
Group=web
WorkingDirectory=/srv/web
ExecStart=/usr/bin/python3 app.py
Restart=always
RestartSec=3

# 리소스 제한

LimitNOFILE=65535
MemoryMax=512M
CPUQuota=80%

# 보안 샌드박스

NoNewPrivileges=true
ProtectSystem=strict
ProtectHome=true
PrivateTmp=true
ProtectKernelTunables=true
RestrictSUIDSGID=true
RestrictNamespaces=true
ReadOnlyPaths=/etc
ReadWritePaths=/srv/web /var/log/web

[Install]
WantedBy=multi-user.target
```
적용:
```bash
systemctl daemon-reload
systemctl enable --now web
```

### 오버라이드(벤더 유닛 변경 없이)

```bash
systemctl edit nginx
# [Service] 블록에 Restart=always 등 추가 후 저장

```

### 타이머(주기 작업)

`/etc/systemd/system/db-backup.service`:
```ini
[Service]
Type=oneshot
ExecStart=/usr/local/bin/backup_db.sh
```
`/etc/systemd/system/db-backup.timer`:
```ini
[Timer]
OnCalendar=*-*-* 02:30:00
Persistent=true
Unit=db-backup.service
[Install]
WantedBy=timers.target
```
```bash
systemctl daemon-reload
systemctl enable --now db-backup.timer
systemctl list-timers | grep db-backup
```

체크리스트
- [ ] 서비스는 **Restart 정책**과 **리소스 상한** 필수.
- [ ] `ProtectSystem/PrivateTmp/NoNewPrivileges` 등 **샌드박스 지시자** 적용.
- [ ] 크론 대신 **타이머**로 표준화.

---

## 파일시스템/스토리지: 파티션·LVM·RAID·마운트 옵션·쿼터

### 파티션/FS 생성

```bash
lsblk -f
parted /dev/vdb mklabel gpt
parted /dev/vdb mkpart primary ext4 1MiB 100%
mkfs.ext4 /dev/vdb1
mkdir -p /data
echo '/dev/vdb1 /data ext4 defaults,noexec,nodev,nosuid 0 2' >> /etc/fstab
mount -a
```

### LVM 확장

```bash
pvcreate /dev/vdc
vgextend vg0 /dev/vdc
lvextend -r -L +20G /dev/vg0/lvdata   # -r은 온라인 FS 확장
```

### 소프트웨어 RAID(md)

```bash
mdadm --create /dev/md0 --level=1 --raid-devices=2 /dev/vdd1 /dev/vde1
mkfs.xfs /dev/md0
```

### 디스크 쿼터

```bash
# /etc/fstab: usrquota,grpquota 추가

/dev/vg0/lvhome /home xfs defaults,usrquota,grpquota 0 2
mount -o remount /home
xfs_quota -x -c 'limit bsoft=10G bhard=12G alice' /home
xfs_quota -x -c 'report -h' /home
```

체크리스트
- [ ] 데이터/임시 경로에 **noexec,nodev,nosuid**.
- [ ] **LVM**으로 유연성 확보, **쿼터**로 오남용 방지.
- [ ] **RAID1/10** 등 내결함성 계획.

---

## 네트워킹: IP/DNS/라우팅/관리도구

### 인터페이스/라우팅/DNS

```bash
ip addr show
ip route
resolvectl status 2>/dev/null || cat /etc/resolv.conf
```

### Netplan(Ubuntu) 예

`/etc/netplan/01-prod.yaml`:
```yaml
network:
  version: 2
  ethernets:
    enp0s3:
      dhcp4: no
      addresses: [10.0.0.10/24]
      gateway4: 10.0.0.1
      nameservers:
        addresses: [10.0.0.2, 1.1.1.1]
```
```bash
netplan apply
```

### nmcli(RHEL) 예

```bash
nmcli con add type ethernet ifname eth0 con-name prod ip4 10.0.0.20/24 gw4 10.0.0.1
nmcli con mod prod ipv4.dns "10.0.0.2 1.1.1.1" ipv4.method manual
nmcli con up prod
```

### 방화벽

firewalld(RHEL):
```bash
firewall-cmd --get-active-zones
firewall-cmd --zone=public --add-service=ssh --permanent
firewall-cmd --reload
```
ufw(Ubuntu):
```bash
ufw default deny incoming
ufw default allow outgoing
ufw allow from 10.0.0.0/24 to any port 22 proto tcp
ufw enable
ufw status verbose
```

체크리스트
- [ ] **고정 IP**·DNS·GW 정확히 설정, 재부팅 후 유지 검증.
- [ ] 인바운드 **기본 차단**, SSH는 **원천지 제한**.

---

## 성능·상태 모니터링: CPU/메모리/IO/네트워크

### 즉시 관측

```bash
top -o %CPU
free -h
vmstat 1 5
iostat -xz 1 3
ss -s
```

### sar(시계열)

```bash
apt install -y sysstat  # Debian/Ubuntu
dnf install -y sysstat  # RHEL
systemctl enable --now sysstat
sar -q 1 5     # load average
sar -n DEV 1 5 # 인터페이스별 네트워크
```

### 병목 단서

- CPU 100%: 실제 사용 vs IO wait 구분(`%wa`).
- 메모리 압박: swap in/out, OOM 로그 확인(`dmesg`).
- IO: `iostat -xz`의 `await`/`util`.
- 네트워크: `drops`/`retransmits`/큐 포화.

체크리스트
- [ ] **기저 라인**(정상 시 지표)을 저장해 이상 탐지.
- [ ] 서비스 단위 로그와 시스템 지표의 **상관관계**를 본다.

---

## 로깅/회전/중앙수집

### journald

```bash
journalctl -p warning -S -2h
journalctl -u web --since "2025-11-11 00:00:00"
```
`/etc/systemd/journald.conf` 주요 옵션:
```
Storage=persistent
SystemMaxUse=1G
MaxRetentionSec=2w
```

### rsyslog(선택)

```bash
# 원격 전송 예시

*.* @@log.example.corp:514
systemctl restart rsyslog
```

### logrotate

```bash
logrotate -d /etc/logrotate.conf     # 드라이런
cat /etc/logrotate.d/nginx
```

체크리스트
- [ ] journald **영속** 저장과 **용량 상한** 설정.
- [ ] 애플리케이션 로그 파일은 **logrotate** 정책 적용.
- [ ] 중앙 수집(SIEM/ELK/Graylog) 연계.

---

## 감사(auditd): 설정과 검색

### 규칙 추가/검색

```bash
apt install -y auditd audispd-plugins || dnf install -y audit
auditctl -w /etc/sudoers -p wa -k sudoers
ausearch -k sudoers --success no
```
영구 규칙: `/etc/audit/rules.d/*.rules`에 작성 후
```bash
augenrules --load
```

체크리스트
- [ ] 인증/권한/핵심 설정 파일에 **감사 룰**.
- [ ] 키(`-k`)로 검색 태깅.

---

## 커널/리밋/sysctl(네트워크/파일/메모리)

### 파일 디스크립터

```bash
ulimit -n
echo '* soft nofile 65535' >> /etc/security/limits.conf
echo '* hard nofile 65535' >> /etc/security/limits.conf
```

### sysctl

`/etc/sysctl.d/99-tuning.conf`:
```conf
# 네트워크 감소 공격 방어·성능

net.ipv4.tcp_syncookies = 1
net.ipv4.tcp_fin_timeout = 30
net.ipv4.ip_forward = 0
net.ipv4.conf.all.rp_filter = 1
net.ipv4.conf.all.accept_redirects = 0
net.ipv4.conf.all.send_redirects = 0

# 파일 핸들 수

fs.file-max = 2097152
```
```bash
sysctl --system
```

체크리스트
- [ ] 서비스에 맞춘 **nofile/stack/core** 등 ulimit.
- [ ] 불필요 **IP 포워딩/리다이렉트** 비활성.

---

## 시간 동기화와 타임존

### chrony/systemd-timesyncd

```bash
timedatectl set-timezone Asia/Seoul
apt install -y chrony || dnf install -y chrony
systemctl enable --now chronyd
chronyc sources -v
```

체크리스트
- [ ] **단일 표준 시간원**(내부 NTP)과 **로그 시각 일관성**.

---

## 백업/복구: rsync/tar/borg + LVM 스냅샷

### rsync 증분 백업

```bash
rsync -aHAX --delete /srv/web/ /backup/web/$(date +%F)/
```

### tar 스냅샷

```bash
tar --xattrs --acls -czf /backup/etc_$(date +%F).tar.gz /etc
```

### LVM 스냅샷 롤백

```bash
lvcreate -s -n snap_web -L 2G /dev/vg0/lvweb
# 패치/배포 → 문제시 롤백

umount /srv/web
lvconvert --merge /dev/vg0/snap_web
mount -a
```

체크리스트
- [ ] 백업은 **오프사이트/불변 저장소**와 **정기 복구 리허설** 포함.
- [ ] 민감 데이터는 **암호화**(예: `restic`, `borg`).

---

## 자동화: cron/at → systemd timers, 그리고 Ansible

### cron → 타이머 전환

```bash
crontab -l
# 가능하면 동일 로직을 systemd timer로 이관(상태/로그 일원화)

```

### 간단 Ansible 예(패키지·서비스)

`site.yml`:
```yaml
- hosts: app
  become: true
  tasks:
    - name: Install packages
      package:
        name: [nginx, git]
        state: present
    - name: Deploy config
      template:
        src: nginx.conf.j2
        dest: /etc/nginx/nginx.conf
      notify: reload nginx
  handlers:
    - name: reload nginx
      service:
        name: nginx
        state: reloaded
```

체크리스트
- [ ] **구성의 코드화**(IaC)로 재현성/감사성 확보.
- [ ] 민감 값은 **비밀관리**(Ansible Vault/외부 KMS).

---

## 부팅/복구: GRUB, initramfs, SELinux/AppArmor

### 부팅 문제 공략

- GRUB 편집: 커널 라인에 `systemd.unit=rescue.target` 부팅.
- initramfs 재생성:
```bash
update-initramfs -u        # Debian/Ubuntu
dracut -f                  # RHEL
```

### SELinux/AppArmor 기본 복구

- SELinux 컨텍스트 복구:
```bash
# RHEL

touch /.autorelabel
reboot
```
- AppArmor 프로파일 상태:
```bash
aa-status
```

체크리스트
- [ ] 커널/드라이버/파일시스템 오류는 **rescue** 모드에서 접근.
- [ ] 보안 프레임워크(SELinux/AppArmor)와 권한 오류 구분.

---

## 트러블슈팅 루틴(관측 → 가설 → 검증)

1) 증상 수집: `journalctl -xe`, `systemctl status`, `dmesg`, 지표(sar/top/iostat).
2) 격리: 포트 차단, 복제 환경 재현.
3) 가설 세우기: 최근 변경(배포/패치/구성) 비교.
4) 실험/검증: 오버라이드 제거/기본값 적용/버전 롤백.
5) 복구/회고: RCA 문서화, 재발 방지(헬스체크/알람/가드).

---

## 미니 랩 1 — 안전한 웹 서비스 유닛 만들기

목표: Flask 앱을 **샌드박스 systemd**로 실행, 포트/파일 제한.

절차 요약:
1) `web` 사용자/그룹 생성, 코드 `/srv/web`.
2) `web.service` 위 예시 적용(ProtectSystem/PrivateTmp 등).
3) 방화벽 ufw/firewalld로 80/TCP만 허용.
4) `journalctl -u web`로 오류 확인, `LimitNOFILE` 효과 검증.

---

## 미니 랩 2 — 로그 회전·보존·중앙수집

1) `/var/log/web/*.log`에 `logrotate` 정책 생성(일일 회전, 14보관, 압축).
2) journald 영속 1GB/2주 제한.
3) rsyslog → 중앙 서버로 전송(테스트 환경).

예:
```bash
cat >/etc/logrotate.d/web <<'EOF'
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
EOF
```

---

## 미니 랩 3 — LVM 확장과 쿼터

1) `/srv/data`를 LVM LV로 구성 후 10G 확장.
2) `xfs_quota`로 팀 그룹 100G 소프트/120G 하드 제한.
3) 쿼터 초과 시 동작과 알람 확인.

---

## 보안 하드닝 체크리스트(운영자 관점)

- [ ] 계정은 **개인계정+sudo 위임**, 비밀번호 정책/락아웃 적용.
- [ ] SSH는 **키 인증/루트 금지/원천지 제한**.
- [ ] 방화벽 **기본 차단**과 필수 서비스만 허용.
- [ ] 파일시스템은 **noexec/nodev/nosuid**, **쿼터**와 **logrotate** 활성.
- [ ] 서비스는 **systemd**로, **Restart/리소스상한/샌드박스** 적용.
- [ ] **패치 자동화**, 커널/대규모는 점검창 롤링.
- [ ] 로그는 **journald 영속 + 중앙수집**, **auditd**로 근거 확보.
- [ ] **백업+스냅샷**과 **복구 리허설** 정례화.
- [ ] 구성을 **코드(IaC/Ansible)** 로 관리.

---

## 예상문제(필기/실기)

1) `/data`를 `ext4`로 `noexec,nodev,nosuid` 옵션으로 영구 마운트하는 `/etc/fstab` 행을 작성하라(장치 `/dev/vdb1`).
2) systemd 서비스에 대해 **다시 시작 정책**과 **샌드박스(최소 3개)** 를 설정하는 예시를 제시하라.
3) RHEL에서 로그인 실패 **락아웃** 정책을 적용하고 확인하는 명령을 쓰라.
4) LVM에서 `/dev/vg0/lvapp`을 **온라인**으로 20G 확장하고 파일시스템을 동시 확장하는 명령.
5) journald를 **영속 저장**으로 전환하고 1GB 상한/2주 보존으로 설정하는 방법.
6) ufw로 인바운드 기본 차단, 22/TCP를 10.0.0.0/24에서만 허용하는 명령.
7) xfs 쿼터로 사용자 `alice`의 홈을 10G 소프트, 12G 하드로 제한하는 명령.

예시 답안 스니펫:
```bash
# 1

echo '/dev/vdb1 /data ext4 defaults,noexec,nodev,nosuid 0 2' >> /etc/fstab
mount -a
```
```ini
# (서비스 일부)

[Service]
Restart=always
RestartSec=3
NoNewPrivileges=true
ProtectSystem=strict
ProtectHome=true
PrivateTmp=true
```
```bash
# 3

authselect select sssd with-faillock --force
faillock --user alice
```
```bash
# 4

lvextend -r -L +20G /dev/vg0/lvapp
```
```bash
# 5

mkdir -p /var/log/journal
sed -i 's/^#\?Storage=.*/Storage=persistent/' /etc/systemd/journald.conf
sed -i 's/^#\?SystemMaxUse=.*/SystemMaxUse=1G/' /etc/systemd/journald.conf
sed -i 's/^#\?MaxRetentionSec=.*/MaxRetentionSec=2w/' /etc/systemd/journald.conf
systemctl restart systemd-journald
```
```bash
# 6

ufw default deny incoming
ufw default allow outgoing
ufw allow from 10.0.0.0/24 to any port 22 proto tcp
ufw enable
```
```bash
# (xfs)

mount -o remount /home
xfs_quota -x -c 'limit bsoft=10G bhard=12G alice' /home
xfs_quota -x -c 'report -h' /home
```

---

## 마무리

이 장의 목표는 **운영 가능한 상태**와 **재현 가능한 절차**다.
- 계정/권한/로그/백업/패치/네트워크/서비스가 **문서화된 습관**으로 굳어질 때, 장애와 침해는 **빠르게 진화**한다.
- 다음 장(UNIX/Linux 서버 보안)에서는 여기서 만든 운영 표준을 **정책화**하고, **SELinux/AppArmor·파일 무결성·구성 스캐닝**으로 확장한다.
