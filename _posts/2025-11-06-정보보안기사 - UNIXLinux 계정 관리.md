---
layout: post
title: 정보보안기사 - UNIX/Linux 계정 관리
date: 2025-11-06 15:25:23 +0900
category: 정보보안기사
---
# SECTION 02 UNIX/Linux 서버 취약점 — 01. 계정 관리 (Account Management)

## 계정 관리 위협 모델과 핵심 원칙

### 위협 모델
- **무단 접근**: 기본/공용 계정, 약한 암호, SSH 패스워드 로그인.
- **권한 오남용**: 광범위한 `sudo NOPASSWD`, 위험 명령 허용, PATH 하이재킹.
- **가시성 부족**: 로그인/권한 변경 로깅 부재, 감사(audit) 미구성.
- **수명주기 누락**: 휴면/퇴사자 계정 잔존, 서비스 계정의 인터랙티브 로그인 허용.
- **키·토큰 노출**: 잘못된 `~/.ssh` 권한, agent 포워딩 남용, 비밀 회전 미흡.

### 핵심 원칙
- **최소 권한(least privilege)**, **개인 계정/역할 그룹**만 사용(공용계정 금지).  
- **강한 인증**: 키 기반 SSH + 필요 시 MFA(PAM TOTP/U2F).  
- **정책과 증적**: 정책(PAM/로그인/암호) + 감사(auditd) + 중앙 로그.  
- **수명주기 자동화**: 생성→운영→휴면→폐기 전 단계 자동화(타이머/IaC).

---

## 시스템 계정 데이터 구조 이해 (/etc/passwd, /etc/shadow, /etc/group)

### 빠른 리마인드
- `/etc/passwd`: `name:x:UID:GID:gecos:home:shell`  
- `/etc/shadow`: `name:hash:lastchg:min:max:warn:inactive:expire:flag`  
- `/etc/group`: `group_name:x:GID:user1,user2,...`

### 점검 포인트(취약 신호)
- **UID=0 복수 존재**(루트 복제)  
- **빈 암호**(shadow에서 `::`), 잠금 미설정(`!`/`*` 없음)  
- **홈 디렉터리 없음/권한 과다**  
- **서비스 계정에 인터랙티브 셸**(예: `/bin/bash`)  
- **중복 UID/GID**, **잘못된 기본 그룹**  
- **비표준 셸**(`nologin`/`false` 미사용)

---

## 계정 위험 신호 자동 점검 스크립트(포터블 bash/awk)

```bash
#!/usr/bin/env bash
# acct_audit.sh — UNIX/Linux 계정 취약 구성 빠른 점검

set -euo pipefail

echo "[1] UID=0 복수 계정"
awk -F: '$3==0 {print}' /etc/passwd

echo -e "\n[2] 빈 암호/락 미설정 계정(shadow)"
awk -F: '($2=="") || ($2=="!""") || ($2==":") {print "EMPTY? ->",$1}' /etc/shadow 2>/dev/null || true
awk -F: '$2!~/^(!|\*)/ && $2!~/^\$/' /etc/shadow 2>/dev/null | awk -F: '{print "NOLOCK? ->",$1}'

echo -e "\n[3] 인터랙티브 셸을 가진 시스템/서비스 계정(UID<1000 또는 <500)"
awk -F: '($3<1000 && $7 ~ /(bash|sh|zsh|ksh)$/) {printf "%s (UID=%s SHELL=%s)\n",$1,$3,$7}' /etc/passwd

echo -e "\n[4] 홈 디렉터리 존재/권한"
while IFS=: read -r u x uid gid gecos home shell; do
  [ -d "$home" ] || { echo "HOME MISSING -> $u:$home"; continue; }
  perms=$(stat -c '%a' "$home" 2>/dev/null || echo "???")
  echo "$u:$home:$perms"
done < /etc/passwd | awk -F: '$3!~/^(700|750|740|600|640)$/ {print "HOME PERM CHECK ->",$0}'

echo -e "\n[5] 중복 UID/GID"
awk -F: '{print $3}' /etc/passwd | sort | uniq -d | while read -r d; do
  echo "DUP UID -> $d"
  awk -F: -v d="$d" '$3==d {print "  ",$0}' /etc/passwd
done
awk -F: '{print $3}' /etc/group | sort | uniq -d | while read -r d; do
  echo "DUP GID -> $d"
  awk -F: -v d="$d" '$3==d {print "  ",$0}' /etc/group
done

echo -e "\n[6] sudoers 위험 패턴(NOPASSWD + 위험 커맨드)"
grep -R "NOPASSWD" /etc/sudoers /etc/sudoers.d 2>/dev/null \
 | grep -Ei "vi|vim|less|more|awk|perl|python|find|tee|bash|sh|tar|cp|rsync|scp" || true

echo -e "\n[7] SSH 패스워드 로그인 허용 여부"
sshd_cfg=$(grep -Ei '^(PasswordAuthentication|PermitRootLogin|PubkeyAuthentication|UsePAM)\s+' /etc/ssh/sshd_config 2>/dev/null)
echo "$sshd_cfg"

echo -e "\n[8] .ssh 권한"
find /home -maxdepth 2 -type d -name ".ssh" -print0 2>/dev/null \
 | xargs -0 -I{} sh -c 'p=$(stat -c "%a %U:%G %n" "{}"); echo "$p"; find "{}" -maxdepth 1 -type f -printf "  %m %u:%g %p\n"'
```

> 실행 결과를 **증적(파일/스크린샷)** 으로 보관하고, 각 이슈별로 티켓/조치·검증을 기록한다.

---

## 암호 정책(PAM, login.defs)과 복잡도·수명 설정

### Debian/Ubuntu — `pam_pwquality` / `login.defs`
```
# /etc/pam.d/common-password (예시)
password requisite pam_pwquality.so retry=3 minlen=12 ucredit=-1 lcredit=-1 dcredit=-1 ocredit=-1 difok=4 enforce_for_root
password [success=1 default=ignore] pam_unix.so obscure sha512 remember=5 use_authtok
```

```
# /etc/login.defs (일부)
PASS_MAX_DAYS   90
PASS_MIN_DAYS   1
PASS_WARN_AGE   14
ENCRYPT_METHOD  yescrypt
```

### RHEL/Rocky/Alma — `faillock`/`system-auth` 프로파일
```bash
authselect select sssd with-faillock --force
# 필요 시 /etc/security/pwquality.conf 조정
```

### 암호 엔트로피 간단 개념식
암호 길이 \(L\), 문자집합 크기 \(N\)일 때, 가능한 조합 수는 \(N^L\).  
정보량(엔트로피) 근사:
$$
H \approx L \cdot \log_2(N)
$$
> 예: 소문자+대문자+숫자+기호(약 90)로 길이 12 → \(H \approx 12\cdot\log_2 90 \approx 12 \cdot 6.49 \approx 78 \) 비트 수준.

### 실무 체크리스트
- [ ] 최솟길이≥12, 문자 다양성, 최근 N개 재사용 금지(`remember`), 사전/유사 패턴 차단.  
- [ ] `yescrypt`(Debian 12/RHEL9) 또는 `sha512` 이상 사용 여부.  
- [ ] 만료/경고/유예(**`chage -l`**로 검증).

---

## 인증 실패 잠금(브루트포스 방지)

### RHEL 계열(권장 경로)
```bash
authselect select sssd with-faillock --force
# 확인
faillock --user alice
```

### Debian/Ubuntu (pam_faillock 지원 버전 or 대체)
- 최신 배포판은 `pam_faillock.so` 지원. 미지원이면 `pam_tally2`(구식) 또는 Fail2ban.

Fail2ban(SSH 예):
```bash
apt install -y fail2ban
cat >/etc/fail2ban/jail.d/sshd.local <<'EOF'
[sshd]
enabled  = true
maxretry = 5
findtime = 10m
bantime  = 2h
ignoreip = 10.0.0.0/24
EOF
systemctl enable --now fail2ban
```

체크리스트
- [ ] 연속 실패 잠금/백오프 동작 확인(테스트 계정으로 시뮬).  
- [ ] 허용망(`ignoreip`)로 운영자 잠금 방지.

---

## 서비스 계정과 인터랙티브 로그인 차단

### 원칙
- 서비스 계정은 **로그온 금지** 셸(`/usr/sbin/nologin` 또는 `/bin/false`), 홈 디렉터리 권한 최소화.  
- 실행은 **systemd**로 관리(환경 격리/권한 제한/샌드박스).

### 예시
```bash
# 서비스 계정 생성(로그인 불가)
useradd -r -s /usr/sbin/nologin -d /nonexistent websvc
# 이미 존재하면 셸 변경
usermod -s /usr/sbin/nologin websvc
```

체크리스트
- [ ] `UID<1000`(또는 <500) 계정 모두 `nologin/false` 확인.  
- [ ] 서비스 파일(systemd)의 `User=websvc`, `NoNewPrivileges`, `ProtectSystem` 등 적용.

---

## SSH 계정·키 관리 하드닝

### 서버 측(`sshd_config`)
```text
Protocol 2
PermitRootLogin no
PasswordAuthentication no
PubkeyAuthentication yes
KbdInteractiveAuthentication no
MaxAuthTries 3
LoginGraceTime 30
AllowGroups ops sshusers
# 필요 시 MFA (PAM TOTP/U2F)
```
적용:
```bash
systemctl reload sshd
```

### 사용자 측 권한
```bash
chmod 700 ~/.ssh
chmod 600 ~/.ssh/authorized_keys
chmod 600 ~/.ssh/id_ed25519
```

### authorized_keys 제약
```text
from="10.0.0.0/24",command="/usr/local/sbin/controlled_cmd.sh",no-pty,no-agent-forwarding,no-port-forwarding ssh-ed25519 AAAA...
```

체크리스트
- [ ] 루트 로그인 금지, 패스워드 로그인 금지, **키만** 허용.  
- [ ] 키 회전 주기, 탈취 시 폐기 절차 설계.  
- [ ] `from=`·`command=`·포워딩 금지 옵션 적극 활용.

---

## sudoers 안전 설계(가장 많은 사고 지점)

### 위험 패턴
- `NOPASSWD` + **셸 실행 가능 도구**(vi, vim, less, more, awk, perl, python, find, tar, rsync, tee, cp…).  
- 광범위 와일드카드, 상대 경로, 환경 의존(`PATH`, `LD_*`).

### 안전 패턴
- **래퍼 스크립트**(절대경로, 고정 인자)로 축소.  
- `Defaults use_pty, log_output, timestamp_timeout=5, passwd_tries=3`  
- `Defaults secure_path="/usr/sbin:/usr/bin:/sbin:/bin"`  
- 필요한 경우에만 `NOPASSWD`, 가능한 **비대화형**만.

예시:
```bash
# /usr/local/sbin/restart_web.sh
#!/usr/bin/env bash
exec /usr/bin/systemctl restart web.service
```

```
# /etc/sudoers.d/webops
Defaults:%webops use_pty,log_output,secure_path="/usr/sbin:/usr/bin:/sbin:/bin"
%webops ALL=(root) NOPASSWD: /usr/local/sbin/restart_web.sh
```

검증:
```bash
visudo -c
sudo -l -U alice
```

체크리스트
- [ ] 위험 명령 허용 금지(위 목록).  
- [ ] `use_pty`,`secure_path`,`log_output` 설정.  
- [ ] `sudo -l` 결과를 **증적**으로 남김.

---

## cron/at 실행 통제

```bash
# 허용 목록 방식(화이트리스트)
echo 'alice' > /etc/cron.allow
touch /etc/cron.deny
chmod 640 /etc/cron.allow /etc/cron.deny

echo 'alice' > /etc/at.allow
touch /etc/at.deny
chmod 640 /etc/at.allow /etc/at.deny
```

감사:
```bash
auditctl -w /etc/cron.d -p wa -k cronchg
ausearch -k cronchg -ts today
```

체크리스트
- [ ] cron/at **화이트리스트** 운영.  
- [ ] 스케줄러 변경은 **auditd**로 추적.

---

## 홈 디렉터리·스켈레톤(skel)·기본 umask

### 홈 디렉터리 권장
- 소유자: 사용자, 그룹: 역할 그룹, 권한: `700` 또는 `750`.

자동 생성 템플릿:
```bash
# /etc/skel 권한 점검 및 기본 .profile, .bashrc 최소화
umask 027
```

시스템 전역 기본 umask:
```bash
echo 'umask 027' | tee /etc/profile.d/secure-umask.sh
```

체크리스트
- [ ] 새 계정 생성 시 홈 권한 자동으로 안전하게 배포.  
- [ ] skel에 불필요 파일/스크립트 없음.

---

## 계정 수명주기(온보딩 → 휴면 → 폐기)

### 휴면/퇴사 처리 자동화 예(systemd timer)
```bash
# /usr/local/sbin/lock_inactive.sh
#!/usr/bin/env bash
# 90일 로그인 없는 계정 잠금(예외: 시스템계정)
THRESHOLD=90
now_days=$(date +%s)
awk -F: '$3>=1000 {print $1}' /etc/passwd | while read -r u; do
  lastlog=$(lastlog -u "$u" | awk 'NR==2 {print $4,$5,$6}')
  # 단순 예시: lastlog 파싱이 어려우면 chage -l 사용/로그 DB 활용
done
```
> 실무에서는 **IdM/LDAP/SSO**와 HR 시스템 연동으로 자동화.

체크리스트
- [ ] 계정 생성 승인·기한·역할 매핑.  
- [ ] 휴면 기준(예: 90일) 및 자동 잠금/폐기.  
- [ ] 퇴사 즉시 접근 폐기(키/토큰 폐기, 그룹 제거, VPN 차단).

---

## 감사(auditd) 규칙(계정·권한 변경 가시화)

```bash
apt install -y auditd audispd-plugins || dnf install -y audit
auditctl -w /etc/passwd -p wa -k passwd
auditctl -w /etc/shadow -p wa -k shadow
auditctl -w /etc/group  -p wa -k group
auditctl -w /etc/sudoers -p wa -k sudoers
auditctl -w /etc/ssh/sshd_config -p wa -k sshcfg
# 영구 규칙은 /etc/audit/rules.d/*.rules 에 저장 후
augenrules --load
```

검색:
```bash
ausearch -k passwd -ts today
ausearch -k sudoers -ts -1h
```

체크리스트
- [ ] 핵심 파일에 **write/attribute** 감시.  
- [ ] 규칙 키(`-k`)로 검색 일원화, 중앙 SIEM 연동.

---

## 취약 구성 → 개선 “케이스 스터디”

### 케이스 1: UID 0 복수 계정
- **증상**: `/etc/passwd`에 `root:x:0:0:...`, `toor:x:0:0:...` 등.  
- **위험**: 모든 제어·감사 우회 가능.  
- **조치**: 즉시 삭제/UID 변경, 해당 계정 접근 폐기, 로그 재검토.  
```bash
# toor UID 변경(예시)
usermod -u 1001 toor
groupmod -g 1001 toor
find / -xdev -uid 0 -user toor -exec chown toor {} +
```

### 케이스 2: 서비스 계정에 /bin/bash
- **조치**:
```bash
usermod -s /usr/sbin/nologin dbsvc
```
- systemd 유닛으로 서비스 관리(비루트 + 샌드박스).

### 케이스 3: sudoers에 vi 허용
- **조치**: 래퍼 스크립트로 대체(절대경로·고정 인자), `secure_path`·`use_pty` 강화, 재검증.

### 케이스 4: SSH 패스워드 로그인 허용
- **조치**: `PasswordAuthentication no`, 키 배포, MFA 필요 시 PAM TOTP.

---

## Ansible로 정책 자동화(샘플)

```yaml
- hosts: linux
  become: true
  vars:
    sshd_cfg: /etc/ssh/sshd_config
  tasks:
    - name: Enforce secure umask
      copy:
        dest: /etc/profile.d/secure-umask.sh
        mode: '0644'
        content: "umask 027\n"

    - name: Harden sshd
      lineinfile:
        path: "{{ sshd_cfg }}"
        regexp: '^{{ item.key }}'
        line: "{{ item.key }} {{ item.val }}"
        create: no
      loop:
        - { key: 'PermitRootLogin', val: 'no' }
        - { key: 'PasswordAuthentication', val: 'no' }
        - { key: 'PubkeyAuthentication', val: 'yes' }
        - { key: 'MaxAuthTries', val: '3' }
    - name: Reload sshd
      service: { name: sshd, state: reloaded }

    - name: Add sudoers drop-in
      copy:
        dest: /etc/sudoers.d/webops
        mode: '0440'
        content: |
          Defaults:%webops use_pty,log_output,secure_path="/usr/sbin:/usr/bin:/sbin:/bin"
          %webops ALL=(root) NOPASSWD: /usr/local/sbin/restart_web.sh
```

---

## 운영자 체크리스트(요약)

- [ ] `/etc/passwd`/`shadow`/`group` **정합성**(UID/GID 중복·UID0 복수).  
- [ ] 휴면/퇴사 계정 **잠금·폐기** 자동화.  
- [ ] SSH: 루트 금지·패스워드 금지·키만 허용·MFA 필요 시 적용.  
- [ ] `.ssh`/키 **권한** 및 `authorized_keys` 제약(`from=`, `command=`).  
- [ ] sudoers: **위험 커맨드 금지**, 래퍼, `use_pty`/`secure_path`/`log_output`.  
- [ ] 서비스 계정 **nologin/false**, systemd 샌드박스 사용.  
- [ ] cron/at **화이트리스트**, 스케줄 변경 **감사**.  
- [ ] PAM: `pwquality`·`faillock`·`login.defs`(만료/경고/유예).  
- [ ] auditd 핵심 파일 감시 + 중앙 SIEM 전송.  
- [ ] 정책/조치 **증적(로그·설정 캡처)** 보관.

---

## 미니 랩 A — “위험 sudoers” 정리 실습

1) 현재 sudoers에서 `NOPASSWD` + 위험 커맨드 찾기:
```bash
grep -R "NOPASSWD" /etc/sudoers /etc/sudoers.d \
 | grep -Ei "vi|vim|awk|perl|python|find|tee|bash|sh|tar|rsync|cp"
```
2) 허용이 필요한 동작을 **래퍼 스크립트**로 치환하고 절대 경로·고정 인자 사용.  
3) `visudo -c`, `sudo -l -U <user>` 로 검증.  
4) `auditd`로 `/etc/sudoers*` 쓰기 감시.

---

## 미니 랩 B — “SSH 키만 허용 + MFA(TOTP)”

1) `sshd_config`에서 `PasswordAuthentication no`, `KbdInteractiveAuthentication no`.  
2) 사용자별 `google-authenticator` 초기화(랩 전용), `PAM`에 `pam_google_authenticator.so` 추가(운영 정책에 따라).  
3) 방화벽으로 관리망만 22/TCP 허용.  
4) 실패 잠금/Fail2ban 동작 확인.

---

## 미니 랩 C — “서비스 계정 로그인 차단 + 홈/권한 정비”

1) 서비스 계정 셸 `nologin`으로 변경.  
2) 홈 디렉터리 권한 일괄 점검·수정(700/750).  
3) `/etc/skel`/`umask 027` 적용 후 신규 계정 생성 테스트.

---

## 예상문제(필기/실기)

1) `/etc/shadow`에서 **잠금 계정**을 의미하는 접두 문자는? 잠금/해제 명령은?  
2) `UID=0` 계정이 2개 이상일 때의 위험성과 즉각적인 조치 2가지를 쓰시오.  
3) sudoers에서 **허용하면 위험한 커맨드** 유형을 5개 이상 쓰고, 안전한 대체 방식을 설계하시오.  
4) SSH에서 루트 금지·패스워드 금지·키만 허용 설정 3가지를 쓰시오.  
5) 서비스 계정의 로그인 차단을 위해 사용하는 셸 두 가지와 적용 명령을 쓰시오.  
6) cron/at 실행 통제를 **화이트리스트** 방식으로 구현하는 파일과 권한을 쓰시오.  
7) `auditd`로 `/etc/passwd`와 `/etc/sudoers` 변경을 추적하는 규칙과 조회 명령을 쓰시오.

예시 스니펫:
```bash
# 1. 잠금/해제(배포판에 따라)
passwd -l alice    # 잠금(선호)
passwd -u alice    # 해제
# 또는 usermod -L/-U
```
```bash
# 4. sshd_config 예
PermitRootLogin no
PasswordAuthentication no
PubkeyAuthentication yes
```
```bash
# 7. auditd
auditctl -w /etc/passwd -p wa -k passwd
auditctl -w /etc/sudoers -p wa -k sudoers
ausearch -k passwd -ts today
```

---

## 마무리

- 계정 관리는 **보안의 최전선**이다.  
- 이 장의 스크립트/체크리스트/정책 템플릿을 **IaC/Ansible**로 고정하고, **auditd + 중앙 로그**로 **증적 기반 운영**을 확립하라.  
- 다음 문서(파일/디렉터리 관리 취약점)에서는 **권한/ACL/특수 비트/마운트 옵션** 과 **무결성/암호화**를 계정 정책과 연결해 닫는다.