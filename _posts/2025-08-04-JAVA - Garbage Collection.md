---
layout: post
title: Java - Garbage Collection
date: 2025-08-04 21:20:23 +0900
category: Java
---
# Garbage Collection(GC)

## 왜 GC인가? (요약 재정리)

- **자동 메모리 관리**: 명시적 `free` 없이 가비지 회수 → 메모리 안전성↑.
- **도달성 기반**: 루트로부터 **도달 불가능**해진 객체를 가비지로 간주.
- **목표의 균형**: **지연시간(Stop-the-World)**, **처리량(Throughput)**, **메모리 사용량** 사이의 트레이드오프.

---

## 힙/영역/루트/도달성: 정확히 이해하기

### JVM 메모리 큰 그림

```
[Thread Stacks]   [Native(Direct) Memory]   [Code Cache]
        \                   |                     |
         \                  |                     |
          +-----------------+---------------------+
                              [Java Heap]
                     +---------------------------+
                     |        Regions/Spaces     |
                     |  Eden | S0 | S1 |   Old   |
                     +---------------------------+
```

- **Java Heap**: GC 대상(Young/Old 또는 Regions).
- **Metaspace**: 클래스 메타데이터(힙 외).
- **Direct/Native Memory**: NIO DirectBuffer, JNI 등(힙 외).
- **Code Cache**: JIT 컴파일 코드.

### GC Roots

- 각 스레드의 **스택 레퍼런스**, **JNI 글로벌 레퍼런스**, **static 필드**, **JVM 내부 루트** 등.
- **Reachability(도달성)**: GC Root에서 **추적 가능한 객체만** “살아있다”고 정의.

### Mark–Sweep–Compact와 트라이컬러 모델

- **Mark**: 도달 객체에 “검은색(visited)” 표시
- **Sweep**: 표시 안 된(흰색) 객체 회수
- **Compact**: 조각난 메모리 **연속화** (단편화 완화)

> G1/ZGC/Shenandoah 등 동시 수집기는 **tri-color invariant**(White/Gray/Black)와 **Barrier**(write 또는 load)로 동시성 중 일관성을 유지합니다.

---

## 할당 경로와 승격(Promotion)

- **TLAB(Thread-Local Allocation Buffer)**: 대부분의 소객체는 **스레드 지역 버퍼**에서 초고속 할당(포인터 bump).
- **Young → Old 승격**: 서바이버(S0/S1)에서 **나이(tenuring age)**를 채우거나 Eden 압박 시 Old로 이동.
- **대형 객체(huge/humongous)**: 수집기별로 **특별 처리**(예: G1의 humongous region).

---

## 수집기별 동작 정리 (선택 가이드 포함)

| 수집기 | 핵심 포지션 | 장점 | 주의/제약 | 대표 옵션 |
|---|---|---|---|---|
| **Serial** | 단일 스레드, 소형 힙 | 단순/작은 오버헤드 | STW 길어짐 | `-XX:+UseSerialGC` |
| **Parallel** (PS/PO) | 처리량 지향 | Young/Old 병렬 | 지연시간↑ | `-XX:+UseParallelGC` |
| **CMS** *(과거)* | 저지연(동시 마크/스윕) | 짧은 Pause | 단편화/복잡성 → 현대엔 **G1 대체** | (구판) |
| **G1** *(기본에 근접)* | 지역화된 리전, **목표 Pause** | 예측가능한 pause, 대용량 힙 | 튜닝 포인트 많음 | `-XX:+UseG1GC` |
| **ZGC** | **초저지연**, TB 힙 | Load Barrier, 거의 짧은 STW | OS/CPU 제약/최신 JDK 권장 | `-XX:+UseZGC` |
| **Shenandoah** | 초저지연 | Brooks ptr, 동시 compaction | 설정/플랫폼 고려 | `-XX:+UseShenandoahGC` |
| **Epsilon** | **No-Op GC** | 마이크로벤치, 테스트 | 회수 없음 → OOM | `-XX:+UseEpsilonGC` |

### G1 GC 한 장 요약

- 힙을 **고정 크기 Region**으로 분할(1~32MB).
- **Young GC**(Eden→Survivor/Old 복사) + 주기적 **Mixed GC**(Old 일부 포함)로 힙 관리.
- **RSet(Remembered Set)** + **Card Table** + **SATB(Snapshot-At-The-Beginning) Write Barrier**로 동시/병렬 마킹.
- 목표 지연시간: `-XX:MaxGCPauseMillis=200` (기본) 등으로 예측 가능한 pause.

### ZGC / Shenandoah 초저지연 개요

- **ZGC**: 컬러드 포인터/Load Barrier 기반 **동시 Relocation**. STW는 루트 스캔 짧게.
- **Shenandoah**: 로드 배리어나 브룩스 포인터로 **동시 Compaction** 구현.
- 최신 JDK에서는 **Generational ZGC** 모드(세대 구분)도 제공 → Young 집중 회수로 효율↑.

---

## GC 트리거, 실패, 병목 패턴

### 일반 트리거

- **Young 부족** → Minor/Young GC
- **Old 부족** → Mixed/Full GC
- **System.gc()** → (명시 요청; 대부분 비권장)

### 실패/경고 지표

- **Promotion Failure**: Young에서 Old로 승격할 공간 부족 → 긴 STW/FullGC 유발.
- **To-space Exhausted**: 복사 대상 공간 부족(복사형 수집기).
- **Humongous Allocation 실패(G1)**: 연속 Region 필요 → Mixed/Full로 이어질 수 있음.
- **Concurrent Mode Failure(CMS/G1 일부 단계)**: 동시 단계 지연으로 STW 강제.

---

## 튜닝 기본기(Collector 공통)

### 힙 크기/비율

- 고정 힙(프리터치 포함):
  ```
  -Xms8g -Xmx8g -XX:+AlwaysPreTouch
  ```
- Young/Old 비율(G1은 별도):
  ```
  -XX:NewRatio=2        # Old:Young ≈ 2:1
  -XX:SurvivorRatio=8   # Eden:Survivor ≈ 8:1
  -XX:MaxTenuringThreshold=10
  ```

### G1 핵심 옵션 묶음

```
-XX:+UseG1GC
-XX:MaxGCPauseMillis=200
-XX:G1HeapRegionSize=8m
-XX:InitiatingHeapOccupancyPercent=45
-XX:G1NewSizePercent=20
-XX:G1MaxNewSizePercent=60
-XX:G1ReservePercent=10
-XX:+ParallelRefProcEnabled
-XX:+UseStringDeduplication
```

### ZGC/초저지연(예)

```
-XX:+UseZGC
-XX:SoftMaxHeapSize=8g     # ZGC의 '소프트' 상한
```

### 병렬도

```
-XX:ParallelGCThreads=<N>
-XX:ConcGCThreads=<N/4~>
```

> **원칙**: 지연시간 목표가 뚜렷하면 G1/ZGC, 처리량이면 Parallel. **측정→조정**의 반복이 핵심입니다.

---

## GC 로그 — Unified Logging로 읽기

### 켜기(현대 JDK)

```
-Xlog:gc*,safepoint=info:file=gc.log:time,uptime,level,tags
```

### Young/Mixed 예시 해설(축약)

```
[3.456s][info][gc,start] GC(12) Pause Young (Normal) (G1 Evacuation Pause)
[3.456s][info][gc,heap]  GC(12) Eden regions: 40->0, Survivors: 4->4, Old: 120->122
[3.457s][info][gc,phases] GC(12)   Pre Evacuate Collection Set: 0.1ms
[3.462s][info][gc]       GC(12) Pause Young ... 6.2ms
```
- **Eden 40→0**: 복사 완료, **Survivor 유지**, Old 증가(승격).
- **Pause 6.2ms**: 지연시간 목표와 비교.

### 흔한 신호

- **경고**: “to-space exhausted”, “humongous regions”, “promotion failed” → **Old/Reserve/Region/Young 비율** 재조정 필요.
- **STW 길어짐**: `MaxGCPauseMillis` 상향/하향, Region 크기, Mixed 비율, RSet 비용 점검.

---

## 코드 실습 ①: 메모리 패턴에 따른 GC 관찰

```java
// GCPressure.java
import java.util.*;
public class GCPressure {
    static byte[] block(int kb) { return new byte[kb * 1024]; }

    public static void main(String[] args) throws Exception {
        List<byte[]> bag = new ArrayList<>();
        long start = System.nanoTime();
        for (int i = 0; i < 200_000; i++) {
            // 소객체 다량 → Young GC 빈번
            bag.add(block(4)); // 4KB
            if (i % 1000 == 0) bag.clear(); // 생존률 조정
        }
        long ms = (System.nanoTime() - start)/1_000_000;
        System.out.println("Done in " + ms + " ms, size=" + bag.size());
    }
}
```

**실행 팁**
```
java -Xms2g -Xmx2g -XX:+UseG1GC -XX:MaxGCPauseMillis=200 \
     -Xlog:gc*:file=gc.log:time,uptime GCPressure
```
- `bag.clear()` 위치/빈도를 바꿔 **생존률**을 조정하면 **승격/Old 압박**이 어떻게 변하는지 바로 관찰 가능합니다.

---

## 코드 실습 ②: GC/메모리 JMX 지표 읽기

```java
// GCStats.java
import java.lang.management.*;
import java.util.*;
public class GCStats {
    public static void main(String[] args) throws Exception {
        for (GarbageCollectorMXBean gc : ManagementFactory.getGarbageCollectorMXBeans()) {
            System.out.printf("GC=%s, count=%d, time(ms)=%d%n",
                    gc.getName(), gc.getCollectionCount(), gc.getCollectionTime());
        }
        for (MemoryPoolMXBean p : ManagementFactory.getMemoryPoolMXBeans()) {
            System.out.printf("Pool=%s, type=%s, used=%d, committed=%d, max=%d%n",
                    p.getName(), p.getType(), p.getUsage().getUsed(),
                    p.getUsage().getCommitted(), p.getUsage().getMax());
        }
    }
}
```

> 이 지표들을 **프로메테우스/JFR** 등과 연계하면 **GC 대시보드**를 구성할 수 있습니다.

---

## Reference 타입(Soft/Weak/Phantom)과 Finalization

| 타입 | 회수 정책 | 용도 | 주의 |
|---|---|---|---|
| **SoftReference** | 메모리 압박 시 회수 지연 | 캐시 | 비결정적, OOM 방어 아님 |
| **WeakReference** | 다음 GC에서 회수 | 맵 키/메타데이터 | 강한 참조와 혼용 주의 |
| **PhantomReference** | 이미 회수 **이후** 큐 통지 | 리소스 정리(네이티브) | **객체 내용 접근 불가** |
| **Finalizer** | **비권장/폐지 수순** | (X) | 불확실/지연/보안 취약 |
| **Cleaner** | 대안(명시) | 네이티브 해제 | 우발적 지연 여전 → 가급적 **명시 닫기** 권장 |

**Phantom + ReferenceQueue** 스케치:
```java
// PhantomDemo.java
import java.lang.ref.*;
class NativeRes { /* close()로 반드시 닫는 것이 원칙 */ }
public class PhantomDemo {
    public static void main(String[] args) throws Exception {
        ReferenceQueue<NativeRes> q = new ReferenceQueue<>();
        NativeRes res = new NativeRes();
        PhantomReference<NativeRes> ref = new PhantomReference<>(res, q);
        res = null; // 강한 참조 제거
        System.gc();
        Reference<?> r = q.remove(5000); // 회수 후 통지
        if (r != null) {
            // 여기에서 네이티브 핸들 정리 로직을 수행(별도 핸들 보관 필요)
            System.out.println("Phantom notified");
        }
    }
}
```

---

## 메모리 누수·OOM의 흔한 원인과 대처

- **캐시/맵에 쌓이는 객체(강한 참조)** → 만료/용량 정책·`Weak/SoftReference`/Caffeine 등 사용.
- **ThreadLocal 누수** → 스레드 풀 재사용 시 **반드시 remove()**.
- **ClassLoader 누수** → 핫 리로드/플러그인 시스템.
- **DirectBuffer/Native** → `-XX:MaxDirectMemorySize`·명시 해제·JFR/Native Memory Tracking.
- **Metaspace** → 클래스 언로딩 활성화(기본), 리플렉션/프록시 폭주 점검.

---

## 지연시간/처리량 목표 잡기 (간단 계산)

서비스 지연 SLO를 $$L_{\text{target}}$$, GC로 허용되는 지연 예산을 $$B_{\text{gc}}$$ 라 하면,
$$
B_{\text{gc}} \le L_{\text{target}} - L_{\text{app\_nonGC}}
$$
- 예: p99=150ms, 애플리케이션 비GC 지연 p99=120ms → GC에 허용 가능한 예산은 **≤30ms**.
- 이때 **G1의 `MaxGCPauseMillis`를 20~30ms**로 맞추고, Region/Young 비율을 조정해 관측치가 들어오는지 **실측**으로 확증해야 합니다.

---

## Collector별 튜닝 포인트 요약

### Parallel (처리량)

- 크고 단순한 배치: `-XX:+UseParallelGC -Xms -Xmx` 고정, `ParallelGCThreads`만 잡고 끝.
- Pause 민감하면 부적합.

### G1 (균형형/기본)

- `MaxGCPauseMillis`: 목표 지연.
- `G1NewSizePercent/G1MaxNewSizePercent`: Young 크기.
- `InitiatingHeapOccupancyPercent`: 동시 마킹 시작점.
- `G1ReservePercent`: 승격/복사 여유.
- 문자열 중복 제거: `-XX:+UseStringDeduplication`.

### ZGC/Shenandoah (초저지연)

- 힙 크게, OS/NUMA 고려.
- `SoftMaxHeapSize`(ZGC), `ShenandoahGCHeuristics` 등.
- **JFR**로 실측 후 Young/Old(Generational 지원 시) 비중 점검.

---

## 실전 체크리스트

- [ ] **SLO 정의**: p95/p99 지연 목표, 스루풋 목표.
- [ ] **Collector 선택**: Latency ↔ Throughput 균형.
- [ ] **힙 고정/프리터치**: 컨테이너/NUMA 환경에 맞춤.
- [ ] **로그/프로파일링**: Unified `-Xlog:gc*`, JFR, JMX.
- [ ] **레이턴시 예산 맞추기**: `MaxGCPauseMillis`(G1)·ZGC baseline.
- [ ] **Promotion/To-space 실패 0화**: Reserve/Young 비율 조절.
- [ ] **레퍼런스/클리너 설계**: Finalizer 금지, 명시 해제 우선.
- [ ] **Direct/Native 추적**: NMT/JFR, `MaxDirectMemorySize`.
- [ ] **캐시 전략**: 만료/최대용량/히트율 측정.
- [ ] **릴리즈 전 부하테스트**: 데이터 실제 분포/생존률 재현.

---

## GC 옵션 치트시트(발췌)

```text
# 공통

-Xms<size> -Xmx<size>                 # 힙 고정 권장(서버)
-XX:+AlwaysPreTouch                   # 힙 미리 터치
-Xlog:gc*,safepoint:file=gc.log:time,uptime,level,tags

# Parallel

-XX:+UseParallelGC
-XX:ParallelGCThreads=<n>

# G1

-XX:+UseG1GC
-XX:MaxGCPauseMillis=200
-XX:G1HeapRegionSize=8m
-XX:G1NewSizePercent=20 -XX:G1MaxNewSizePercent=60
-XX:G1ReservePercent=10
-XX:InitiatingHeapOccupancyPercent=45
-XX:+UseStringDeduplication

# ZGC

-XX:+UseZGC
-XX:SoftMaxHeapSize=8g
# (필요 시) Generational 모드/관련 옵션은 JDK 버전에 맞춰 확인

# Shenandoah

-XX:+UseShenandoahGC
-XX:ShenandoahGCHeuristics=balanced
```

---

## 자주 묻는 질문(FAQ)

**Q. `System.gc()`를 호출해야 하나요?**
A. 일반적으로 **아니오**. 테스트/디버그용 외 권장하지 않습니다. `-XX:+DisableExplicitGC` 고려.

**Q. Full GC가 왜 터졌나요?**
A. Old 부족/Promotion 실패/Humongous 실패/동시 단계 지연 등. **로그로 원인 태그** 확인 → Young/Old/Reserve/Region 재조정.

**Q. 작은 객체 수백만 개 vs 큰 배열 몇 개?**
A. 작은 객체 다량은 **메타/카드/RSet 비용↑**. 종종 **배열·구조체화**가 유리.

**Q. Escape Analysis가 GC에 주는 영향?**
A. **스칼라 대체/스택 할당**으로 힙 할당↓ → GC 압력↓. (JIT 최적화에 의존)

---

## 부록: G1 마이그레이션 가이드(Parallel→G1)

1) 동일 힙 크기로 시작: `-Xms/-Xmx` 고정
2) `-XX:+UseG1GC -XX:MaxGCPauseMillis=200`
3) 로그 수집 → **Young 비율/생존률/승격 실패** 점검
4) `G1NewSizePercent/G1MaxNewSizePercent` 조절로 Young 크기 튜닝
5) `InitiatingHeapOccupancyPercent`로 마킹 타이밍 조절
6) 진동(oscillation) 시 Region 크기/Reserve 재평가

---

## 부록: 힙/지연 예산 간단 수식

$$
\text{Pause} \approx \frac{\text{Live Bytes to Evacuate}}{\text{Copy Throughput}} + \text{Overheads}
$$

- 라이브 바이트(생존률 ↑)가 크면 **복사형 수집기**의 Pause↑.
- 해결: **Young 축소**(생존률 낮추기) 또는 **Old 대상 Mixed 조절**, **스레드 수** 증가로 Throughput↑.

---

## 마무리

- GC는 **원리 이해 + 측정 기반 튜닝**이 전부입니다.
- **지표/JFR/로그**로 실제 워크로드의 **생존률/할당률/승격/장애**를 숫자로 파악하고,
  Collector 특성(G1/ZGC/Parallel)에 맞게 **목표 지연/처리량**을 만족하도록 **작게 한 번씩** 조정하세요.

---

## 부록 코드: 스트링 중복 제거 효과(체감 예)

```java
// StringDedupDemo.java (G1에서만 효과)
// 실행: java -Xms2g -Xmx2g -XX:+UseG1GC -XX:+UseStringDeduplication -Xlog:gc*:file=gc.log StringDedupDemo
import java.util.*;
public class StringDedupDemo {
    public static void main(String[] args) {
        List<String> list = new ArrayList<>();
        for (int i=0;i<2_000_000;i++) {
            // 동일 내용(동적 생성) → Dedup 대상
            list.add(new String("product-") + (i % 10_000));
        }
        System.out.println("size=" + list.size());
        // gc.log에서 "String Deduplication" 섹션으로 효과 확인
    }
}
```

> `-Xlog:gc+stringdedup=debug` 로 중복 제거 통계까지 확인 가능합니다.
