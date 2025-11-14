---
layout: post
title: 시스템보안 - 실습(랩에서 Kerberoast → ACL Misconfig → DA 달성 경로 모델링)
date: 2025-10-26 19:30:23 +0900
category: 시스템보안
---
# 실습: 랩에서 **Kerberoast → ACL Misconfig → DA 달성 경로** 모델링

## 랩 목표와 안전 가이드

- **목표**:
  1) **SPN 보유 서비스 계정**에서 **약한 구성(예: RC4 허용)**이 남아 있을 때 로그·지표가 어떻게 보이는지 **정상 요청**만으로 관찰
  2) **ACL 오구성(WriteDACL/GenericAll 등)**이 **권한 전이(권한 그래프 경로)**를 형성하는 **조건**을 **읽기 전용 점검**으로 포착
  3) 위 두 축이 결합될 때(예: 서비스 계정 탈취 가능성 **모델**) **DA에 닿는 경로**를 **그래프적**으로 식별하고 **제거 계획**을 수립
- **금지**: 티켓/해시/토큰 **탈취·재사용** 및 비인가 권한 **변경/삽입**
- **허용**: 랩에서 **정상 로그인/서비스 접근**으로 TGS 이벤트 발생, AD **읽기 전용 인벤토리**, 모의 **ACL 드리프트 탐지** 및 **교정(정상화)**

---

## 준비: 랩 토폴로지 & 정책

### A) 최소 구성

- **DC/KDC**: Windows Server(AD DS) 1대
- **앱 서버**: IIS 또는 파일 서버 1대(서비스 계정으로 구동)
- **클라이언트**: Windows 11(도메인 조인)
- **수집**:
  - DC: Security 4768/4769/4771/4776
  - 멤버서버: Security 4624/4672, Sysmon(선택)
  - SIEM: Splunk/Elastic/Sentinel 중 택1

### B) 권장 보안 기본선(초기부터)

- **Kerberos AES-only**(RC4 비활성), TGT/TGS 수명 단축
- **서비스 계정 gMSA**(가능하면) 또는 **긴 랜덤 암호+주기 회전**
- **LDAP Signing + Channel Binding**, **SMB 서명/암호화**
- **LSA 보호(PPL), Credential Guard**, **NTLM 축소(감사→거부)**

> ※ 실습 중 **일부 약한 설정**을 의도적으로 남겨 **로그 시그널**을 관찰할 수 있으나, **프로덕션에는 즉시 적용 금지**입니다.

---

## 단계 1 — Kerberoast 위험 **시그널**을 “정상 요청”으로 관찰

### 개념 요약

- Kerberoast는 **SPN 보유 서비스 계정**의 **TGS**를 가져다가 **오프라인**에서 키 크래킹을 시도하는 공격군입니다.
- 실습에서는 **정상 TGS 요청**만 유도해 **DC 보안 로그**에서 **암호화 타입/요청량/분포**를 관찰하고, **정책 개선점**을 도출합니다.

### (PowerShell) SPN·암호화 타입 인벤토리(읽기 전용)

```powershell
Import-Module ActiveDirectory

# SPN 보유 계정 나열 (서비스 계정 후보)

Get-ADUser -LDAPFilter "(servicePrincipalName=*)" -Properties servicePrincipalName, userAccountControl |
  Select-Object SamAccountName, Enabled, userAccountControl, servicePrincipalName

# RC4 허용 여부(예시: "이 계정은 RC4만 허용" 같은 플래그를 직접 검사하기보다는
#    도메인 Kerberos 정책과 계정 암호 업데이트 시점/암호화 정책을 함께 봅니다.)

(Get-ADDefaultDomainPasswordPolicy).KerberosEncryptionType

# 최근 TGS(4769)에서 Ticket Encryption Type 추출(DC에서 실행 권장)

$since = (Get-Date).AddHours(-8)
Get-WinEvent -FilterHashtable @{LogName='Security'; Id=4769; StartTime=$since} |
  ForEach-Object {
    $p = $_.Properties
    [pscustomobject]@{
      Time         = $_.TimeCreated
      ServiceName  = $p[0].Value    # SPN/서비스 계정
      Client       = $p[1].Value
      ClientAddr   = $p[8].Value
      EncType      = $p[10].Value   # 0x12,AES256; 0x11,AES128; 0x17,RC4 등
    }
  } | Group-Object EncType | Sort-Object Count -Desc | Select-Object Name,Count
```

**해석 포인트**
- **EncType=0x17(RC4)** 비율이 의미 있게 존재 → **AES-only**로 전환 필요
- 특정 시간대/사용자에서 **TGS 급증** → 의심 패턴(대량 SPN 조회/접근) 검토

### (KQL 예시) TGS 급증 탐지(모델)

```kusto
SecurityEvent
| where EventID == 4769 and TimeGenerated > ago(1h)
| summarize TGS=count() by bin(TimeGenerated, 5m), Account
| where TGS > 100   // 환경 기준에 맞게 조정
```

---

## 단계 2 — ACL Misconfig **모델링**: 권한 전이 경로 “읽기 전용” 탐지

### 개념 요약

- AD 객체의 DACL에 **WriteDACL/GenericAll/Owner/ExtendedRight**가 부여되면, **간접 권한 상승 경로**가 생성됩니다.
- 실습에서는 **권한 변경 없이** 현재 AD의 **위험 ACL** 존재 여부만 **리포트** 합니다.

### (PowerShell) 위험 ACL 스캐너(읽기 전용 스케치)

```powershell
Import-Module ActiveDirectory
$root = (Get-ADRootDSE).defaultNamingContext

# DCSync 관련 ExtendedRights GUID들

$ext = @(
  "1131f6aa-9c07-11d1-f79f-00c04fc2dcd2", # Replicating Directory Changes
  "1131f6ad-9c07-11d1-f79f-00c04fc2dcd2", # Replicating Directory Changes All
  "89e95b76-444d-4c62-991a-0facbeda640c"  # In Filtered Set
)

$sd = Get-Acl ("AD:$root")
$danger = @()

# 루트 DACL에서 ExtendedRight/DCSync 권한자

$danger += $sd.Access | Where-Object {
  $_.ActiveDirectoryRights -match 'ExtendedRight' -and $_.ObjectType -in $ext
} | Select-Object IdentityReference, ActiveDirectoryRights, ObjectType

# Unconstrained delegation(컴퓨터)

$uDeleg = Get-ADComputer -Filter {TrustedForDelegation -eq $true} -Properties TrustedForDelegation |
  Select-Object Name, DNSHostName

# RBCD 보유 객체

$rbcd = Get-ADComputer -Filter * -Properties 'msDS-AllowedToActOnBehalfOfOtherIdentity' |
  Where-Object {$_.('msDS-AllowedToActOnBehalfOfOtherIdentity')} |
  Select-Object Name, @{n='RBCD';e={$_.('msDS-AllowedToActOnBehalfOfOtherIdentity').Access}}

[pscustomobject]@{
  DCSyncHolders = ($danger | Group-Object IdentityReference | % { "$($_.Name)=$($_.Count)" }) -join "; "
  UnconstrainedDelegation = ($uDeleg.Name -join ", ")
  RBCDCount = $rbcd.Count
} | Format-List
```

**해석 포인트**
- **DCSync 권한 보유자**는 백업/IDM 등 **필수만** 있어야 함
- **Unconstrained Delegation**은 위험. (특히 Tier-0 근처) → Constrained/RBCD 또는 제거
- **RBCD**는 소유자/ACL이 **정확한 팀·서비스**로 제한되어야 함

---

## 단계 3 — “Kerberoast(시그널) + ACL Misconfig(그래프)” 결합 경로 시각화

> 실제 권한 상승을 하지 않고, **경로가 존재하는지**만 **그래프적**으로 파악합니다.

### 경로 구성 요소(개념)

1) **SPN 계정**의 **AES 미사용/RC4 허용/TGS 다량** → **서비스 계정 노출 위험**
2) 해당 서비스 계정 또는 그에 연계된 객체에 대해 **GenericAll/WriteDACL**을 보유한 주체 존재
3) 위 주체가 **그룹/OU/GPO**를 통해 **Tier-0 자산(DC/DA)**에 닿는 간선 보유

### 경로 식별(간단 의사코드)

- 노드: Users/Groups/Computers/GPO/OU
- 엣지: AdminTo, HasSession, MemberOf, GPOApply, WriteDACL, RBCD, AllowedToDelegateTo
- 목표: “모든 노드 → Tier-0” **최단 경로** (길이 ≤ 4) 리스트업
- 조치: 가장 **작은 비용**으로 **끊을 수 있는 엣지**(권한 축소/위임 제거/GPO 권한 교정) 선정

> 실무에서는 BloodHound/Pathfinding 쿼리를 활용(조직 승인 하에). 본 문서에선 개념만 설명.

---

## 단계 4 — 교정(하드닝) 실행 계획 수립(안전)

**우선순위 예시**
1) **SPN 계정 AES-only** 강제, **암호 회전/주기화**(가능하면 **gMSA 전환**)
2) **DCSync 권한 보유자 정리**, 필요한 시스템 계정만 유지
3) **Unconstrained Delegation 제거**, RBCD/Constrained로 대체(정확한 SPN 한정)
4) **GPO/OU 권한** 재정립: 편집·링크 권한은 **전담 그룹**으로 최소화
5) **LDAP Signing/Channel Binding**, **SMB 서명/암호화**, **LSA PPL** 즉시 반영

---

# 방어: **Tiering 모델, PAW, gMSA, LDAP/Signed SMB, LSA 보호**

## Tiering 모델(계층화) & 세션 격리

### 개념

- **Tier-0**: DC/PKI/IdP/저장소 루트 등 “신뢰 루트”
- **Tier-1**: 서버(애플리케이션/DB/파일)
- **Tier-2**: 워크스테이션
- **원칙**: **상위 계층 계정이 하위 계층에 로그인 금지**(세션 스필 방지)

### GPO/운영 수칙(핵심)

- 보호 그룹(DA/EA 등) **워크스테이션 로그인 차단**
- 로컬 관리자 **철폐** + **LAPS/Windows LAPS**로 계정 랜덤화
- **WinRM/PSRemoting/SMB/LDAP** 접근 제어 목록을 계층별로 분리

---

## PAW(Privileged Access Workstation)

### 설계

- **관리 전용 단말**: 인터넷·메일·문서 편집 금지 or 격리 VDI
- **보안 베이스라인**: WDAC/AppLocker, Device Guard, SmartScreen, ASR 규칙
- **크리덴셜 보호**: Credential Guard, LSA PPL, 브라우저 저장 암호 금지

### 체크 스크립트(개요)

```powershell
# LSA PPL 상태

Get-ItemProperty -Path "HKLM:\SYSTEM\CurrentControlSet\Control\Lsa" -Name RunAsPPL

# Credential Guard(장치 가상화 기반 보안)

Get-CimInstance -ClassName Win32_DeviceGuard -Namespace root\Microsoft\Windows\DeviceGuard |
  Select SecurityServicesConfigured, SecurityServicesRunning
```

---

## gMSA(그룹 관리 서비스 계정) — 서비스 비밀번호 자동 관리

### 장점

- **자동 회전**, **호스트 바인딩**, 인터랙티브 로그인 불가 → **도난 가치 낮음**

### 생성/배포 예시(안전)

```powershell
# KDS 루트 키(포리스트 최초 1회) — 랩임을 가정

Add-KdsRootKey -EffectiveImmediately

# gMSA 생성 (IISApp용 예)

New-ADServiceAccount -Name "gmsa-IISApp" -DNSHostName "ad.lab.local" -PrincipalsAllowedToRetrieveManagedPassword "LAB-IIS01$"

# 대상 서버에서 설치/테스트

Install-ADServiceAccount -Identity "gmsa-IISApp"
Test-ADServiceAccount -Identity "gmsa-IISApp"
```

> 앱 서비스 구성에서 gMSA를 서비스 로그온 계정으로 지정(벤더 가이드 준수).

---

## LDAP Signing & Channel Binding

### GPO 경로

- **도메인 컨트롤러 보안 옵션**
  - *도메인 컨트롤러: LDAP 서버 서명 요구* = 사용
  - *도메인 컨트롤러: LDAP 클라이언트에 채널 바인딩 요구* (OS 버전/패치 기준)

### 점검 스크립트(서버 측)

```powershell
# LDAP 서명/채널 바인딩 관련 레지스트리 살펴보기(참고)

Get-ItemProperty "HKLM:\SYSTEM\CurrentControlSet\Services\NTDS\Parameters" |
  Select "LDAPServerIntegrity" # 2=Require
Get-ItemProperty "HKLM:\SYSTEM\CurrentControlSet\Control\SecurityProviders\Schannel" |
  fl ChannelBinding*
```

**효과**
- LDAP 중간자/릴레이 난이도 상승(서명/채널 바인딩 필요)

---

## SMB 서명/암호화 강제

### 서버 구성 확인

```powershell
Get-SmbServerConfiguration | Select EnableSMB1Protocol,RequireSecuritySignature,EncryptData
```
- **RequireSecuritySignature = True**, **EncryptData = True** 권장
- 클라이언트/서버 GPO로 서명 **필수화**(감사→강제 단계)

**효과**
- SMB 릴레이/중간자 저지, 세션 하이재킹 방어력 향상

---

## LSA 보호(PPL) & Credential Guard

### LSA PPL

- **목적**: LSASS 접근(핸들/메모리 덤프)을 **커널 수준**에서 차단
- **설정**: `HKLM\SYSTEM\CurrentControlSet\Control\Lsa\RunAsPPL = 1` (또는 보안 기준/GPO)
- **점검**: 위 레지스트리 + 이벤트 3033(LSASS 보호)

### Credential Guard

- **VBS 기반 격리**로 **자격 토큰 보호**
- Intune/GPO 보안 기준으로 **구성 상태** 중앙관리

---

## 운영 체크리스트(압축)

**Kerberos / SPN**
- [ ] 도메인 **AES-only**, RC4 비활성
- [ ] SPN 계정 **gMSA** 또는 **장기 비번 회전** 자동화
- [ ] 4769 EncType 분포 **모니터링**(RC4 발생 시 원인 추적)

**ACL / 위임**
- [ ] **DCSync 권한 보유자** 점검/최소화
- [ ] **Unconstrained Delegation 제거**, RBCD/Constrained로 제한
- [ ] **GPO 편집/링크 권한** 재정립(전담 그룹만)

**프로토콜 / 서명**
- [ ] **LDAP Signing/Channel Binding**, **SMB 서명/암호화** 강제
- [ ] **NTLM 축소**: 감사→거부, Kerberos 우선 구성

**Tiering / PAW / LSA**
- [ ] **Tier-0/1/2** 세션 격리, 보호 그룹 워크스테이션 로그인 금지
- [ ] **PAW** 보급, WDAC/AppLocker/ASR/SmartScreen
- [ ] **LSA PPL + Credential Guard** 전면화

**탐지 / SIEM**
- [ ] **TGS 급증**(4769), **야간 비정상 패턴** 룰
- [ ] **NTLM 급증**, **SMB 서명 미사용** 알림
- [ ] **RBCD/Delegation/ACL 변경** 디렉터리 감사 수집

---

## 트러블슈팅 & 예외 관리

- **레거시 앱 때문에 RC4/NTLM 필요**
  → **세그먼트/서버 한정** + **감사 로그 상시** + **마이그레이션 계획/마감일** 문서화.
- **LDAP 서명 강제 후 일부 연동 실패**
  → 해당 클라이언트에 **패치/라이브러리 업그레이드**. 임시 예외는 **만료일** 설정.
- **gMSA 전환 시 애플리케이션 이슈**
  → **테스트 환경에서 서비스 계정 전환 가이드** 확보 후 점진 배포.
- **LSA PPL로 운영툴 장애**
  → 벤더 서명 드라이버/지원 모드 사용. 예외는 **서명/경로/기간 제한** 필수.

---

# 결론 요약

- **Kerberoast**는 “티켓 자체가 재사용 가능한 비밀”이라는 현실에서 출발합니다. **AES-only + gMSA + 짧은 수명 + 모니터링**으로 **유출 가치/시간**을 줄이세요.
- **ACL Misconfig**는 권한 그래프 문제입니다. **DCSync/WriteDACL/Delegation**을 정기 점검해 **경로 자체**를 제거하세요.
- **Tiering/PAW/LDAP·SMB 서명/LSA 보호**를 함께 적용하면, **후방 활동의 연결 고리**가 구조적으로 끊깁니다.
- 본 문서의 스크립트·체크리스트·룰은 **CI/MDM/SIEM 파이프라인**으로 자동화하여 “배포 전 차단 → 배포 후 감시 → 주간 리포트 → 예외 만료”의 **운영 루프**를 완성하세요.
