---
layout: post
title: 시스템보안 - 실습(misconfigured service → SYSTEM, UAC bypass 패턴)
date: 2025-10-22 20:30:23 +0900
category: 시스템보안
---
# 5.5 실습: misconfigured service → SYSTEM, UAC bypass 패턴 따라잡기 (방어 중심)

> 이 섹션은 **공격 재현(Exploit Steps)**이 아니라, **어떻게 발견·설명·수정**할지에 초점을 둡니다.
> 실제 악용을 유도할 수 있는 **정확한 명령 시퀀스·페이로드·주소**는 제공하지 않습니다. 모두 **본인 소유 랩 VM**에서, 오직 **감사/개선** 목적의 스크립트만 사용하세요.

---

## 5.5.1 “SYSTEM이 되는” 서비스 오구성, 어디서 생기나?

### A) 경로 문제(공백 + 따옴표 없음, Unquoted Service Path)
- **징후**: `ImagePath`가 `"C:\Program Files\Vendor App\bin\app.exe"`가 아니라
  `C:\Program Files\Vendor App\bin\app.exe -k arg`처럼 **따옴표 없이 공백 포함**.
- **위험**: 서비스가 `LocalSystem`으로 시작되고, 상위 경로에 사용자 쓰기가 가능하다면 **의도치 않은 실행 파일**이 먼저 로드될 수 있음.
- **방어 요약**:
  1) **항상** 이중 따옴표로 감싸기
  2) 실행 파일/상위 디렉터리 **일반 사용자 쓰기 금지**

### B) 쓰기 가능한 경로(Executable/Directory writable)
- 서비스 바이너리나 상위 디렉터리가 `Users/Everyone` 등에 **Write/Modify** 허용 → **코드 주입 가능 지점**.
- 방어: **소유자=Administrators 또는 SYSTEM**, `Users/Everyone`의 **쓰기 권한 제거**.

### C) 느슨한 서비스 DACL(서비스 객체 권한)
- 서비스 자체의 보안 서술자(SDDL)가 느슨하여 **CONFIG/START/STOP/CHANGE**를 일반 사용자가 수행 가능.
- 방어: **SCM 보안**(서비스 DACL)을 표준 템플릿으로 재설정.

### D) UAC(승격 흐름) 우회 패턴(개념)
- 잘못된 **자동 상승 경로**(예: 일부 autoElevate COM/작업 스케줄러/설치 정책)와 **사용자 쓰기 경로** 결합.
- 방어: **승격 트리거의 실행 파일·경로·스크립트**를 모두 **서명/경로 보호/정책(AppLocker/WDAC)**로 고정.

---

## 5.5.2 랩 감사 스크립트(안전) — 서비스/작업/레지스트리 빠른 헬스체크

> 출력은 **징후**입니다. 발견 즉시 **화이트리스트**(정상 벤더·경로)를 구성하고, 나머지를 **개선 티켓**으로 전환하세요.

```powershell
<# win_misconfig_audit.ps1  — 읽기 전용 점검 (관리자 권장)
   1) Unquoted Service Path
   2) Executable/Directory 쓰기 가능
   3) 서비스 DACL 느슨
   4) SYSTEM 작업 + 쓰기 가능한 실행 경로
   5) UAC 취약 정책(AlwaysInstallElevated)
#>

function Test-Writable {
  param([string]$Path)
  try {
    $acl = Get-Acl -LiteralPath $Path -ErrorAction Stop
    $bad = $acl.Access | Where-Object {
      ($_.IdentityReference -match 'Everyone|Users|Authenticated Users') -and
      ($_.FileSystemRights.ToString() -match 'Write|Modify|FullControl')
    }
    return [bool]$bad
  } catch { return $false }
}

Write-Host "== Services: Unquoted / Writable Evidences =="
Get-WmiObject Win32_Service | ForEach-Object {
  $name = $_.Name; $img = $_.PathName
  $unquoted = ($img -notmatch '^".*"$') -and ($img -match '\s')
  $exe = ($img -replace '^"','' -replace '"$','').Split(' ')[0]
  if(Test-Path $exe){
    $dir = Split-Path $exe -Parent
    $wExe = Test-Writable -Path $exe
    $wDir = Test-Writable -Path $dir
    if($unquoted -or $wExe -or $wDir){
      [PSCustomObject]@{
        Service      = $name
        UnquotedPath = $unquoted
        WritableExe  = $wExe
        WritableDir  = $wDir
        ImagePath    = $img
      }
    }
  }
} | Sort-Object Service | Format-Table -AutoSize

Write-Host "`n== Services: Weak Service DACL (CONFIG/WRITE) =="
# 서비스 DACL은 sc.exe sdshow 활용
$svc = sc.exe query state= all | Select-String "SERVICE_NAME:" | ForEach-Object {
  ($_ -split ':')[1].Trim()
}
foreach($s in $svc){
  $sd = (sc.exe sdshow $s 2>$null)
  if($LASTEXITCODE -eq 0 -and $sd){
    # 간단 탐지: Everyone/Users에 WRITE/CHANGE 권한이 있는지 문자열 기반 점검(보수적)
    if($sd -match 'S-1-1-0' -or $sd -match 'S-1-5-11'){ # Everyone / Authenticated Users
      Write-Host "[*] Review Service DACL:" $s
    }
  }
}

Write-Host "`n== Scheduled Tasks: SYSTEM + Writable Exec Directory =="
Get-ScheduledTask | ForEach-Object {
  $sys = ($_.Principal.UserId -match 'SYSTEM')
  $_.Actions | ForEach-Object {
    if($sys -and $_.Execute -and (Test-Path $_.Execute)){
      $d = Split-Path $_.Execute -Parent
      if(Test-Writable -Path $d){
        [PSCustomObject]@{
          Task      = $_.Id
          RunAs     = "SYSTEM"
          ExecPath  = $_.Execute
          DirWritable = $true
        }
      }
    }
  }
} | Format-Table -AutoSize

Write-Host "`n== UAC Policy Check: AlwaysInstallElevated =="
$keys = @(
 "HKLM:\Software\Policies\Microsoft\Windows\Installer",
 "HKCU:\Software\Policies\Microsoft\Windows\Installer"
)
foreach($k in $keys){
  try {
    $v = Get-ItemProperty -Path $k -Name "AlwaysInstallElevated" -ErrorAction Stop
    if($v.AlwaysInstallElevated -eq 1){ Write-Host "[!] $k AlwaysInstallElevated=1" }
  } catch {}
}
```

**읽는 법**
- `UnquotedPath = True` → 반드시 **따옴표 추가** 및 **경로 쓰기 제한**
- `WritableExe/WritableDir = True` → 소유자/권한 재설정(일반 사용자 쓰기 제거)
- “Weak Service DACL” 출력 → `sc sdshow`/`sdset`로 **표준 템플릿** 재적용

---

## 5.5.3 수정(하드닝) 실습 — 안전하게 고치기

> **주의**: 아래 수정은 **벤더 지원 범위**를 확인하고, 변경 전 **백업** 및 **변경 이력**을 남기세요.

### A) 경로를 항상 이중 따옴표로
```powershell
# 예: ImagePath가 C:\Program Files\Vendor App\bin\app.exe -k start 인 서비스
# 1. 현재 값 확인
$svcKey = "HKLM:\SYSTEM\CurrentControlSet\Services\VendorApp"
(Get-ItemProperty $svcKey).ImagePath
# 2. 따옴표 포함 절대경로로 재설정
Set-ItemProperty $svcKey -Name ImagePath -Value '"C:\Program Files\Vendor App\bin\app.exe" -k start'
# 3. 서비스 재시작(가능하면 유지보수 시간에)
Restart-Service -Name "VendorApp" -Force
```

### B) 실행 파일/디렉터리 ACL 고정(Users/Everyone 쓰기 제거)
```powershell
$exe = "C:\Program Files\Vendor App\bin\app.exe"
$dir = Split-Path $exe -Parent

# 1. 소유자: BUILTIN\Administrators (또는 TrustedInstaller, 벤더 지침 준수)
icacls $dir /setowner "Administrators" /T
# 2. 일반 사용자 쓰기 제거(필요 권한만 남김)
icacls $dir /inheritance:r
icacls $dir /grant:r "SYSTEM:(OI)(CI)(F)" "Administrators:(OI)(CI)(M)"
icacls $dir /remove:g "Users" "Authenticated Users" "Everyone"
# 3. 실행 파일도 동일 기준 적용
icacls $exe /inheritance:r
icacls $exe /grant:r "SYSTEM:(F)" "Administrators:(M)"
icacls $exe /remove:g "Users" "Authenticated Users" "Everyone"
```

### C) 서비스 DACL 표준화 (SDDL)
- **목표**: 일반 사용자의 `SERVICE_CHANGE_CONFIG/WRITE_DAC/START/STOP` 등 **과도 권한 제거**.
- **방법**: `sc sdshow <svc>`로 현재 DACL 확인 → 벤더/조직 표준 SDDL로 `sc sdset <svc>` 적용.

```powershell
# 보기
sc.exe sdshow "VendorApp"
# 적용(예시는 개념용; 실제 조직 표준 SDDL 사용)
# D:(A;;CCLCSWRPWPDTLOCRRC;;;SY)(A;;CCDCLCSWRPWPDTLOCRSDRCWDWO;;;BA)
#  - SY(System), BA(Builtin Admin)만 구성 변경/제어 허용
sc.exe sdset "VendorApp" "D:(A;;CCLCSWRPWPDTLOCRRC;;;SY)(A;;CCDCLCSWRPWPDTLOCRSDRCWDWO;;;BA)"
```

### D) 작업 스케줄러 하드닝
```powershell
# SYSTEM으로 도는 작업의 실행 파일 디렉터리 쓰기 차단
Get-ScheduledTask | Where-Object { $_.Principal.UserId -match 'SYSTEM' } | ForEach-Object {
  $_.Actions | ForEach-Object {
    if($_.Execute -and (Test-Path $_.Execute)){
      $d = Split-Path $_.Execute -Parent
      icacls $d /inheritance:r
      icacls $d /grant:r "SYSTEM:(OI)(CI)(F)" "Administrators:(OI)(CI)(M)"
      icacls $d /remove:g "Users" "Authenticated Users" "Everyone"
    }
  }
}
```

### E) UAC 취약 정책 제거
```powershell
# AlwaysInstallElevated 끄기 (HKLM/HKCU 모두)
$keys = @(
 "HKLM:\Software\Policies\Microsoft\Windows\Installer",
 "HKCU:\Software\Policies\Microsoft\Windows\Installer"
)
foreach($k in $keys){
  if(!(Test-Path $k)){ New-Item -Path $k -Force | Out-Null }
  Set-ItemProperty -Path $k -Name "AlwaysInstallElevated" -Value 0
}
```

---

## 5.5.4 UAC bypass 패턴 “이해하고 막기”(재현 대신 방어 설명)

### 패턴 1) **자동 상승 경로(autoElevate) + 경로/구성 취약**
- 일부 시스템 컴포넌트는 **서명/정책** 하에 자동 상승.
- 만약 이들이 참조하는 실행 파일/스크립트/COM 서버 경로가 **사용자 쓰기 가능**이면 **우회 여지**.
- **방어**:
  - **서명 강제( WDAC/AppLocker )**로 비서명 바이너리/스크립트 차단
  - autoElevate 체인에 등장하는 경로를 **Program Files, System32** 등 보호 디렉터리로 고정
  - **LoadLibrary/COM 등록 경로**는 **절대경로 + 서명** 전제

### 패턴 2) **COM/MMC/WMI 초기화 타이밍 악용**
- 레거시 등록/바인딩 과정에서 **검색 순서**나 **권한 위임**이 빈틈을 보일 수 있음.
- **방어**:
  - **SetDefaultDllDirectories + LoadLibraryEx 플래그**(개발팀)
  - **COM 서버 경로 보호 + WDAC 규칙**
  - **WMI 영구 구독** 정기 점검(root\subscription)

### 패턴 3) **설치 정책/작업 스케줄러 조합**
- AlwaysInstallElevated, 잘못된 작업 정의, PATH 오염이 결합.
- **방어**: 위 하드닝 절차 + **EDR/Sysmon**으로 등록/실행 이벤트 모니터

---

# 5.6 방어: LSA 보호, Credential Guard, AppLocker/WDAC, Attack Surface Reduction

> 이 절의 목표는 “**운영 표준**으로 만들기 좋은 설정/스크립트/정책 템플릿”을 제공하는 것입니다.

---

## 5.6.1 LSA 보호(Protected Process Light, RunAsPPL)

### 개념
- **LSASS**를 **PPL(보호 프로세스)**로 실행 → 비서명/비보호 프로세스의 **메모리/핸들 접근 차단**.
- **효과**: 메모리 덤프/핸들 오남용을 크게 억제.

### 배포(랩 기준 예시)
```powershell
# 레지스트리 설정: 재부팅 필요
New-ItemProperty -Path "HKLM:\SYSTEM\CurrentControlSet\Control\Lsa" -Name "RunAsPPL" -Value 1 -PropertyType DWord -Force
# 부팅 초기에도 보호: (선택) RunAsPPLBoot=1
New-ItemProperty -Path "HKLM:\SYSTEM\CurrentControlSet\Control\Lsa" -Name "RunAsPPLBoot" -Value 1 -PropertyType DWord -Force
```

### 검증
```powershell
tasklist /FI "IMAGENAME eq lsass.exe" /V
# PPL이면 EDR/보안도구 외 접근이 차단됨(운영에서는 EDR 예외/서명 장치 필요)
```

**운영 팁**
- 백업/메모리 진단 도구 등 **합법적 접근**은 **서명·드라이버 기반** 절차로 전환
- PPL 활성 상태에서 **서드파티 보안도구 호환성** 점검

---

## 5.6.2 Credential Guard (VBS 기반 자격 격리)

### 개념
- NTLM/커버로스 비밀을 **가상화 기반(VSM)**에 격리 저장 → LSASS 공격시에도 핵심 자격 정보 유출 방지.
- **요구 사항**: 하드웨어(VT-x/AMD-V, SLAT), OS SKU/버전, UEFI/Secure Boot 등.

### 배포(개요: GPO/MDM 권장, 랩 수동 예시)
```powershell
# 장치가 VBS/Hyper-V 적합한지 사전 점검 필요
# 그룹 정책: 컴퓨터 구성 > 관리 템플릿 > 시스템 > Device Guard / Credential Guard
# 수동 레지스트리 (랩용 개념):
New-Item -Path "HKLM:\SYSTEM\CurrentControlSet\Control\DeviceGuard" -Force | Out-Null
New-ItemProperty -Path "HKLM:\SYSTEM\CurrentControlSet\Control\DeviceGuard" -Name "EnableVirtualizationBasedSecurity" -Value 1 -PropertyType DWord -Force | Out-Null
New-Item -Path "HKLM:\SYSTEM\CurrentControlSet\Control\Lsa" -Force | Out-Null
New-ItemProperty -Path "HKLM:\SYSTEM\CurrentControlSet\Control\Lsa" -Name "LsaCfgFlags" -Value 1 -PropertyType DWord -Force | Out-Null
# 재부팅 이후 systeminfo 등으로 확인
```

### 확인
```powershell
# Windows 10/11: 보안 센터, systeminfo, 또는 PowerShell 전용 모듈로 상태 확인
systeminfo | findstr /i "Virtualization Guard Credential"
```

**운영 팁**
- 도메인 환경: **패스더해시/티켓 재사용** 난이도 상승
- 호환성 이슈(구형 드라이버/보안툴) 사전 검증 필수

---

## 5.6.3 AppLocker / WDAC (정책으로 실행을 “허용 목록”화)

### AppLocker (파일/패키지 규칙, 엔터프라이즈에서 간편)
- **규칙 만들기(간단)**
```powershell
# 기본 규칙 생성(관리자/Program Files/Windows 허용) + 감사 모드
New-AppLockerPolicy -DefaultRule -RuleType EXE,Script,MSI,AppX -User 'Everyone' -XMLPolicy C:\applocker_default.xml
Set-AppLockerPolicy -XMLPolicy C:\applocker_default.xml -Merge
# 감사 모드 권장 → Event Viewer(Microsoft-Windows-AppLocker/EXE and DLL) 확인 후 엄격 모드 전환
```

- **엄격 모드 전환**
```powershell
# 감사에서 거부로(배포 전 파일/서명 기준 화이트리스트 충분히 수집)
Set-AppLockerPolicy -XMLPolicy C:\applocker_approved.xml -Enforce
```

### WDAC (서명 기반 커널/사용자 모드 코드 무결성, 더 강력/정교)
- **기본 정책 생성(감사 모드)**
```powershell
# Windows Defender Application Control: New-CIPolicy
$Out = "C:\wdac_base.xml"
New-CIPolicy -Level FilePublisher -Fallback Hash -FilePath $Out -UserPEs 3 -MultiplePolicyFormat
# 감사 모드로 변환
ConvertFrom-CIPolicy $Out "C:\wdac_base.bin"
Set-RuleOption -FilePath $Out -Option 3  # 3: Audit Mode
# 정책 서명/배포(기업 표준에 맞추어 서명 모듈/CI 포함)
```

- **팁**
  - **서명 우선**, 임시 예외는 **해시**
  - 운영 전 **광범위 감사 수집** → 정상 애플리케이션 화이트리스트 충분화
  - 드라이버/보호 소프트웨어와 **호환성 테스트**

---

## 5.6.4 Attack Surface Reduction(ASR) — 즉효 수비

> Microsoft Defender의 **ASR 규칙**은 자주 악용되는 **Living-off-the-Land** 경로를 차단합니다.
> (Defender가 아니라면 유사 기능을 가진 EDR의 정책 사용)

### 대표 규칙(예: 일부 GUID)
- **Office에서 자식 프로세스 생성 차단**: `{D4F940AB-401B-4EFC-AADC-AD5F3C50688A}`
- **WMI/PsExec 등 프로세스 생성 차단**: `{D3E037E1-3EB8-44C8-A917-57927947596D}`
- **스크립트 기반 공격 차단**: `{5BEB7EFE-FD9A-4556-801D-275E5FFC04CC}`
- **LSASS 인증자격 유출 차단(자격 훔치기)**: `{9E6C4E1F-7D60-4721-A4B2-BC0D3F6D9D3E}` (환경/버전에 따라 가용성 상이)

### 배포(감사 → 차단)
```powershell
# 감사 모드로 먼저
Set-MpPreference -AttackSurfaceReductionOnlyExclusions @()
Add-MpPreference -AttackSurfaceReductionRules_Ids D4F940AB-401B-4EFC-AADC-AD5F3C50688A -AttackSurfaceReductionRules_Actions AuditMode
Add-MpPreference -AttackSurfaceReductionRules_Ids D3E037E1-3EB8-44C8-A917-57927947596D -AttackSurfaceReductionRules_Actions AuditMode
Add-MpPreference -AttackSurfaceReductionRules_Ids 5BEB7EFE-FD9A-4556-801D-275E5FFC04CC -AttackSurfaceReductionRules_Actions AuditMode

# 운영 전환 시(업무 영향 검토 후)
Add-MpPreference -AttackSurfaceReductionRules_Ids D4F940AB-401B-4EFC-AADC-AD5F3C50688A -AttackSurfaceReductionRules_Actions Enabled
Add-MpPreference -AttackSurfaceReductionRules_Ids D3E037E1-3EB8-44C8-A917-57927947596D -AttackSurfaceReductionRules_Actions Enabled
Add-MpPreference -AttackSurfaceReductionRules_Ids 5BEB7EFE-FD9A-4556-801D-275E5FFC04CC -AttackSurfaceReductionRules_Actions Enabled
```

**운영 팁**
- **감사 로그**를 일정 기간 수집 → **업무 오탐** 제외 목록(Exclusions) 최소화 등록
- ASR은 **사용자 불만**이 나올 수 있음 → 변경관리/커뮤니케이션 필수

---

## 5.6.5 모니터링 & 포렌식 신호 (Sysmon/이벤트 채널)

- **Sysmon 권장 이벤트**
  - **1**(Process Create): 서비스/작업에서 비정상 경로 실행, `cmd.exe`/`rundll32.exe` 자식
  - **7**(Image Load): 비서명 DLL 로딩
  - **10**(Process Access): `lsass.exe` 접근(핸들 요청)
  - **12/13/14**(Registry): 서비스/Run 키 변경
  - **19/20/21**(WMI): Filter/Consumer/Binding 생성(영구 구독)
- **Windows 로그**
  - 서비스 설치/변경: **4697**, **7045**
  - 권한 사용(특권): **4672**
  - 작업 스케줄러 Operational 채널: 작업 등록/실행/실패

---

## 5.6.6 “운영에 녹이는” 체크리스트

- **서비스**
  - [ ] ImagePath **이중 따옴표 + 절대경로**
  - [ ] 실행 파일/디렉터리 **Users/Everyone 쓰기 금지**
  - [ ] Service DACL: **SY/BA 중심**, 일반 사용자 WRITE 금지
- **스케줄러**
  - [ ] SYSTEM/관리자 작업의 실행 경로 디렉터리 **쓰기 금지**
  - [ ] 상대경로·환경 변수 의존 제거
- **UAC/정책**
  - [ ] `AlwaysInstallElevated` **비활성(HKLM/HKCU)**
  - [ ] 승격 경로는 **서명/WDAC/AppLocker**로 통제
- **코드 로딩**
  - [ ] 앱에서 `SetDefaultDllDirectories` 호출
  - [ ] `LoadLibraryEx(..., LOAD_LIBRARY_SEARCH_SYSTEM32|...)`
- **자격 보호**
  - [ ] LSA **RunAsPPL**
  - [ ] **Credential Guard**(가능 환경)
  - [ ] EDR/Sysmon으로 `lsass.exe` 접근 탐지
- **정책**
  - [ ] **AppLocker/WDAC** 감사 → 거부 전환
  - [ ] **ASR** 감사 → 활성 전환
- **거버넌스**
  - [ ] 변경관리(위험 서비스/작업/레지스트리 수정 시 CAB/티켓)
  - [ ] 주간 리포트(신규 SUID? X / 신규 서비스/작업/COM 등록 변화 O)

---

## 5.6.7 트러블슈팅(현장에서 자주 묻는 문제)

- **서비스가 시작되지 않아요 (권한 바꾼 뒤)**
  - 이벤트 로그에서 “Access is denied” 확인 → 벤더 문서의 **필수 권한** 재적용(특정 서비스 계정에 `READ & EXECUTE` 필요)
- **AppLocker/WDAC로 정상 앱이 막혀요**
  - 감사 로그 기반 **서명 발행자 규칙** 추가(해시보다 유지보수 용이)
  - 업데이트 채널/경로를 **고정된 디렉터리**로 집약
- **ASR 켰더니 매크로/자동화가 멈춰요**
  - **감사 기간** 충분히 두고 업무 플로우 식별 → 최소한의 **Exclusion** 등록
  - возможно “Office 자식 프로세스 차단”은 가장 효과/부작용 모두 큰 규칙 → 우선순위 평가

---

# 마무리

- “misconfigured service → SYSTEM”은 **경로·권한·정책**의 합성 문제입니다.
- UAC bypass도 **자동 상승 트리거**와 **경로/서명/권한**의 결합 실패에서 나옵니다.
- **LSA 보호/자격 격리/허용 목록(앱 제어)/ASR**을 **겹겹이** 적용하면, 실무에서의 로컬 LPE/자격 탈취 리스크가 **현저히 감소**합니다.
- 위 스크립트·설정 템플릿을 **사내 표준 Runbook/GPO/CI**에 녹이면, 지속적으로 안전한 상태를 유지할 수 있습니다.
