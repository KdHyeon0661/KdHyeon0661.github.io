---
layout: post
title: 정보보안기사 - UNIX/Linux 서버 보안
date: 2025-11-05 23:25:23 +0900
category: 정보보안기사
---
# SECTION 01 시스템 기본 학습 — 04. UNIX/Linux 서버 보안

## 개요: 위협 모델과 보안 목표 요약

- **외부 침투**: 인터넷 노출 서비스(SSH/HTTP/DB) 취약점, 약한 인증, 구성 실수.
- **내부 확산**: 탈취 자격증명, SUID/권한 상승, 세계 쓰기 가능 경로 악용.
- **지속성(영속화)**: 스케줄러/서비스/Run 스크립트/동적 라이브러리/커널 모듈.
- **데이터 손실/유출**: 권한 과다, 로그/백업 미암호화, 원격 전송 평문.
- **목표**: **최소 노출·최소 권한·가시성/검증 가능성** 확보, 자동화/표준화.

---

## 보안 베이스라인 설계(운영→보안 통제 상향)

1) **계정/인증**: 개인계정+역할 그룹, `sudoers` 최소 허용, 잠금/만료, 실패 잠금.
2) **SSH**: 키 인증·루트 로그인 금지·원천지 제한·강한 암호군·Fail2ban.
3) **서비스 최소화**: 리스닝 포트 제거, systemd 샌드박스로 격리.
4) **파일시스템/권한**: `noexec,nodev,nosuid`, SUID 감축, 세계 쓰기 경로/Sticky, ACL/Capabilities.
5) **네트워크**: 인바운드 기본 차단, 필요 포트만 허용, IPv6 정책 동등 적용.
6) **커널/시스템**: sysctl 하드닝, 모듈 블랙리스트, 코어덤프 제한.
7) **로깅/감사**: journald 영속/상한, rsyslog TLS 원격 전송, auditd 규칙.
8) **무결성/취약점 진단**: AIDE/Tripwire, Lynis/SCAP, 패치 자동화.
9) **탐지/차단**: Fail2ban, Suricata/Zeek(선택), EDR 연계.
10) **백업/복구/암호화**: LUKS/키 관리, 오프사이트·불변 저장소.

---

## 계정/인증 하드닝

### 정책 적용(요약)

- **비밀번호**: 길이/복잡도/재사용 금지, 만료/경고/유예.
- **잠금**: 실패 N회 시 잠금/백오프.
- **sudo**: 명령 단위 허용, 로깅·시간 제한.
- **MFA(SSH)**: 가능 시 TOTP(U2F/FIDO) 채택.

예제(암호 품질·잠금: Debian/Ubuntu)
```bash
apt install -y libpam-pwquality libpam-google-authenticator

# /etc/pam.d/common-password (예시)

password requisite pam_pwquality.so retry=3 minlen=12 ucredit=-1 lcredit=-1 dcredit=-1 ocredit=-1 remember=5

# /etc/pam.d/common-auth (faillock는 RHEL 계열, Debian은 pam_tally2 또는 외부 사용)
# RHEL 계열(권장): authselect select sssd with-faillock --force

```

sudo 최소권한 위임:
```bash
visudo
# /etc/sudoers.d/webops

%webops ALL=(root) NOPASSWD: /usr/bin/systemctl restart web,/usr/bin/journalctl -u web
Defaults:%webops timestamp_timeout=5,log_output
```

계정 만료/잠금/감사:
```bash
chage -M 90 -W 14 -I 7 alice
usermod -L old_user
last -n 20
```

체크리스트
- [ ] 공용계정 금지, 개인계정+그룹+sudo 위임만 사용.
- [ ] 실패 잠금/백오프 동작 검증(Screen/Docs).
- [ ] `sudo -l` 결과 캡처(증적), `last/lastb`로 로그인 이력 점검.

---

## SSH 하드닝(키·암호군·원천지·탐지)

`/etc/ssh/sshd_config` 핵심 지시문:
```text
Protocol 2
HostKey /etc/ssh/ssh_host_ed25519_key
KexAlgorithms curve25519-sha256,curve25519-sha256@libssh.org
Ciphers chacha20-poly1305@openssh.com,aes256-gcm@openssh.com
MACs hmac-sha2-512-etm@openssh.com,hmac-sha2-256-etm@openssh.com

PermitRootLogin no
PasswordAuthentication no
PubkeyAuthentication yes
MaxAuthTries 3
LoginGraceTime 30
ClientAliveInterval 300
ClientAliveCountMax 2
AllowUsers alice webops
Banner /etc/issue.net
```
적용:
```bash
systemctl reload sshd
```

원천지 제한(ufw/firewalld):
```bash
# ufw

ufw default deny incoming
ufw allow from 10.0.0.0/24 to any port 22 proto tcp
ufw enable
```

Fail2ban(ssh):
```bash
apt install -y fail2ban
cat >/etc/fail2ban/jail.d/sshd.local <<'EOF'
[sshd]
enabled = true
maxretry = 5
findtime = 10m
bantime = 2h
ignoreip = 10.0.0.0/24
EOF
systemctl enable --now fail2ban
fail2ban-client status sshd
```

체크리스트
- [ ] 패스워드 로그온 금지, 키 인증만.
- [ ] 강한 KEX/Cipher/MAC 적용(배포판 기본 이상).
- [ ] Fail2ban 동작 검증(의도적 실패 로그인 후 차단 확인).

---

## 서비스 최소화와 노출 축소

리스닝 서비스 식별/정리:
```bash
ss -tulpn | sort
systemctl list-unit-files --type=service | grep enabled
systemctl disable --now cups avahi-daemon rpcbind 2>/dev/null || true
```

systemd 샌드박스(서비스 격리) — 운영(03편) 확장:
```ini
# /etc/systemd/system/web.service (추가 보안)

[Service]
NoNewPrivileges=true
PrivateTmp=true
ProtectSystem=strict
ProtectHome=true
ProtectKernelTunables=true
ProtectKernelModules=true
ProtectControlGroups=true
RestrictSUIDSGID=true
MemoryDenyWriteExecute=true
RestrictNamespaces=true
SystemCallFilter=@system-service
CapabilityBoundingSet=CAP_NET_BIND_SERVICE
IPAddressDeny=any
IPAddressAllow=10.0.0.0/24
```
적용/검증:
```bash
systemctl daemon-reload
systemctl restart web
systemd-analyze security web.service
```

체크리스트
- [ ] 불필요 데몬/소켓 비활성·제거.
- [ ] 샌드박스 지시자 최소 5개 이상 적용, `systemd-analyze security`로 점수 확인.
- [ ] 루트 필요 없는 기능은 **Capabilities**만 부여.

---

## 파일시스템/권한 하드닝

### 마운트 옵션

`/etc/fstab` 예:
```text
/dev/vdb1 /data ext4 defaults,noexec,nodev,nosuid 0 2
tmpfs /tmp tmpfs defaults,noexec,nodev,nosuid 0 0
```
적용:
```bash
mount -a
```

### SUID/세계 쓰기 점검

```bash
# SUID/SGID 바이너리 조사

find / -xdev -perm -4000 -o -perm -2000 -type f -print 2>/dev/null | sort

# 세계 쓰기 디렉터리(Sticky 없는 경우 포함) 조사

find / -xdev -type d -perm -0002 ! -perm -1000 -print 2>/dev/null
```

### Sticky bit(공유 디렉터리)

```bash
chmod +t /tmp /var/tmp
```

### Capabilities 활용(특권 최소화)

```bash
setcap 'cap_net_bind_service=+ep' /usr/local/bin/web-lite
getcap /usr/local/bin/web-lite
```

체크리스트
- [ ] `/tmp`, `/var/tmp`에 Sticky 설정.
- [ ] 데이터/임시 경로 `noexec,nodev,nosuid`.
- [ ] SUID 바이너리 목록 보관·리뷰, 불필요 SUID 제거(업그레이드 시 재검토).

---

## 네트워크 방화벽: nftables/ufw/firewalld

nftables 최소 정책(IPv4/IPv6 동시):
```bash
cat >/etc/nftables.conf <<'EOF'
table inet filter {
  chain input {
    type filter hook input priority 0;
    ct state established,related accept
    iif lo accept
    tcp dport { 22, 80, 443 } accept
    ip protocol icmp accept
    ip6 nexthdr icmpv6 accept
    drop
  }
}
EOF
nft -f /etc/nftables.conf
systemctl enable --now nftables
```

firewalld(RHEL):
```bash
firewall-cmd --set-default-zone=drop
firewall-cmd --zone=public --add-service=ssh --permanent
firewall-cmd --zone=public --add-service=http --permanent
firewall-cmd --reload
```

체크리스트
- [ ] 인바운드 기본 차단, 서비스 단위 허용.
- [ ] IPv6도 동일 통제 적용(불사용 시 명시 비활성).
- [ ] 관리자 망/프록시 망 **분리** 검토.

---

## 커널/sysctl 하드닝

`/etc/sysctl.d/99-security.conf`:
```conf
# Anti-spoofing / redirect / source route

net.ipv4.conf.all.rp_filter = 1
net.ipv4.conf.default.rp_filter = 1
net.ipv4.conf.all.accept_redirects = 0
net.ipv4.conf.default.accept_redirects = 0
net.ipv6.conf.all.accept_redirects = 0
net.ipv4.conf.all.send_redirects = 0

# TCP

net.ipv4.tcp_syncookies = 1
net.ipv4.tcp_timestamps = 1
net.ipv4.tcp_fin_timeout = 30

# IP forwarding off (기본 서버)

net.ipv4.ip_forward = 0
net.ipv6.conf.all.forwarding = 0

# 파일 핸들/코어덤프/ASLR

fs.suid_dumpable = 0
kernel.randomize_va_space = 2
kernel.kptr_restrict = 2
```
적용:
```bash
sysctl --system
```

모듈 블랙리스트(불필요 USB 저장장치 등):
```bash
cat >/etc/modprobe.d/blacklist.conf <<'EOF'
blacklist usb_storage
blacklist firewire_ohci
EOF
```

체크리스트
- [ ] 리다이렉트/소스 라우팅 차단, rp_filter=1.
- [ ] 코어덤프·주소공간 무작위화(ASLR) 활성.
- [ ] 불필요 모듈 블랙리스트.

---

## 로깅·중앙수집·보존

journald 영속/상한:
```bash
mkdir -p /var/log/journal
sed -i 's/^#\?Storage=.*/Storage=persistent/' /etc/systemd/journald.conf
sed -i 's/^#\?SystemMaxUse=.*/SystemMaxUse=1G/' /etc/systemd/journald.conf
sed -i 's/^#\?MaxRetentionSec=.*/MaxRetentionSec=2w/' /etc/systemd/journald.conf
systemctl restart systemd-journald
```

rsyslog TLS 원격 전송(예):
```bash
# /etc/rsyslog.d/90-remote.conf

*.* @@(o)logs.example.corp:6514
# (o)=TLS, 인증서 신뢰 구성 필요

systemctl restart rsyslog
```

의심 행위 빠른 검색:
```bash
journalctl -p err -S -2h
journalctl -u sshd --since -1d | grep -i "failed"
grep -E 'segfault|denied|permission' /var/log/syslog 2>/dev/null | tail
```

체크리스트
- [ ] 로그 영속·상한·보존 기간 지정.
- [ ] 원격 전송 TLS 설정(중앙 수집/검색).
- [ ] 핵심 서비스 단위 쿼리 템플릿 준비.

---

## 감사(auditd) 규칙과 탐지

핵심 파일/권한 변경/영속성 탐지:
```bash
apt install -y auditd audispd-plugins || dnf install -y audit
# sudoers/계정 DB

auditctl -w /etc/sudoers -p wa -k sudoers
auditctl -w /etc/passwd -p wa -k passwd
auditctl -w /etc/shadow -p wa -k shadow
# SSH 설정

auditctl -w /etc/ssh/sshd_config -p wa -k sshcfg
# 서비스·유닛 변경

auditctl -w /etc/systemd/system -p wa -k unitdir
# setuid 파일 생성/변경 탐지(권장: SCAP 규칙 사용)

auditctl -a always,exit -F arch=b64 -S chmod,fchmod,fchmodat -F a2&~0x1FF=0x0 -k permchange
```
검색:
```bash
ausearch -k sudoers --success no
ausearch -k unitdir -ts today
```
영구 규칙:
```bash
augenrules --load
```

체크리스트
- [ ] 계정/권한/서비스/SSH/패키지 DB에 감사 규칙.
- [ ] 키(`-k`)로 태깅해 조회 용이.
- [ ] 룰·로그 보존과 중앙집중 분석 연계.

---

## 무결성 점검: AIDE/Tripwire

AIDE 초기화/검증:
```bash
apt install -y aide || dnf install -y aide
aideinit
mv /var/lib/aide/aide.db.new /var/lib/aide/aide.db
# 정기 검증(타이머/크론)

aide --check | tee /var/log/aide/latest.txt
```
정책 파일(`/etc/aide/aide.conf`)에서 감시 경로/속성 지정.
변경 감지 → **변경 승인/거부 절차** 문서화.

체크리스트
- [ ] 초기 DB는 **정상 상태에서** 생성(골든 스냅샷).
- [ ] 결과는 중앙 저장·알림, 거짓 양성 튜닝.

---

## SELinux/AppArmor(LSM) 적용

### SELinux(RHEL/Alma/Rocky)

상태/모드:
```bash
getenforce
setenforce 1      # Enforcing (임시)
# 영구: /etc/selinux/config -> SELINUX=enforcing

```
컨텍스트 복구:
```bash
restorecon -R -v /var/www/html
```
부울 예(웹의 네트워크 연결 허용):
```bash
getsebool -a | grep httpd
setsebool -P httpd_can_network_connect on
```
감사/정책 생성(deny 학습):
```bash
audit2allow -w -a
audit2allow -a -M mypolicy
semodule -i mypolicy.pp
```

### AppArmor(Ubuntu/Debian)

상태:
```bash
aa-status
```
프로파일 생성/튜닝:
```bash
aa-genprof /usr/sbin/nginx
# 안내에 따라 허용/거부 학습

```

체크리스트
- [ ] Enforcing 모드(허용→경고가 아닌 **차단**).
- [ ] 서비스별 정책/부울로 **필요 최소 권한**만.
- [ ] 레이블/프로파일 오류 시 `restorecon`/`aa-status`로 진단.

---

## 취약점 점검: Lynis/SCAP

Lynis:
```bash
apt install -y lynis || dnf install -y lynis
lynis audit system | tee /var/log/lynis.log
```

SCAP(Security Content Automation Protocol, OpenSCAP):
```bash
apt install -y openscap-scanner scap-security-guide || dnf install -y openscap-scanner scap-security-guide
oscap xccdf eval --profile xccdf_org.ssgproject.content_profile_standard \
  --results /var/tmp/ssg-results.xml \
  /usr/share/xml/scap/ssg/content/ssg-*.xml
```
결과에서 **패스/실패/권고**를 운영 표준에 반영(자동화 가능).

체크리스트
- [ ] 주기적 점검/추세 비교, 미조치 항목 티켓화.
- [ ] 자동 수정(Ansible/SCAP Fix) 적용 전 **스테이징**에서 검증.

---

## 백업·암호화·키 관리

LUKS 예(새 디스크, 실습 환경 전용):
```bash
apt install -y cryptsetup || dnf install -y cryptsetup
cryptsetup luksFormat /dev/vdb
cryptsetup open /dev/vdb securedata
mkfs.xfs /dev/mapper/securedata
mkdir -p /secure
mount /dev/mapper/securedata /secure
```
원격 백업(암호화 + 무결성):
```bash
# borg/ restic 권장. 예시는 restic

apt install -y restic
export RESTIC_REPOSITORY=sftp:backup@backup.example:/backups/server1
export RESTIC_PASSWORD=...
restic init
restic backup /etc /var/www /secure
```

체크리스트
- [ ] 민감 데이터 암호화(LUKS/애플리케이션 레벨).
- [ ] 키/패스프레이즈 별도 보관, 회수/교체 절차.
- [ ] 정기 **복구 리허설**(단순 백업은 무의미).

---

## 미니 랩 A — “SSH+Fail2ban+auditd” 통합 검증

1) `sshd_config` 강화(키, 루트 금지, AllowUsers, KEX/Cipher).
2) ufw/firewalld로 원천지 제한.
3) Fail2ban `sshd` 활성화, 의도적 실패 → `banned` 확인.
4) `auditd`: `/etc/ssh/sshd_config` 변경 감사, `ausearch -k sshcfg`.
5) 중앙 로그 전송이 켜져 있는지 원격 수신 측에서 검색.

---

## 미니 랩 B — “systemd 샌드박스 + Capabilities”로 웹 경량화

1) `web-lite` 바이너리(비루트) + `cap_net_bind_service` 부여.
2) `web.service`에 Protect*/Restrict*/CapabilityBoundingSet/MemoryDenyWriteExecute/ SystemCallFilter 적용.
3) `systemd-analyze security web.service` 점수 확인.
4) AIDE DB 생성 후 유닛 파일 변경 감지 실험.

---

## 미니 랩 C — “AIDE + SCAP + Lynis” 정기 점검 배치

1) AIDE DB 초기화, 타이머로 `aide --check` 결과 메일/Slack 전송.
2) 주간 SCAP 표준 프로파일 평가, 결과 XML → HTML 변환.
3) Lynis 점수 추세 그래프화(간단 스크립트).

---

## 흔한 구성 실수와 방지

- **SSH 패스워드 허용**: 무차별 대입/크리덴셜 스터핑 → 키 인증, Fail2ban, 원천지 제한.
- **세계 쓰기 디렉터리(Sticky 없음)**: 타인 파일 삭제/교체 → Sticky 추가/권한 조정.
- **SUID 남발**: 권한 상승 표면 → 목록 주기 점검/제거/Capabilities 대체.
- **로깅 미흡**: 사고 재현 불가 → journald 영속+상한, 원격 전송, auditd 핵심 규칙.
- **방화벽 허술**: 불필요 포트 노출 → 기본 차단, 서비스 단위 허용, IPv6 동등.
- **Enforcing 비활성**: SELinux/AppArmor Permissive 방치 → 프로파일/부울 튜닝 후 Enforcing 전환.
- **무패치 장기 운영**: 익스플로잇 가능성 누적 → 자동 보안 업데이트+창구 롤링.

---

## 점검 체크리스트(운영 증적 기준)

- [ ] SSH: `PermitRootLogin no`, `PasswordAuthentication no`, AllowUsers, 강한 KEX/Cipher/MAC, Fail2ban 동작 스크린샷.
- [ ] 방화벽: 인바운드 기본 차단, 정책 스크립트/스냅샷.
- [ ] 파일시스템: `/etc/fstab`에 `noexec,nodev,nosuid`, `/tmp` Sticky.
- [ ] 권한: SUID/세계 쓰기 디렉터리 목록 리포트.
- [ ] sysctl: 99-security.conf 값과 `sysctl -a` 캡처.
- [ ] journald: 영속/상한/보존 설정, rsyslog TLS 전송 캡처.
- [ ] auditd: 룰 파일과 `ausearch` 결과 캡처.
- [ ] AIDE: 초기 DB와 최근 검사 리포트.
- [ ] LSM: SELinux/AppArmor Enforcing, 정책/부울/프로파일 증적.
- [ ] 진단: Lynis/SCAP 결과와 보완 조치 티켓.

---

## 실전 문제(필답형·실습형 혼합)

1) **문제**: `/data`를 실행 금지·장치파일 무효·SUID 무효로 영구 마운트하라.
   **답**:
   ```text
   /dev/vdb1 /data ext4 defaults,noexec,nodev,nosuid 0 2
   ```
   ```bash
   mount -a
   ```

2) **문제**: SSH에서 루트 로그인을 금지, 비밀번호 인증 금지, 허용 사용자 `alice webops`만 남기고 KEX/Cipher/MAC을 강화하라.
   **답**: (본문 `sshd_config` 예시 + `systemctl reload sshd`)

3) **문제**: 2시간 내 `sshd_config` 변경을 auditd로 찾는 명령은?
   **답**:
   ```bash
   ausearch -k sshcfg -ts -2h
   ```

4) **문제**: setuid/setgid 파일을 모두 나열하는 원라인 명령?
   **답**:
   ```bash
   find / -xdev -perm -4000 -o -perm -2000 -type f -print 2>/dev/null
   ```

5) **문제**: systemd 유닛에 최소 5개 샌드박스 지시자를 넣고 80/tcp 바인딩만 허용하라.
   **답**: (본문 `web.service` 샌드박스 예시 + `CapabilityBoundingSet=CAP_NET_BIND_SERVICE`)

6) **문제**: journald를 영속 저장으로 바꾸고 1GB 상한/2주 보존 설정, 재시작 명령?
   **답**: (본문 설정 + `systemctl restart systemd-journald`)

7) **문제**: Fail2ban에서 `sshd` jail을 10분 내 5회 실패 시 2시간 차단하도록 설정하는 파일과 키 파라미터는?
   **답**: `/etc/fail2ban/jail.d/sshd.local`의 `maxretry/findtime/bantime`.

---

## 결론

- 보안은 **구성의 습관화**다. 운영(03편)의 절차를 토대로, 본 편의 통제를 **코드화(IaC)** 하고 **정기 진단(AIDE/Lynis/SCAP)** 과 **감사(auditd)** 로 닫아라.
- 격리(systemd 샌드박스·SELinux/AppArmor), 최소 권한(Capabilities/sudo), 최소 노출(방화벽/서비스 제거), 가시성(로그/감사/중앙수집)이 함께 굴러갈 때 **침해 억제력**이 나온다.
- 다음 단계는 **취약점 진단·모의해킹 운영 흐름(스캐닝→분석→보고)** 으로 연결하여, 서버 보안을 **지속 가능한 프로세스**로 완성한다.
