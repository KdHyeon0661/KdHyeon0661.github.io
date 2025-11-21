---
layout: post
title: C# - PInvoke, Marshal, NativeMemory
date: 2024-11-01 19:20:23 +0900
category: Csharp
---
# C# Span<T>와 Memory<T> 심화: 고성능 메모리 조작 기법

## 준비: 왜 이 도구들이 성능을 바꾸는가?

GC 관점의 비용 모델을 간단히 수식화하면:

\[
\text{Total Cost} \approx \underbrace{\sum \text{AllocCost}}_{\text{힙할당/승격}} + \underbrace{\sum \text{CopyCost}}_{\text{불필요 복사}} + \underbrace{\sum \text{PinCost}}_{\text{핀 고정 비용}} + \text{Branch/BoundsCheck}
\]

목표는 다음을 **최소화**:
- **힙 할당 수/크기** ↓ (`stackalloc`, 풀 재사용, 슬라이스)
- **복사** ↓ (슬라이스/Span로 뷰만 전달)
- **핀 고정** ↓ (`Span<T>`/`Memory<T>` 조합, Pool/Owner 패턴)
- **경계 검사**는 JIT 최적화 + Span의 안전성으로 **암묵적 최소화**

---

## Span<T> — 스택 전용 고성능 슬라이스

### 기본 사용

```csharp
Span<int> numbers = stackalloc int[5];
for (int i = 0; i < numbers.Length; i++) numbers[i] = i;
// 0 1 2 3 4
Span<int> slice = numbers.Slice(2, 2); // 2, 3
```

- `Span<T>`는 **ref struct** → **힙 저장 불가**, **박싱 불가**, **비동기 상태 머신/람다 캡처 불가**
- 슬라이스는 **복사 없이** 동일 메모리 범위에 대한 뷰를 만든다.

### 배열/포인터/네이티브 버퍼에서 Span 얻기

```csharp
byte[] arr = new byte[10];
Span<byte> s1 = arr;                 // 배열 → Span
unsafe
{
    byte* p = stackalloc byte[10];
    Span<byte> s2 = new Span<byte>(p, 10); // 포인터 → Span
}
```

> 배열→Span 변환은 **0-비용**. 포인터로부터 생성은 `unsafe` 문맥이 필요.

### 인덱서/범위(Range)·Index

```csharp
ReadOnlySpan<char> s = "Hello, World".AsSpan();
ReadOnlySpan<char> world = s[7..];       // "World"
ReadOnlySpan<char> hello = s[..5];       // "Hello"
ReadOnlySpan<char> inner = s[7..12];     // "World"
```

---

## ReadOnlySpan<T> — 읽기 전용, API 안정성↑

- 읽기 전용 뷰로 API를 정의하면 **방어적 복사 없이** 안전한 불변 계약을 제공.
- 문자열 파싱·토큰화에 최적.

```csharp
static int CountComma(ReadOnlySpan<char> text)
{
    int cnt = 0;
    foreach (var ch in text) if (ch == ',') cnt++;
    return cnt;
}

int c = CountComma("a,b,c".AsSpan()); // 2
```

---

## Memory<T> / ReadOnlyMemory<T> — 비동기·필드 저장 가능

- `Memory<T>`는 **힙에 저장 가능**한 슬라이스로, `Span<T>`의 **장기 저장/비동기 전달** 대안
- 필요 시 `.Span`으로 순간 뷰 획득

```csharp
class BufferHolder
{
    private Memory<byte> _mem;
    public BufferHolder(int size) => _mem = new byte[size];
    public async Task<int> FillAsync(Stream s, int n)
    {
        var mem = _mem[..n];                // Slice
        int read = await s.ReadAsync(mem);  // ReadOnlyMemory/Memory friendly
        return read;
    }
}
```

> 비동기/람다에서 버퍼를 오래 들고 있어야 한다면 `Span<T>` 대신 `Memory<T>`로 필수 전환.

---

## stackalloc — 힙을 건너뛰는 초저지연 임시 버퍼

- **작고 일시적인 버퍼**에 최적 (수백~수천 바이트 수준 권장)
- 스코프 종료 시 자동 해제 → **GC 압력 0**

```csharp
Span<byte> tmp = stackalloc byte[64];
// 빠른 포맷팅/인코딩/스칼라 처리
```

> 너무 큰 크기는 **스택 오버플로우** 위험. 반복문에서 크기가 큰 `stackalloc`은 피함.

---

## 고급 문자열/바이너리 처리 with Span

### Substring 없이 토큰화

```csharp
static (ReadOnlySpan<char> head, ReadOnlySpan<char> tail) SplitOnce(ReadOnlySpan<char> s, char sep)
{
    int i = s.IndexOf(sep);
    if (i < 0) return (s, ReadOnlySpan<char>.Empty);
    return (s[..i], s[(i + 1)..]);
}

var input = "apple,banana,kiwi".AsSpan();
var (first, rest) = SplitOnce(input, ',');
// first: "apple", rest: "banana,kiwi" — 복사 없음
```

### UTF-8 <-> UTF-16 변환 (부분 변환/확장 버퍼)

```csharp
using System.Text;

static int Utf16ToUtf8(ReadOnlySpan<char> src, Span<byte> dst, out int usedChars, out int written)
{
    var enc = Encoding.UTF8;
    enc.Convert(src, dst, false, out usedChars, out written, out bool completed);
    return completed ? 0 : -1;
}

Span<byte> buf = stackalloc byte[64];
int rc = Utf16ToUtf8("안녕하세요".AsSpan(), buf, out var used, out var wrote);
```

> 대량 변환은 `Encoding.UTF8.GetBytes(ReadOnlySpan<char>, Span<byte>)` 오버로드 적극 활용.

### BinaryPrimitives로 엔디언 안전 접근

```csharp
using System.Buffers.Binary;

Span<byte> b = stackalloc byte[4];
BinaryPrimitives.WriteUInt32LittleEndian(b, 0x11223344);
uint v = BinaryPrimitives.ReadUInt32LittleEndian(b); // 0x11223344
```

---

## MemoryMarshal/Unsafe — 구조 재해석과 브리지 (주의!)

### 구조체 ↔ 바이트 캐스팅

```csharp
using System.Runtime.InteropServices;

[StructLayout(LayoutKind.Sequential, Pack = 1)]
struct Header { public ushort len; public byte type; }

Span<byte> raw = stackalloc byte[3];
raw[0] = 0x34; raw[1] = 0x12; raw[2] = 0xA5;

ref Header h = ref MemoryMarshal.AsRef<Header>(raw);
ushort len = h.len;  // 0x1234 (LE 환경 가정)
byte type = h.type;  // 0xA5
```

- **blittable**(관리형 필드 없음, 고정 레이아웃)이어야 안전
- 정렬/패킹, 엔디언스 **정확히** 일치해야 함

### Span< T > → ref T

```csharp
ref byte first = ref MemoryMarshal.GetReference(raw);
first = 0xFF;
```

> 고성능 루프에서 **인덱싱 오버헤드**를 줄이는 패턴(최적화 상황 한정)

---

## ArrayPool<T> vs MemoryPool<T> — 장수명 버퍼 재사용

### ArrayPool<T> — 배열 재사용(가장 흔함)

```csharp
using System.Buffers;

var pool = ArrayPool<byte>.Shared;
byte[] rented = pool.Rent(1024);
try
{
    Span<byte> span = rented.AsSpan(0, 1024);
    // 활용
}
finally
{
    pool.Return(rented, clearArray: true); // 민감 데이터면 clear
}
```

- 장점: 간단, **핫 패스에서 GC 압력 크게 감소**
- 주의: **Return 누락 금지**, 크로스-스레드 수명 관리

### MemoryPool<T> — `IMemoryOwner<T>` 수명 명시

```csharp
using System.Buffers;

using IMemoryOwner<byte> owner = MemoryPool<byte>.Shared.Rent(4096);
Memory<byte> mem = owner.Memory[..2048];
// mem.Span 로 작업
// using 블록 종료 시 자동 반환
```

- `using` 스코프로 수명 확실
- **API에 “소유권”을 명시**하는 패턴에 적합

---

## IBufferWriter<T> / SequenceReader<T> — 스트리밍 핵심

### IBufferWriter<T>

```csharp
using System.Buffers;

static void WriteVarInt(IBufferWriter<byte> writer, uint value)
{
    Span<byte> s = writer.GetSpan(5);
    int i = 0;
    while (value >= 0x80)
    {
        s[i++] = (byte)((value & 0x7F) | 0x80);
        value >>= 7;
    }
    s[i++] = (byte)value;
    writer.Advance(i);
}
```

- **출력 버퍼 확보 → 채우기 → Advance()**
- `ArrayBufferWriter<T>`로 간단히 구현 가능

```csharp
var buffer = new ArrayBufferWriter<byte>(256);
WriteVarInt(buffer, 300);
ReadOnlySpan<byte> outSpan = buffer.WrittenSpan;
```

### SequenceReader<T> (ReadOnlySequence<T> 기반)

**네트워크/파일 스트리밍**처럼 **분절된 버퍼**를 연속으로 읽기:

```csharp
using System.Buffers;
using System.IO.Pipelines;

static async Task<int> SumAsync(PipeReader reader)
{
    int sum = 0;
    while (true)
    {
        ReadResult r = await reader.ReadAsync();
        var buffer = r.Buffer;
        var seqReader = new SequenceReader<byte>(buffer);
        while (seqReader.TryRead(out byte b)) sum += b;
        reader.AdvanceTo(seqReader.Position, buffer.End);
        if (r.IsCompleted) break;
    }
    return sum;
}
```

- 복사 없이 **조각난 데이터**를 순차 파싱 가능
- `System.IO.Pipelines`와 조합하면 **고성능 네트워크** 처리

---

## 고급 팁: `string.Create`, `SpanAction`, `TryFormat`

### string.Create — 문자열 생성 시 버퍼 직접 채우기

```csharp
string s = string.Create(8, 0x1234ABCD, (span, state) =>
{
    state.TryFormat(span, out _, provider: null);
});
```

- 중간 할당/복사 없이 바로 결과 문자열 버퍼에 출력

### TryFormat 패턴

```csharp
Span<char> tmp = stackalloc char[64];
if (12345.TryFormat(tmp, out int written))
{
    var result = new string(tmp[..written]);
}
```

- `ToString()` 대신 **사전 할당 버퍼**에 출력 → **GC 감소**

---

## 성능 패턴/안티패턴

### 권장 패턴

- **입출력 경계에만** 복사: 내부는 슬라이스/Span로 전달
- **작은 임시 버퍼**는 `stackalloc` / 큰 버퍼는 Pool
- 비동기 경계/필드 저장은 `Memory<T>`/`ReadOnlyMemory<T>`
- 반복 파싱은 `SequenceReader<T>`/`Pipelines` 고려
- 바이너리 엔디언은 `BinaryPrimitives`로 표준화

### 피해야 할 것

- 큰 `stackalloc`을 루프마다 수행
- `Span<T>`를 **필드에 저장**하려는 시도(ref struct 제약 위반)
- `MemoryMarshal`로 **비-blittable** 구조를 재해석
- 풀에서 빌린 배열을 **Return 누락**/중복 반환
- Span을 `Task`에 캡처(상태 머신 힙화 → UB 위험)

---

## 실전 예제 모음

### CSV 라인 파싱 (복사 0회)

```csharp
static bool TrySplitCsvLine(ReadOnlySpan<char> line, Span<Range> fields, out int fieldCount)
{
    fieldCount = 0;
    int start = 0;
    while (true)
    {
        int idx = line[start..].IndexOf(',');
        if (idx < 0)
        {
            fields[fieldCount++] = start..line.Length;
            return true;
        }
        fields[fieldCount++] = start..(start + idx);
        start += idx + 1;
        if (fieldCount >= fields.Length) return false; // 필드 한도 초과
    }
}

// 사용
Span<Range> ranges = stackalloc Range[8];
var line = "id,name,age".AsSpan();
if (TrySplitCsvLine(line, ranges, out int n))
{
    for (int i = 0; i < n; i++)
        Console.WriteLine(line[ranges[i]].ToString());
}
```

### 수치 포맷팅 (GC 미발생)

```csharp
Span<char> buf = stackalloc char[64];
if (123456789.TryFormat(buf, out int written, provider: null))
{
    ReadOnlySpan<char> view = buf[..written];
    // 로그 출력, 파일 기록 등
}
```

### 바이너리 메시지 읽기 (엔디언 + 구조체)

```csharp
[StructLayout(LayoutKind.Sequential, Pack = 1)]
struct MsgHeader
{
    public ushort LengthLE; // Little Endian
    public byte Type;
}

static bool TryParseHeader(ReadOnlySpan<byte> src, out MsgHeader h)
{
    h = default;
    if (src.Length < 3) return false;
    h.LengthLE = BinaryPrimitives.ReadUInt16LittleEndian(src);
    h.Type     = src[2];
    return true;
}
```

### ArrayPool로 큰 JSON 청크 재사용

```csharp
using System.Buffers;
using System.Text.Json;

var pool = ArrayPool<byte>.Shared;
byte[] rented = pool.Rent(1 << 20); // 1MB
try
{
    // 네트워크 수신 → rented 채움
    var mem = new ReadOnlyMemory<byte>(rented, 0, /*len*/ 500_000);
    var doc = JsonDocument.Parse(mem); // 복사 없이 Parse
    // JSON 탐색
}
finally { pool.Return(rented, clearArray: true); }
```

### MemoryPool + IBufferWriter로 가변 출력

```csharp
using System.Buffers;

var writer = new ArrayBufferWriter<byte>(256);
WriteVarInt(writer, 12345); // §8 예제 재사용
WriteVarInt(writer, 300);
ReadOnlySpan<byte> written = writer.WrittenSpan;
// 소켓/파일로 일괄 전송
```

---

## 체크리스트 (현장 적용 가이드)

- [ ] **핫 패스**에서 Substring/ToArray 남발 여부 점검 → Span/슬라이스로 전환
- [ ] 대형 임시 버퍼 → ArrayPool/MemoryPool로 전환 + 수명 규약 명시
- [ ] 비동기 경계에 Span 전달 금지 → Memory<T>
- [ ] 구조체 재해석/엔디언은 BinaryPrimitives/MemoryMarshal 조합
- [ ] 로깅/포맷팅은 TryFormat/string.Create로 힙 할당 최소화
- [ ] 파이프라인/소켓 계층은 IBufferWriter + SequenceReader 기반 재설계 고려

---

## 주요 API 한눈 요약

| 범주 | API | 요약 |
|---|---|---|
| 슬라이스 | `Span<T>`, `ReadOnlySpan<T>` | 스택/포인터/배열 뷰 |
| 힙 슬라이스 | `Memory<T>`, `ReadOnlyMemory<T>` | 필드·비동기에 안전 |
| 스택 버퍼 | `stackalloc` | 임시 고속 버퍼 |
| 캐스팅 | `MemoryMarshal.Cast<TFrom, TTo>` | 바이트↔구조체 재해석 |
| 참조 | `MemoryMarshal.GetReference` | Span의 첫 요소 ref |
| 엔디언 | `BinaryPrimitives` | 읽기/쓰기 통합 |
| 풀 | `ArrayPool<T>` / `MemoryPool<T>` | 버퍼 재사용 |
| 스트림 | `IBufferWriter<T>`, `ArrayBufferWriter<T>` | 출력 스트리밍 |
| 입력 | `ReadOnlySequence<T>`, `SequenceReader<T>` | 분절 입력 파싱 |
| 문자열 | `string.Create`, `TryFormat` | 무할당 포맷팅 |

---

## 안전성 주의

- `Span<T>`는 **ref struct** → **필드/박싱/비동기** 금지
- `MemoryMarshal`/`Unsafe`는 **전제(레이아웃/정렬/엔디언)**가 틀리면 **UB**
- `stackalloc`은 **크기 제한**을 지키고, 루프 내부 대형 할당 금지
- Pool은 **Return 보장** 및 외부로 **장기 노출 금지**

---

## 결론

- 문자열/바이너리/네트워크/파일 등 **모든 I/O 경계**에서 `Span<T>`/`Memory<T>`를 잘 활용하면 **할당/복사/핀 고정**을 대폭 줄일 수 있다.
- 스트리밍은 `IBufferWriter` + `SequenceReader`/`Pipelines`가 정석.
- 고급 변환은 `MemoryMarshal`/`BinaryPrimitives`로 **정확·안전**하게.

이 글의 예제들을 바로 적용해, **GC 압력과 지연시간을 눈에 띄게 낮추는** 코드를 만들어 보자.
