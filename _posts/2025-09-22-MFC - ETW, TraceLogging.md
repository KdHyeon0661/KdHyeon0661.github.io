---
layout: post
title: MFC - ETW, TraceLogging
date: 2025-09-22 18:25:23 +0900
category: MFC
---
# ETW/TraceLogging 성능 분석 완전 가이드
**사용자 시나리오 트레이스 → WPA 분석, 프레임타임/할당(“GC 없는 C++”) 히트맵, I/O 큐 병목 찾기**

> 목표: “**실사용 시나리오를 정확히 재현해 한 번의 .etl로 전체 병목을 잡는다**”
> 도구: **ETW(Event Tracing for Windows)**, **TraceLogging**(경량 계측), **WPR/WPA**, (선택) **xperf**, **PresentMon**(DX 프레임 참고)

---

## 0. 설계 개요 (파이프라인)

1. **앱 계측**: TraceLogging(경량)으로 **시나리오 경계(START/STOP)**, **세부 태스크(활동)**, **핵심 파라미터**를 로깅
2. **수집**: **WPR**로 **Kernel + User** 트레이스 프로필( CPU 샘플링, 스택, File I/O, Disk, Networking, GPU/Present, Heap )을 한 번에 켠다
3. **재현**: 사용자가 실제로 느린 경로를 수행(“파일 열기 → 썸네일 생성 → 스크롤”)
4. **중지/저장**: `.etl` 획득 (필요 시 PDB 심볼 준비)
5. **WPA 분석**:
   - **시나리오 마커**를 기준으로 관심 구간 선택
   - **CPU 샘플링**(Hot path + 콜스택)
   - **프레임타임**(DXGI/DWM Present) 히스토그램/스파이크
   - **Heap(할당)** 히트맵/수명/누수/과도한 임시 객체
   - **I/O 큐** 길이·대기(Service Time vs Wait Time)
   - **스레드 컨텐션/Ready Time** 확인
6. **수정 → 재측정**: 변경 전/후 **ETL 비교**로 회귀 방지

---

## 1. TraceLogging 계측(시나리오 마커)

### 1.1 최소 계측(시나리오 시작/끝)
```cpp
// NuGet: Microsoft.TraceLogging.Dynamic 또는 VS 내장 TraceLogging 헤더
#include <TraceLoggingProvider.h>
#include <winmeta.h>
TRACELOGGING_DECLARE_PROVIDER(g_MyProvider);
TRACELOGGING_DEFINE_PROVIDER(g_MyProvider, "MyApp", // friendly name
    (0x9a4a1c9d,0x5e66,0x4b7e,0x9d,0x04,0x16,0x9a,0x0e,0x40,0x8c,0xdf)); // 새 GUID 생성

struct ScenarioGuard {
    GUID act{};
    ScenarioGuard(const char* name, const char* user, int docCount) {
        CoCreateGuid(&act);
        TraceLoggingWriteActivity(g_MyProvider, "Scenario",
            &act, nullptr, // ActivityId, RelatedActivityId
            TraceLoggingString(name, "Name"),
            TraceLoggingString(user, "User"),
            TraceLoggingInt32(docCount, "DocCount"),
            TraceLoggingOpcode(WINEVENT_OPCODE_START));
    }
    ~ScenarioGuard() {
        TraceLoggingWriteActivity(g_MyProvider, "Scenario",
            &act, nullptr,
            TraceLoggingOpcode(WINEVENT_OPCODE_STOP));
    }
};

int RunOpenAndScroll() {
    TraceLoggingRegister(g_MyProvider);
    ScenarioGuard s("Open+Scroll", "testerA", /*docCount*/24);
    // 실제 동작: 파일 열기→썸네일→스크롤
    // ...
    TraceLoggingUnregister(g_MyProvider);
    return 0;
}
```

- **ActivityId**로 **자식 태스크**(디코딩/네트워크/스레드풀 작업)와 **관계**를 이어줄 수 있음.
- 시나리오 경계를 두면 WPA에서 **“타임라인 슬라이스”** 가 쉬워짐.

### 1.2 하위 태스크와 관련 활동(RelatedActivityId)
```cpp
void DecodeThumbs(const GUID& parent) {
    GUID act{}; CoCreateGuid(&act);
    TraceLoggingWriteActivity(g_MyProvider, "DecodeThumbs", &act, &parent,
        TraceLoggingOpcode(WINEVENT_OPCODE_START));
    // 디코딩 작업...
    TraceLoggingWriteActivity(g_MyProvider, "DecodeThumbs", &act, &parent,
        TraceLoggingOpcode(WINEVENT_OPCODE_STOP));
}
```

---

## 2. 수집: WPR 프로필 (& WPRP 예시)

### 2.1 빠른 명령형(권장)
```powershell
# Admin PowerShell
# 1. 시작: CPU 샘플+스택, 디스크/파일 I/O, 네트워크, GPU, 힙(할당), CLR off
wpr -start CPU -start FileIO -start DiskIO -start Networking -start GPU -heap -stackwalk profile

# 추가로 NT Kernel + DxgKrnl Present, DWM 등은 기본 프로필 포함
# 2. 재현(앱에서 시나리오 실행)
# 3. 중지/저장
wpr -stop trace.etl
```

> **팁**
> - `-heap`은 **프로세스 대상**을 줄일 수 있음: `wpr -heap -providers "Heap" -processnames MyApp.exe`
> - `-filemode` 로 링 버퍼 대신 파일에 직기(긴 세션)
> - 심볼: `setx _NT_SYMBOL_PATH "srv*C:\symbols*https://msdl.microsoft.com/download/symbols"`

### 2.2 WPR UI (WPRUI.exe)
- Profiles: **General** + **GPU** + **File I/O** + **Memory(Heap)** 선택, *Stack* 체크
- **Scenario** 탭: **“Start”** → 앱 조작 → **“Save”**

### 2.3 WPRP (커스텀 프로필) 일부
```xml
<!-- MyPerf.wprp -->
<WPRProfile Version="1.0" Name="MyPerf">
  <Profiles>
    <Profile Name="MyPerfProfile" LoggingMode="File" DetailLevel="Verbose">
      <Collectors>
        <EventCollector Id="Kernel" Name="NT Kernel Collector" StackCaching="true"/>
        <EventCollector Id="Heap" Name="Heap Collector" StackCaching="true"/>
        <EventCollector Id="User" Name="User Provider Collector" StackCaching="true"/>
      </Collectors>
      <Providers>
        <KernelProvider Id="Cpu" Name="Cpu" Keywords="Process Thread Image Load Profile" Stack="true"/>
        <KernelProvider Id="FileDisk" Name="FileDisk" Keywords="FileIO DiskIO" Stack="true"/>
        <Provider Id="DxgKrnl" Guid="{802ec45a-1e99-4b83-9920-87c98277ba9d}" Keywords="0xFFFFFFFFFFFFFFFF" Stack="true"/>
        <Provider Id="DwmCore" Guid="{9e9bba3c-2e38-40cb-99f4-9e8281425164}"/>
        <Provider Id="Heap" Guid="{32634E98-402F-4E37-9E7A-D1B8694B2D3C}" Level="5" Stack="true"/> <!-- 예시: Heap Trace -->
        <Provider Id="MyApp" Name="MyApp" Level="5"/> <!-- TraceLogging -->
      </Providers>
      <SystemCollector Id="System" Name="NT Kernel Logger"/>
      <CollectionPlan>
        <EventCollectorRef Id="Kernel">
          <SystemProviderRef Id="Cpu"/>
          <SystemProviderRef Id="FileDisk"/>
        </EventCollectorRef>
        <EventCollectorRef Id="Heap">
          <ProviderRef Id="Heap"/>
        </EventCollectorRef>
        <EventCollectorRef Id="User">
          <ProviderRef Id="DxgKrnl"/>
          <ProviderRef Id="DwmCore"/>
          <ProviderRef Id="MyApp"/>
        </EventCollectorRef>
      </CollectionPlan>
    </Profile>
  </Profiles>
</WPRProfile>
```

> **주의**: Heap/스택은 이벤트량이 큼 → **재현 시간을 짧게**(수십 초~수 분).

---

## 3. WPA(Windows Performance Analyzer) 분석 루틴

### 3.1 타임라인 슬라이싱(시나리오 마커로 범위 고정)
- **Generic Events** → `ProviderName = MyApp` → `EventName = Scenario`
- Opcode=Start/Stop 쌍 확인 → 마우스로 해당 구간 **Time Selection**
- 아래 모든 그래프는 **Time Selection**을 기준으로 집계됨(같은 구간만 보게 하라)

### 3.2 CPU 샘플링: Hot Path, Ready Time
- **Graph Explorer → CPU Usage (Sampled)**
  - **By: Process, Thread, Stack**
  - **주요 칼럼**: `% Weight`, `New Process`, `New Thread`, `Stack`, `Module!Function`
  - **Hot 함수**를 찾아 **Inclusive/Exclusive** 비교
- **CPU Usage (Precise)** / **Thread Activity**에서 **Ready Time**(스레드가 실행 대기한 시간) ↑는 **컨텐션/스케줄링 병목** 신호

### 3.3 프레임타임(게임/미디어/리치 UI)
- **Windows Graphics** → **GPU Utilization**
- **DxgKrnl** → **Present**(또는 **Microsoft-Windows-DxgKrnl** Provider 기반 **PresentMon** 등)
  - **Display**: `Frame Time(ms)`, `App Miss`, `Display Miss`, `Sync Interval`, `PresentMode`
  - **히스토그램**: 16.7ms(60fps) 레일, 33ms 이상 스파이크 **원인**을 **다른 그래프와 상관**(CPU/FileIO/GCish Alloc)

> **DirectX 앱**이라면 **PresentMon**(ETW 기반)으로 **프레임타임/여러 큐 상태**를 별도 보기 굿.

### 3.4 할당(“GC는 없지만, 할당은 있다”) — Heap 히트맵/누수
- **Memory** → **Heap Allocations (with stacks)**
  - **By: Process, Call stack**
  - 칼럼: `Allocation Size`, `Outstanding Size`(잔존), `Count`, `Lifetime`
  - **수명 히트맵**(Allocation vs Free 타임라인)을 열어 **짧은 수명 반복 할당**(churn) 또는 **누수성 증가**(Outstanding 증가)를 확인
- **패턴**
  - 프레임 경계마다 큰 할당/해제가 반복? → **풀링/예약(reserve)**, **소규모 객체 모아쓰기(SBO/arena)**
  - 특정 경로 **Outstanding 증가**만 계속? → **해제 누락** 또는 **참조 cycle** 의심

### 3.5 파일/디스크 I/O 큐 병목
- **Storage** → **Disk Usage** / **File I/O**
  - 칼럼: `IO Time`, `Queue Length`, `Irp Flags`, `FileName`, `Process/Thread`
  - **Service Time vs Wait**: 디스크 자체가 느린지(서비스 시간↑) / 큐가 많아 대기인지(대기↑)
  - **“랜덤 작은 읽기”** 폭증이면 **리드어헤드/캐시/배치** 전략 필요
  - **동기 I/O로 UI 스레드 블락**? → **Async + Awaitable(OVERLAPPED/IOCP)** 로 전환

### 3.6 네트워크
- **Networking** → **TCP/IP**
  - RTT/재전송, 소켓 큐 확인 → 서버/클라 병목 분리

### 3.7 스레드 블로킹/락
- **Computation** → **CPU Usage (Sampled)** + **Synchronization**
  - Wait chain, `CriticalSection` 과다 대기 시점 파악
  - **Alternatives**: lock-free 구조, 더 큰 임계구역 쪼개기, 조건변수 → 이벤트 기반

---

## 4. 프레임타임/히스토그램 정밀 팁

- **PresentMode**: `Hardware: Independent Flip` vs `Composed: Flip` 등 각 모드의 **플립 경로**
- **App Miss**: 앱이 제시간에 프레임을 못 만든 것(보통 CPU/GPU 과부하)
- **Display Miss**: DWM/디스플레이가 미스 → **프레임 큐 관리/Sync Interval** 조정
- **상관 분석**: **사용자 시나리오 마커** + **Present Table** + **CPU/Heap/File** 동시 정렬

---

## 5. “힙/할당 히트맵” 읽는 법(실전)

- **가로축**: 시간, **세로축**: 할당 이벤트 스트림(스택 그룹)
- **점**: 할당(진한색) vs 해제(연한색)
- **패턴**
  - **빗살무늬**: 고주기 임시 할당/해제 → **reserve**/object pool 도입
  - **밴드가 위로만**: 해제가 없는 경로(누수/메모리 축적)
  - **대형 덩어리**: 아주 큰 할당이 드문/느림 → **가상 메모리 예약 + 커밋 단위 조정** 검토

---

## 6. I/O 큐 길이 진단(디스크 병목 레시피)

1) **Disk Usage** 그래프에서 **Queue Length**를 **Time**에 그려 **피크** 확인
2) **File I/O** 표로 내려가 **Top files** / **Top call stacks** 도출
3) I/O Size/패턴 확인 → **128KB 이상 시퀀셜**로 묶을 방법 있는지
4) UI 스레드인지? → **WPA의 Thread column**에서 UI 스레드 ID 매핑
5) **캐시 사용**(FILE_FLAG_SEQUENTIAL_SCAN / FILE_FLAG_OVERLAPPED) 전략 적합성 검토

---

## 7. 스택/심볼 준비

- PDB가 없으면 **cpu sample stack**이 함수명 미표시 → `_NT_SYMBOL_PATH` 설정
- 사내 빌드라면 **Source Link** / **SRV*cache*MSDL** 병행
- WPR Stop 후 WPA에서 **Trace → Load Symbols** 또는 자동 로드

---

## 8. 사례별 “레시피”

### 8.1 스크롤 스터터링(프레임 스파이크)
- 증상: 200ms 스파이크가 간헐
- 절차: **Present graph**에서 스파이크 프레임 클릭 → 같은 구간 **CPU/Heap/File** 교차
- 결과 예: **이미지 디코더**가 UI 스레드에서 **동기 File I/O + 할당**
- 처방: 디코딩 **백그라운드 + 메모리 풀** / 프리로드 후 **UI에 포인터 전달**

### 8.2 썸네일 생성 느림(I/O 큐↑)
- **Disk Queue Length** 치솟고 **File I/O** size 4KB 랜덤
- 처방: **배치 리드**, **WIC 디코딩 스트라이프** 증가, **OS 캐시 힌트** 추가

### 8.3 메모리 증가/Out-of-memory
- Heap Outstanding Size 지속 증가, 특정 스택 그룹
- 처방: 해당 경로 **유한한 캐시/eviction**, 스마트 포인터 순환 참조 차단, 대형 버퍼 **재사용**

---

## 9. xperf 대체(레거시/스택워크 세부)

```powershell
# 커널 + 디스크/파일 + 프로파일 샘플 + 스택워크
xperf -on PROC_THREAD+LOADER+PROFILE+DISK_IO+FILE_IO -stackwalk Profile+CSwitch+ReadyThread+DiskReadInit+DiskWriteInit
# 재현 …
xperf -d trace.etl
```

- 분석은 WPA로 동일하게 열기(혹은 xperfview)

---

## 10. 체크리스트(수집 품질)

- [ ] **시나리오 마커**(Start/Stop/ActivityId) 있음
- [ ] **Stack** 켰는가(핫링크 찾기 필수)
- [ ] **Heap**은 꼭 필요한 구간만(용량 폭증 주의)
- [ ] **심볼 경로** 준비
- [ ] 재현 시 **네트워크/디스크 캐시 상태** 초기화 고려(다음 런 간 차이 줄이기)

---

## 11. 고급: 활동 상관(ETW ActivityId)

- `TraceLoggingWriteActivity`의 **ActivityId/RelatedActivityId**로 **부모-자식** 연결
- WPA: **Generic Events** 테이블에 **ActivityId** 컬럼 표시 → **CPU/Thread** 뷰와 조인(“Selection -> Filter To Selection”)

---

## 12. 최소 예제: TraceLogging + WPR + WPA

1) 위 **ScenarioGuard**로 앱에 시나리오 마커
2) `wpr -start CPU -start DiskIO -start FileIO -heap -stackwalk profile`
3) 앱에서 **Open+Scroll** 수행 → `wpr -stop trace.etl`
4) WPA에서 열기 → **Generic**에서 Scenario 슬라이스 → **CPU/Present/Heap/File** 순회
5) 병목 스택 복사 → 코드 최적화 → **같은 조건 재측정**

---

## 13. 최적화 가이드(관찰→행동)

- **CPU**: 상위 5 스택의 **Exclusive** 줄이기(알고리즘/자료구조/브랜치/메모리 접근)
- **프레임**: **메인 스레드**에서 장기 작업 금지, **프레임 예산(16.7ms/8.3ms)** 엄수
- **할당**: 프레임 내 **할당 0** 목표(풀/arena/소규모 SBO), 문자열/컨테이너 **reserve**
- **I/O**: **비동기** + **배치/리드어헤드** + **캐시 힌트**
- **락**: 긴 임계구역을 **분할**, 읽기 대부분이면 **SRWLock shared** 적극 사용

---

## 14. 자주 묻는 오해 교정

- **“C++는 GC가 없으니 메모리 문제는 없다”** → **할당 히트맵** 보면 논리적 “GC-유사” 이슈(짧은 수명 객체 폭주)가 성능을 갉아먹음
- **“FPS만 보면 된다”** → **프레임 변동(파형)과 스파이크**가 UX를 좌우. 히스토그램/퍼센타일로 보라
- **“디스크가 SSD라 문제 없다”** → **랜덤 4K 수만 건**은 SSD도 느리다. 큐·대기·서비스 시간으로 실증

---

## 15. 마무리

- 한 번의 ETL로 **전체 병목 지형**(CPU/프레임/할당/I/O/락)을 본다
- **TraceLogging 시나리오 마커**로 분석 범위를 짧고 정확하게
- **WPA**의 **교차 필터/조인**으로 **원인-결과**를 연결
- **고칠 것**은 **상위부터**, **재현/재측정**으로 회귀 차단
