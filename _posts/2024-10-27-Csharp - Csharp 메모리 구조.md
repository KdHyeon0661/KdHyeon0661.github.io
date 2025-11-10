---
layout: post
title: C# - 열거형, 구조체, 튜플
date: 2024-10-27 19:20:23 +0900
category: Csharp
---
# C# 메모리 구조


## 0) 한 장 요약

- **값 타입**(`struct`, 기본형)은 “값 자체”가 복사된다. **작고 불변/단수 소유**에 적합.
- **참조 타입**(`class`, `string`, 배열)은 힙에 객체, 스택/필드엔 **참조(포인터)**. 복사는 **주소 복사**.
- **스택**: 함수 호출 프레임/지역 값 타입 저장. **매우 빠름**, 범위 벗어나면 자동 해제.
- **힙**: 객체 저장. **GC**가 세대별(Gen0/1/2)로 회수. **LOH(대형 객체 힙)**, **POH(고정 객체 힙)** 존재.
- **박싱/언박싱**은 힙 할당/복사를 유발 → **핫패스에서 회피**.
- **Dispose/Finalizer/SafeHandle** 구분: **메모리 말고 OS 자원은 반드시 명시 해제**.
- **누수 패턴**: 이벤트 핸들러 미해제, 정적 컬렉션 보관, Timer/Task 참조, 핀(Pin) 남용 → **메모리/핸들 고갈**.
- **진단**: `dotnet-counters`, `dotnet-gcdump`, `GC.GetAllocatedBytesForCurrentThread()`, `EventCounters`.

---

## 1) 메모리 구역: 스택 vs 힙

| 구역 | 용도 | 해제 방식 | 특성 |
|---|---|---|---|
| **스택(Stack)** | 호출 프레임, 지역 값 타입, 일부 `ref struct` | 스코프 종료 시 자동 | **초고속**, 연속 메모리, 크기 제한 |
| **힙(Heap)** | 참조 타입 인스턴스, 배열, 박싱 객체 | **GC가 회수** | 유연, 상대적으로 느림, 단편화 관리 |

### 1.1 간단 예
```csharp
class Person { public string Name = ""; }

void Demo()
{
    int x = 10;              // 스택 (값)
    var p = new Person();    // 힙 (객체), p는 스택의 참조
    p.Name = "Alice";        // 힙 객체의 필드 접근
} // x는 프레임 종료와 함께 소멸. p는 범위 밖이지만, 힙 객체는 참조가 없을 때 GC 대상
```

---

## 2) 값 타입 vs 참조 타입

| 구분 | 값 타입(Value Type) | 참조 타입(Reference Type) |
|---|---|---|
| 대표 | `int`, `double`, `bool`, `struct`, `enum` | `class`, `string`, `array`, `object` |
| 저장 | **값 자체** 저장(스택/필드 내 inline) | 힙 객체 + **참조** 저장 |
| 복사 | **깊은 복사**(필드 전체) | **얕은 복사**(참조값만 복사) |
| 상속 | 불가(인터페이스 구현 가능) | 가능 |
| GC 대상 | 보통 아님(힙에 박싱되면 대상) | 맞음 |

### 2.1 복사 의미
```csharp
struct Money { public int Won; }

var m1 = new Money { Won = 1000 };
var m2 = m1;            // 값 복사
m2.Won = 2000;
Console.WriteLine(m1.Won); // 1000

class Box { public int V; }
var b1 = new Box { V = 1000 };
var b2 = b1;            // 참조 복사
b2.V = 2000;
Console.WriteLine(b1.V); // 2000
```

> 값 타입은 “작고 자주 생성/폐기되는 불변 데이터”에 특히 좋다. 반대로 **큰 struct**는 **복사 비용**이 커질 수 있으니 주의.  
> 필요 시 `in` 매개변수, `readonly struct`, `ref struct`를 고려.

---

## 3) 박싱(Boxing) & 언박싱(Unboxing)

- **박싱**: 값 타입을 **`object`** 또는 인터페이스로 취급하기 위해 **힙에 새 객체로 감싸는 것**.
- **언박싱**: 박싱된 객체를 **원래 값 타입으로 꺼내는 것**(캐스트 필요).

```csharp
int n = 42;
object boxed = n;        // 박싱 (힙 할당)
int n2 = (int)boxed;     // 언박싱 (값 복사)
```

### 3.1 박싱 비용 사례
```csharp
// ❌ 박싱 발생: object 리스트에 값 타입을 넣음
var list = new List<object>();
for (int i = 0; i < 1000; i++)
    list.Add(i); // 1000번 박싱

// ✅ 제네릭 사용
var ints = new List<int>();
for (int i = 0; i < 1000; i++)
    ints.Add(i); // 박싱 없음
```

> **핫 루프·대량 데이터**에서 박싱은 GC 압력을 크게 만든다. **제네릭** 또는 `Span<T>`를 활용하자.

---

## 4) .NET GC 구조 이해하기

GC는 **세대(Generations)**, **세그먼트(Segments)** 개념으로 성능을 최적화한다.

### 4.1 세대별 수집

| Generation | 대상 | 특징 |
|---|---|---|
| **Gen 0** | 새로 생성된 객체 | **자주** 수집. 살아남으면 Gen1 승격 |
| **Gen 1** | 중간 수명 | 완충 역할 |
| **Gen 2** | 장수 객체 | **드물게** 수집. 큰 비용 |
| **LOH** (Large Object Heap) | **대형 객체(≈ 85KB+)** | 별도 영역. 수집 비용 큼 |
| **POH** (Pinned Object Heap) | **Pinned** 객체 | .NET 5+ 고정된 객체 별도 관리 |

> 대략 **85,000 바이트 이상의 배열/문자열**은 LOH로 간다. LOH는 **압축(compaction)이 제한**되어 **단편화** 이슈가 큼.

### 4.2 세그먼트
- **Ephemeral Segment**: Gen0/Gen1가 위치하는 **가장 뜨거운** 세그먼트.
- **Gen2 Segment**: 장수 객체.
- **LOH/POH Segment**: 각각 별도.

### 4.3 GC 모드
| 모드 | 설명 | 기본 |
|---|---|---|
| Workstation GC | 데스크탑/개발 환경 | 기본 |
| Server GC | 서버/스레드당 힙, 병렬 수집 | ASP.NET/서비스에 적합 |
| Background GC | Gen2 수집을 백그라운드로 수행 | 기본 활성(플랫폼/버전에 따름) |

`runtimeconfig.json`이나 `gcServer`, `gcConcurrent` 설정으로 제어 가능.

---

## 5) GC의 작동 메커니즘(개략)

1. **루트(Root) 탐색**: 스택/레지스터/정적 변수/GCHandle/핀 등 **도달 가능한 참조**를 찾는다.
2. **마킹(Mark)**: 도달 가능한 객체 마킹.
3. **스윕/압축(Sweep/Compact)**: 도달 불가능 객체 회수, 필요 시 **압축**으로 단편화 최소화(LOH는 제한적).
4. **승격(Promotion)**: 생존 객체는 상위 세대로 **승격**(Gen0→Gen1→Gen2).

> 수집 시점은 “메모리 부족/어떤 힌트/휴지 시간” 등 **스케줄링**에 따른다. 즉 **해제가 즉시 일어나지 않는다**.

---

## 6) LOH/POH와 Pinning

### 6.1 LOH (Large Object Heap)
- **큰 배열/문자열**은 LOH에 할당.
- 압축 비용이 커서 **단편화 위험**. 큰 버퍼는 **풀링(ArrayPool<T>)** 고려.

### 6.2 POH (Pinned Object Heap)
- **Pinned** 객체 고유 영역.
- Pinning은 **GC 이동/압축을 방해**. 장시간 Pin은 단편화 가속 → **최소화**가 핵심.

```csharp
// GCHandle로 pinning (짧게!)
var handle = GCHandle.Alloc(buffer, GCHandleType.Pinned);
try
{
    IntPtr addr = handle.AddrOfPinnedObject();
    // 네이티브 호출 등
}
finally { handle.Free(); }
```

---

## 7) Finalizer(소멸자) vs Dispose vs SafeHandle

- **Finalizer**(`~TypeName()`): GC가 객체를 회수할 때 한 번 호출(언제일지 모름). **OS 자원**을 여기서 해제하면 **비결정적/늦음**.
- **Dispose**(`IDisposable`): **명시적** 자원 해제(파일 핸들/소켓/DB 연결 등).
- **SafeHandle**: OS 핸들을 안전하게 래핑. Finalizer보다 **권장**.

### 7.1 표준 Dispose 패턴(.NET 6+)
```csharp
public class MyResource : IDisposable
{
    private bool _disposed;
    private readonly SafeHandle _handle; // 권장

    public MyResource(SafeHandle handle) => _handle = handle;

    // public Dispose()는 관리/비관리 모두 해제
    public void Dispose()
    {
        Dispose(disposing: true);
        GC.SuppressFinalize(this); // 파이널라이저 생략
    }

    protected virtual void Dispose(bool disposing)
    {
        if (_disposed) return;
        if (disposing)
        {
            // 관리 리소스 해제 (IDisposable 호출)
            _handle?.Dispose();
        }
        // 비관리 리소스 해제(필요 시)
        _disposed = true;
    }

    ~MyResource() => Dispose(disposing: false); // 예외 상황 대비
}
```

### 7.2 `using` / using 선언
```csharp
using var fs = File.OpenRead("a.bin"); // 스코프 종료 시 Dispose 자동
// ...
```

> 원칙: **Finalizer에 의존하지 말고** `using`/`Dispose()`로 **즉시 해제**.

---

## 8) 실전: 메모리/GC 관점으로 코드 읽기

### 8.1 값/참조 차이로 인한 의도치 않은 공유
```csharp
class Node { public int V; }
var a = new Node { V = 1 };
var b = a;       // 같은 힙 객체
b.V = 99;
Console.WriteLine(a.V); // 99
```

### 8.2 박싱 폭탄
```csharp
// ToString, IComparable, non-generic 컬렉션 등에서 암묵 박싱 주의
object Acc(object acc, int x) => (int)acc + x; // 매번 박싱/언박싱
```

### 8.3 큰 버퍼 할당/해제 반복 → LOH 단편화
```csharp
// ❌ 매 요청마다 1MB 배열 생성
byte[] buf = new byte[1_000_000]; // LOH
// ✅ ArrayPool 사용
var pool = ArrayPool<byte>.Shared;
byte[] rented = pool.Rent(1_000_000);
try { /* 사용 */ }
finally { pool.Return(rented, clearArray: true); }
```

---

## 9) 자주 묻는 질문(FAQ)

### Q1. **값 타입은 항상 스택에, 참조 타입은 항상 힙에?**  
A. **아니다.** 값 타입은 **필드**로 존재하면 **컨테이너의 메모리**에 **inline**으로 저장된다(예: 클래스 필드면 해당 객체의 힙 메모리). 참조 타입은 객체는 힙이지만 **참조 변수** 자체는 스택/필드에 저장된다.

### Q2. **Dispose를 호출하면 GC가 덜 도나요?**  
A. 메모리 관점의 GC 빈도와는 직접적 상관이 없다. 하지만 **OS 자원 누수**를 막아 시스템 전체 안정성에 중요.

### Q3. **GC.Collect()를 직접 호출해도 되나요?**  
A. 특별한 이유(대형 상태 전환 지점/벤치마크 격리 등)가 아니면 **권장하지 않음**. 런타임이 더 잘 안다.

### Q4. **string은 왜 참조 타입인데 불변?**  
A. **불변(Immutable)** 설계로 해싱/공유/스레드 안전/인터닝에 유리. 수정은 **새 인스턴스**를 만든다.

---

## 10) 고급: 스택 vs 힙—어디에 저장되나(도식)

```
[스택 프레임]
  ├─ int x (10)         // 값
  ├─ Person p ──────────────┐
  └─ Span<int> s (ref+len)  │ (스택/힙/비관리 버퍼 위 “뷰”)
                             ▼
                           [힙]
                           Person { Name → "Alice"(힙) }
                           int[] arr
                           byte[] large (LOH)
```

---

## 11) 수식으로 보는 “복사 비용” 직관

값 타입 크기를 \( S \)바이트, 복사 횟수를 \( N \)이라 하면 단순 복사 총량은

\[
\text{Cost}_\text{copy} \approx N \cdot S
\]

- \( S \)가 작은 **소형 struct**(예: 8~16B)는 복사 부담이 미미할 수 있다.  
- \( S \)가 큰 **대형 struct**(수백 B~)는 참조 타입 전환이나 `in` 매개변수·`ref` 반환 검토.

---

## 12) 진단/도구

- **빠른 계측**: `GC.GetAllocatedBytesForCurrentThread()`
- **카운터**: `dotnet-counters monitor System.Runtime`
  - `gen-0-gc-count`, `loh-size`, `time-in-gc` 등
- **덤프/스냅샷**: `dotnet-gcdump`, `dotnet-dump`
- **프로파일러**: Visual Studio Diagnostic Tools, PerfView

```csharp
long before = GC.GetAllocatedBytesForCurrentThread();
// 작업
long after = GC.GetAllocatedBytesForCurrentThread();
Console.WriteLine($"Allocated: {after - before} bytes");
```

---

## 13) 누수/leak 패턴 & 대응

| 패턴 | 원인 | 해결 |
|---|---|---|
| 이벤트 핸들러 미해제 | Publisher가 Subscriber를 참조 | `-=`, `WeakEvent` 패턴 |
| 정적 컬렉션 누적 | Cache/Dictionary에 영구 참조 | 만료 정책, `ConditionalWeakTable` |
| Timer/Task 지속 참조 | 콜백 캡처로 루트 유지 | `Dispose()`, 취소 토큰 |
| Pinning 남용 | GC 압축 방해, 단편화 | pin **짧게**, 복사 후 작업 |
| 파이널라이저 의존 | 해제가 지연/불안정 | `IDisposable`/`SafeHandle` 사용 |

---

## 14) 실전 샘플

### 14.1 박싱 제거 리팩터링
```csharp
// Before: 박싱
int Sum(IEnumerable<object> xs)
{
    int s = 0;
    foreach (var x in xs) s += (int)x; // 언박싱 반복
    return s;
}

// After: 제네릭
int Sum(IEnumerable<int> xs)
{
    int s = 0;
    foreach (var x in xs) s += x;
    return s;
}
```

### 14.2 대형 버퍼 풀링
```csharp
var pool = ArrayPool<byte>.Shared;
byte[] buf = pool.Rent(1024 * 1024); // 1MB
try
{
    // I/O 처리
}
finally
{
    pool.Return(buf, clearArray: true);
}
```

### 14.3 구조체 복사 비용 줄이기
```csharp
readonly struct Big
{
    public readonly long A, B, C, D; // 32B
    public Big(long a, long b, long c, long d) => (A, B, C, D) = (a, b, c, d);
}

long Use(in Big big) // in: 읽기 전용 참조 전달 → 복사 회피
{
    return big.A + big.D;
}
```

---

## 15) 체크리스트 (메모리 친화 설계)

- [ ] 값 타입은 **작고 불변**인가? (크면 참조 타입 고려)
- [ ] **박싱**이 숨어 있지 않은가? (인터페이스/비제네릭 컬렉션 주의)
- [ ] 대형 버퍼는 **풀링**하는가? (LOH 단편화 회피)
- [ ] **Pinning** 범위는 **짧고 명확**한가?
- [ ] **IDisposable** 리소스는 `using`으로 닫는가?
- [ ] 이벤트 구독은 해제되는가? (누수 방지)
- [ ] 성능 문제 시 **카운터/덤프**로 근거를 확보했는가?

---

## 16) 요약

- 스택/힙 구분은 “저장 위치”와 “수명 관리”의 차이: 스택은 **자동**, 힙은 **GC**.  
- 값/참조 타입의 복사 의미를 이해하면 **무의식적 공유/복사 비용**을 피할 수 있다.  
- GC는 **세대/LOH/POH**로 최적화되어 있으나, **박싱/대형 할당/핀**은 여전히 비용 높다.  
- OS 자원은 GC 대상이 아니므로 반드시 **Dispose/SafeHandle**로 관리한다.  
- **진단 도구**로 근거를 수집하고, **제네릭/Span/풀링** 같은 현대 .NET 관용구를 적극 사용하라.

---

## 부록 A) 미니 퀴즈

1) 아래 코드에서 할당(힙) 발생 지점을 모두 고르시오.  
```csharp
int a = 10;
int[] arr = new int[8];
object o = a;
string s = "hi";
```
- 정답: `new int[8]`(배열 힙), `object o = a`(박싱 힙), `"hi"`(문자열 힙; 상수 풀/인터닝 있음)

2) 다음 중 LOH로 갈 가능성이 가장 높은 것은?  
- A) `new int[1000]`  
- B) `new byte[90_000]`  
- C) `new string('a', 1000)`  
- 정답: **B** (약 85KB+ 넘는 대형 객체)

---

## 부록 B) 참고 코드 모음

### B.1 스레드별 할당량 측정
```csharp
long before = GC.GetAllocatedBytesForCurrentThread();
// work
long after = GC.GetAllocatedBytesForCurrentThread();
Console.WriteLine($"{after - before:N0} bytes allocated");
```

### B.2 using 선언과 다중 리소스
```csharp
using var fs = File.OpenRead("in.bin");
using var ms = new MemoryStream();
// ...
```

### B.3 WeakReference로 캐시 힌트
```csharp
var cache = new Dictionary<string, WeakReference<byte[]>>();

void Put(string key, byte[] data) => cache[key] = new WeakReference<byte[]>(data);

bool TryGet(string key, out byte[]? data)
{
    data = null;
    if (cache.TryGetValue(key, out var wr) && wr.TryGetTarget(out var target))
    {
        data = target;
        return true;
    }
    return false;
}
```