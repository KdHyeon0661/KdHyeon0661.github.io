---
layout: post
title: 디자인패턴 - Template Method
date: 2025-06-30 19:20:23 +0900
category: 디자인패턴
---
# Template Method (템플릿 메서드 패턴)

## 1. 핵심 아이디어와 용어

- 템플릿 메서드(Template Method): 알고리즘의 전체 흐름과 각 단계의 호출 순서를 **하나의 메서드로 고정**한다.
- 기본 단계(Primitive Operation): 하위 클래스가 재정의해야 하는 단계(추상 또는 기본 제공 후 재정의 가능).
- 훅(Hook): 선택적 재정의 지점. 기본 구현을 제공하거나 불린 값을 반환해 흐름을 제어한다.
- 불변부 vs 가변부: 상위 클래스는 **불변부(순서, 전후관계, 공통 사전/사후 처리)**를 담당하고, 가변부(구체 구현, 정책)는 하위 클래스에 위임한다.
- 헐리우드 원칙(Hollywood Principle): "우리에게 연락하지 마세요. 우리가 연락하겠습니다." 상위 템플릿이 흐름을 제어하고 하위는 콜백처럼 참여한다.

---

## 2. 언제 쓰나 — 동기와 코드 스멜

다음과 같은 상황에서 템플릿 메서드가 유리하다.

- 여러 클래스가 **동일한 처리 순서**를 따르되 단계별 구현만 다를 때(파서, 렌더러, 데이터 처리 파이프라인).
- 중복된 절차 코드가 **여러 하위 클래스에 반복**될 때.
- 절차의 전후처리(로그·트랜잭션·리소스 해제)를 중앙에서 **일관되게** 보장해야 할 때.
- 상속 기반 프레임워크에서 사용자가 **오버라이드 포인트**만 제공받을 때.

반대로, 알고리즘의 **순서 자체가 자주 바뀌거나** 상황에 따라 전혀 다른 경로를 선택한다면 Strategy/Composite/Chain of Responsibility가 더 맞다.

---

## 3. 구조와 역할 (UML/ASCII)

```
┌──────────────────────────┐
│ AbstractClass            │
│  ─ templateMethod()      │  ← 알고리즘 골격(순서 고정, final 권장)
│  ─ step1()               │  ← 기본/추상
│  ─ step2()               │  ← 기본/추상
│  ─ hook()?               │  ← 선택적 재정의 지점
└───────────┬──────────────┘
            │
   ┌────────▼─────────┐      ┌────────▼─────────┐
   │ ConcreteClassA    │      │ ConcreteClassB    │
   │  step1/2 override │      │  step1/2 override │
   └───────────────────┘      └───────────────────┘
```

- AbstractClass: 템플릿 메서드를 제공(흐름 고정). 단계 메서드는 추상 또는 기본 구현.
- ConcreteClass: 단계 메서드를 재정의하여 변 variation 을 구현.

---

## 4. 수학 메모 — 중복 제거 효과의 직관

유사 알고리즘을 가진 변형이 n개 있고, 알고리즘 절차 길이가 m이며 공통부가 k(0<k≤m)일 때, 공통부를 템플릿으로 올리지 않으면 중복 LOC는 대략
$$
\text{dup} \approx (n-1)\cdot k
$$
으로 늘어난다. 템플릿으로 공통부 k를 1회만 정의하면 중복이 0에 수렴한다. 또한 사이클로매틱 복잡도는 템플릿에서 제어 흐름을 집약함으로써 **변형별 클래스의 분기 수**가 줄어드는 경향이 있다.

---

## 5. 구현 가이드 (Python)

### 5.1 기본 형태 — 데이터 임포터 예시

```python
from abc import ABC, abstractmethod

class DataImporter(ABC):
    # 템플릿 메서드: 순서 고정
    def run(self, source: str) -> None:
        self.before_all(source)
        raw = self.read(source)
        items = self.parse(raw)
        valid = [x for x in items if self.validate(x)]
        self.persist(valid)
        self.after_all(len(valid))

    # 공통/훅
    def before_all(self, source: str) -> None:
        pass

    def after_all(self, count: int) -> None:
        pass

    # 변형 포인트
    @abstractmethod
    def read(self, source: str) -> str: ...

    @abstractmethod
    def parse(self, raw: str) -> list: ...

    def validate(self, item) -> bool:
        return True

    @abstractmethod
    def persist(self, items: list) -> None: ...

class CSVImporter(DataImporter):
    def read(self, source: str) -> str:
        with open(source, "r", encoding="utf-8") as f:
            return f.read()

    def parse(self, raw: str) -> list:
        return [line.split(",") for line in raw.splitlines() if line.strip()]

    def validate(self, row) -> bool:
        return len(row) >= 2

    def persist(self, items: list) -> None:
        for row in items:
            pass  # DB 저장 로직

class JSONImporter(DataImporter):
    import json
    def read(self, source: str) -> str:
        with open(source, "r", encoding="utf-8") as f:
            return f.read()
    def parse(self, raw: str) -> list:
        return self.json.loads(raw)
    def persist(self, items: list) -> None:
        pass  # 다른 저장 전략
```

### 5.2 훅으로 조건적 단계 수행

```python
from abc import ABC, abstractmethod

class CaffeineBeverage(ABC):
    def prepare_recipe(self):
        self.boil_water()
        self.brew()
        self.pour_in_cup()
        if self.want_condiments():
            self.add_condiments()

    def boil_water(self): print("Boiling water")
    def pour_in_cup(self): print("Pouring into cup")

    @abstractmethod
    def brew(self): ...
    @abstractmethod
    def add_condiments(self): ...

    # 훅: 기본 True/False로 흐름 제어
    def want_condiments(self) -> bool:
        return True

class Americano(CaffeineBeverage):
    def brew(self): print("Dripping coffee")
    def add_condiments(self): print("No condiments")
    def want_condiments(self) -> bool:
        return False

class Latte(CaffeineBeverage):
    def brew(self): print("Pulling espresso shot")
    def add_condiments(self): print("Adding steamed milk")
```

### 5.3 테스트 스케치(PyTest)

```python
def test_template_calls_order(monkeypatch, capsys):
    order = []
    class T(CaffeineBeverage):
        def brew(self): order.append("brew")
        def add_condiments(self): order.append("cond")
        def boil_water(self): order.append("boil")
        def pour_in_cup(self): order.append("pour")
    T().prepare_recipe()
    assert order == ["boil", "brew", "pour", "cond"]
```

---

## 6. Java 구현 — 템플릿 final, 단계 protected

```java
abstract class FileProcessor {
    public final void process(String path) {
        beforeAll(path);
        String raw = read(path);
        var model = transform(raw);
        if (shouldValidate()) validate(model);
        persist(model);
        afterAll();
    }

    protected void beforeAll(String path) {}
    protected void afterAll() {}
    protected boolean shouldValidate() { return true; }

    protected abstract String read(String path);
    protected abstract Object transform(String raw);
    protected void validate(Object model) { /* default validation */ }
    protected abstract void persist(Object model);
}

class XmlProcessor extends FileProcessor {
    @Override protected String read(String path) { return "..."; }
    @Override protected Object transform(String raw) { return new Object(); }
    @Override protected void persist(Object model) { /* save */ }
}

class CsvProcessor extends FileProcessor {
    @Override protected String read(String path) { return "..."; }
    @Override protected Object transform(String raw) { return new Object(); }
    @Override protected boolean shouldValidate() { return false; } // 훅
    @Override protected void persist(Object model) { /* save */ }
}
```

권장: 템플릿 메서드 `final`로 보호(순서 변경 금지), 단계 메서드는 `protected`로 노출 범위 제한.

---

## 7. C# 구현 — sealed 템플릿, async 변형

```csharp
public abstract class Pipeline<TIn, TOut>
{
    public async Task<TOut> RunAsync(TIn input, CancellationToken ct = default)
    {
        await BeforeAllAsync(input, ct);
        var prepared = await PrepareAsync(input, ct);
        var processed = await ProcessAsync(prepared, ct);
        if (ShouldValidate()) await ValidateAsync(processed, ct);
        await PersistAsync(processed, ct);
        await AfterAllAsync(processed, ct);
        return processed;
    }

    protected virtual Task BeforeAllAsync(TIn input, CancellationToken ct) => Task.CompletedTask;
    protected virtual Task AfterAllAsync(TOut output, CancellationToken ct) => Task.CompletedTask;
    protected virtual bool ShouldValidate() => true;

    protected abstract Task<object> PrepareAsync(TIn input, CancellationToken ct);
    protected abstract Task<TOut> ProcessAsync(object prepared, CancellationToken ct);
    protected virtual Task ValidateAsync(TOut data, CancellationToken ct) => Task.CompletedTask;
    protected abstract Task PersistAsync(TOut data, CancellationToken ct);
}

public sealed class ImageIngestPipeline : Pipeline<string, byte[]>
{
    protected override async Task<object> PrepareAsync(string path, CancellationToken ct)
    {
        return await File.ReadAllBytesAsync(path, ct);
    }
    protected override Task<byte[]> ProcessAsync(object prepared, CancellationToken ct)
    {
        var bytes = (byte[])prepared;
        return Task.FromResult(bytes); // 변환/전처리 자리
    }
    protected override Task PersistAsync(byte[] data, CancellationToken ct)
    {
        return Task.CompletedTask; // 저장
    }
}
```

주의: 템플릿이 스레드 안전해야 한다면, 상태를 공유하지 말고 지역 변수로 유지하라.

---

## 8. TypeScript 구현 — 템플릿과 훅

```ts
abstract class Renderer<T> {
  public render(data: T): string {
    this.before(data);
    const head = this.header(data);
    const body = this.body(data);
    const tail = this.footer(data);
    this.after(data);
    return [head, body, tail].join("\n");
  }

  protected before(data: T): void {}
  protected after(data: T): void {}

  protected abstract header(data: T): string;
  protected abstract body(data: T): string;
  protected abstract footer(data: T): string;
}

class MarkdownRenderer extends Renderer<{title:string, items:string[]}> {
  protected header(d){ return `# ${d.title}`; }
  protected body(d){ return d.items.map(x=>`- ${x}`).join("\n"); }
  protected footer(_){ return ""; }
}
```

---

## 9. 실전 시나리오

1) ETL/배치
- 템플릿: 로드 → 파싱 → 검증 → 변환 → 저장 → 보고
- 변형: 파일 형식별 리더, 대상 스키마별 변환기

2) 빌드/배포 파이프라인
- 템플릿: 체크아웃 → 의존성 설치 → 테스트 → 빌드 → 패키징 → 배포
- 변형: 언어별 빌드·테스트 단계 구현

3) UI 렌더링
- 템플릿: 레이아웃 시작 → 섹션 렌더 → 프래그먼트 → 레이아웃 종료
- 변형: 테마, 지역화, 접근성 옵션

4) 게임 턴 처리
- 템플릿: 시작 효과 → 입력 처리 → AI 업데이트 → 충돌 판정 → 스코어 업데이트 → 렌더
- 변형: 캐릭터/스테이지별 규칙만 오버라이드

5) 파서/해석기
- 템플릿: 토큰화 → 구문 분석 → 의미 분석 → 코드 생성/실행
- 변형: 언어 방언별 규칙, 백엔드별 코드 생성

---

## 10. Template Method와의 조합

- Template Method + Factory Method: 템플릿의 특정 단계에서 제품 생성을 **팩토리 메서드**로 위임하여 서브타입을 분리.
- Template Method + Hook + Strategy: 알고리즘의 큰 틀은 템플릿, 일부 정책은 **전략 객체**로 주입하여 상속의 단점을 보완.
- Template Method + Template Callback: 프레임워크가 템플릿을 제공하고 사용자가 콜백을 전달(콜백이 곧 단계).

---

## 11. 유사/대조 패턴 비교

| 항목 | Template Method | Strategy | Chain of Responsibility | Builder |
|---|---|---|---|---|
| 목적 | 순서 고정, 단계 재정의 | 알고리즘 교체 | 단계적 처리의 전달 | 단계적 조립으로 생성 |
| 기술 | 상속(오버라이드) | 조합(위임) | 연결된 처리자 | 조합/지시자 |
| 교체 시점 | 컴파일(정적) | 런타임(동적) | 런타임(흐름 선택) | 빌드 시점 |
| 적합 | 절차 동일, 세부만 다름 | 절차 자체가 다양 | 누가 처리할지 모를 때 | 복잡한 객체 생성 |

---

## 12. 안티패턴과 주의점

- 취약한 기반 클래스(Fragile Base Class): 상위 수정이 모든 하위에 파급. → 공통부를 작게 유지하고 테스트로 보호.
- 오버라이드 폭발: 단계가 너무 많아 서브클래스가 과도해짐. → 훅 최소화, 공통 단계는 합치기, Strategy 주입 병행.
- 순서 역전: 서브클래스가 템플릿 순서를 바꾸려 함. → 템플릿 메서드는 final/sealed로 고정.
- 상태 공유로 인한 스레드 안전성 문제. → 템플릿 내 상태는 지역화, 공유필드 금지, 불변 객체 활용.
- 잘못된 적용: 절차가 자주 바뀌는데 상속으로 고정. → Strategy/파이프라인으로 전환 고려.

---

## 13. 품질 체크리스트

- [ ] 템플릿 메서드가 **알고리즘 순서를 명확히** 고정하고 있는가(한 눈에 읽힌다)?
- [ ] 각 단계의 책임이 **단일**하고 이름이 행위를 잘 설명하는가?
- [ ] 훅의 수가 **최소**이며 기본값이 안전한가?
- [ ] 템플릿 자체가 **final/sealed** 또는 외부에서 순서를 못 바꾸게 보호되는가?
- [ ] 공통 전후 처리(로그, 계측, 예외 변환, 리소스 해제)가 템플릿에 일관되게 있다?
- [ ] 서브클래스는 **정말 필요한 단계만** 오버라이드하는가?
- [ ] 경쟁 상태, 재진입 문제 없이 **스레드 안전**한가?

---

## 14. 리팩터링 절차(중복 절차 → 템플릿)

1) 유사 절차를 가진 클래스 N개에서 **공통 순서**를 식별한다.
2) 상위 클래스를 만들고 템플릿 메서드를 도입한다(공통 순서만 포함).
3) 각 변형 지점을 **추상/가상 메서드**로 승격한다.
4) 하위 클래스에서 해당 단계만 오버라이드한다.
5) 템플릿 메서드를 final/sealed로 고정한다.
6) 공통 전후처리를 템플릿으로 끌어올린다(리소스, 트랜잭션, 로깅).
7) 테스트로 순서 및 호출 보장을 검증한다.

---

## 15. 예외 처리·트랜잭션·리소스 관리

- 템플릿에서 try/finally로 전역 리소스 해제 보장(파일, 커넥션).
- 단계별 예외를 템플릿에서 공통 포맷으로 변환(도메인 예외).
- 트랜잭션 경계를 템플릿 시작/종료에 배치, 단계는 순수 함수에 가깝게.

Python 스케치:

```python
def run(self, src: str):
    self.before_all(src)
    try:
        raw = self.read(src)
        data = self.parse(raw)
        if self.should_validate():
            for x in data: self.validate(x)
        self.persist(data)
    except Exception as e:
        self.on_error(e)
        raise
    finally:
        self.after_all(len(data) if 'data' in locals() else 0)
```

---

## 16. 요약

- 템플릿 메서드는 **절차의 순서를 상위에서 고정**하고, **세부 구현을 하위로 위임**해 중복을 제거하고 일관성을 높인다.
- 훅과 기본 구현으로 **유연성**을 확보하되, 순서 변경은 금지하여 **예측 가능성**을 유지한다.
- Strategy 등 조합 기반 패턴과 함께 사용하면 상속의 경직성을 보완할 수 있다.
- 적용 전, 절차의 **공통 순서가 안정적**인지 반드시 판단하라. 잦은 절차 변경에는 부적합하다.
