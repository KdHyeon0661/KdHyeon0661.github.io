---
layout: post
title: 객체지향설계 - DRY, KISS, YAGNI
date: 2025-07-20 18:20:23 +0900
category: 객체지향설계
---
# DRY, KISS, YAGNI

## 0. 왜 DRY·KISS·YAGNI인가?

- **SOLID**가 클래스/모듈 설계의 **원리**라면,  
  **DRY·KISS·YAGNI**는 팀이 **과설계/과최적화/중복**을 피하게 돕는 **실천 규범**이다.
- 세 원칙은 **상호 보완**하지만 **긴장**도 있다(예: DRY 추상화 vs KISS 단순성).  
  → **적용 순서/시점**과 **증거 기반(테스트·측정)**이 중요하다.

---

## 1. DRY (Don’t Repeat Yourself)

> “같은 지식을 반복하지 말라.” — *코드 중복이 아니라 ‘지식’ 중복을 겨냥한다.*

### 1.1 중복의 관점

| 유형 | 설명 | 예 |
|---|---|---|
| **문자적 중복** | 동일 코드 조각의 복붙 | 같은 수식/조건식이 여러 모듈에 흩어짐 |
| **지식(의도) 중복** | 동일한 **규칙/정책**이 다른 표현으로 산재 | 할인 규칙을 UI와 백엔드 각각 하드코딩 |
| **구조적 중복** | 동일한 제어/데이터 흐름 패턴 반복 | 비슷한 파이프라인 조립 로직이 여기저기 |
| **데이터 중복** | 상수/스키마/매핑의 중복 정의 | 통화코드 목록을 서비스마다 들고 있음 |

> **핵심**: DRY의 타깃은 *지식(knowledge)*. *AHA(Avoid Hasty Abstraction)* — 성급한 추상화는 금물.

### 1.2 적용 절차(미니 트리)
```
발견된 것 = 중복인가? ── 아니오 → 유지
            │
            └─ 예 → 의미/의도 동일? ── 아니오 → 유지(거짓 양성)
                          │
                          └─ 예 → 변화 방향 동일? ── 아니오 → 일단 중복 허용(성급 추상화 방지)
                                          │
                                          └─ 예 → 추출(함수/모듈/전략/템플릿)
```

### 1.3 리팩토링 레시피

- **Extract Function/Method, Extract Class/Module**  
- **Introduce Parameter / Template Method / Strategy / Policy**  
- **Pull Up Method**(상속), **Composition**(구성)으로 공통화  
- 공통 상수 → **Config/Enum/Value Object**로 이동

#### 예: 문자적 중복 → 함수 추출 (Java)
```java
// Before
double area1 = Math.PI * r1 * r1;
double area2 = Math.PI * r2 * r2;

// After
static double circleArea(double r){ return Math.PI * r * r; }
```

#### 예: 지식 중복 → 정책 객체(Strategy)
```java
interface DiscountPolicy { int apply(int price); }

final class FlatPolicy implements DiscountPolicy {
    public int apply(int price){ return price - 1000; }
}

final class RatePolicy implements DiscountPolicy {
    public int apply(int price){ return (int)(price * 0.9); }
}

final class Checkout {
    private final DiscountPolicy policy;
    Checkout(DiscountPolicy p){ this.policy = p; }
    int pay(int price){ return policy.apply(price); }
}
```

### 1.4 계측(간단)
$$
\text{DuplicationRate}=\frac{\text{Duplicated Lines}}{\text{Total Lines}}
$$
- 도구: SonarQube/SonarCloud, PMD CPD, IntelliJ Duplicates.

### 1.5 주의: DRY vs AHA
- **두 번 반복은 우연**, **세 번째에서 추상화**(*Rule of Three*).  
- 과한 DRY는 **KISS** 위반 및 **변경 경로 결합**을 만든다(Shotgun Surgery 역설).

---

## 2. KISS (Keep It Simple, Stupid)

> “쉽게 유지하라.” — 불필요한 복잡성을 제거하면 **이해/변경/테스트** 모두 쉬워진다.

### 2.1 실천 지침

- **SRP**: 메서드/클래스는 **하나의 명확한 이유**로만 변한다.
- **낮은 매개변수 수**: 3개 초과 시 Value Object/옵션 빌더 고려.
- **짧은 함수**: 한 화면 안(20–40줄)·한 레벨의 추상화 유지.
- **선형 흐름**: 깊은 중첩/분기/반복을 **초기 리턴, 가드절**로 평탄화.
- **패턴 남용 금지**: 필요할 때만 도입(“작동하게 → 단순하게 → 빠르게”).

### 2.2 복잡도 직관
- **Cyclomatic Complexity(CC)**는 분기 수에 비례.  
  CC가 높을수록 테스트 케이스가 급증 → KISS 위반 신호.

### 2.3 예: 복잡 조건 → 의도 노출
```java
// Before
if ((age > 18 && isMember) || (age > 21 && hasInvitation)) allowEntry();

// After
if (canEnter(age, isMember, hasInvitation)) allowEntry();

private boolean canEnter(int age, boolean isMember, boolean hasInvitation){
    return (age > 18 && isMember) || (age > 21 && hasInvitation);
}
```

### 2.4 패턴보다 합성
- 상속 계층보다 **합성+역할(인터페이스)** 로 단순화.  
- 예: 데코레이터/체인으로 **조건에 따른 분기**를 **구성**으로 대체.

---

## 3. YAGNI (You Aren’t Gonna Need It)

> “지금 필요 없는 건 만들지 마라.” — 과설계를 막는 **시점 원칙**.

### 3.1 위험한 사전구현
- “언젠가 쓸지도” → 유지비↑, 복잡도↑, 오히려 **재작성** 일어남.
- **Speculative Generality**(악취): 추상클래스/인터페이스가 의미 없이 존재.

### 3.2 실천
- **현재 승인된 요구**만 구현, 미래는 **테스트/계약**으로만 가늠.
- 확장이 불가피한 **경계(포트/어댑터)**만 미리 정의하고 **구현은 보류**.
- 피처 토글/실험은 **고립된 경로**로 넣고 *쉽게 제거 가능*하도록.

### 3.3 예: 불필요한 선행 기능 제거
```java
// Before: 아직 요구 없음
public void exportToXML(){ /* speculative */ }

// After: 삭제. 필요 시점에 구현.
```

---

## 4. 세 원칙의 상호관계와 우선순위

| 충돌 상황 | 권장 판단 |
|---|---|
| DRY 추상화가 KISS를 크게 저해 | **KISS 우선**(AHA: 성급한 추상화 회피) |
| DRY vs YAGNI(미래 확장 추상화) | **YAGNI 우선**(3회 반복 전엔 추상화 보류) |
| 단순화를 위해 지식 중복 허용? | **일시적 허용** → 패턴/테스트로 **정제 시점** 명시 |

**권장 흐름**  
1) **작동하게**(Green) → 2) **단순하게**(KISS) → 3) **공통화**(DRY) → 4) **확장 시점에만**(YAGNI).

---

## 5. 케이스 스터디: 주문 할인 모듈

### 5.1 초기(중복·복잡·과설계 전)
```java
class OrderService {
    int checkout(int price, String coupon, boolean vip){
        int discounted = price;
        if (coupon != null && coupon.startsWith("FIX")) {
            discounted -= 1000;
        }
        if (vip && price > 100000) {
            discounted *= 0.9;
        }
        // ... 다른 조건들 복잡하게 이어짐
        return discounted;
    }
}
```
- **문제**: 조건 중복, 규칙 산재 → DRY 위반, KISS 위반 가능.

### 5.2 1차 리팩토링(KISS: 의도 드러내기)
```java
class OrderService {
    int checkout(int price, String coupon, boolean vip){
        int d1 = applyCoupon(price, coupon);
        return applyVip(d1, vip);
    }
    private int applyCoupon(int price, String coupon){ /* ... */ }
    private int applyVip(int price, boolean vip){ /* ... */ }
}
```

### 5.3 공통화 시점(DRY + Strategy) — “세 번째 규칙 등장”
```java
interface DiscountRule { boolean applicable(Context c); int apply(Context c); }

final class FixCouponRule implements DiscountRule { /* ... */ }
final class VipRule       implements DiscountRule { /* ... */ }
// 이후 PercentageRule, SeasonalRule 등 추가

final class DiscountEngine {
    private final List<DiscountRule> rules;
    DiscountEngine(List<DiscountRule> rules){ this.rules = rules; }
    int applyAll(Context c){
        int price = c.price();
        for (var r: rules) if (r.applicable(c)) price = r.apply(new Context(price, c));
        return price;
    }
}
```

### 5.4 YAGNI 체크
- 아직 **조합 우선순위/스택/쿠폰중복** 요구가 없다면 **규칙 조합 엔진**(DSL)은 보류.  
- 필요해지면 그때 **정렬/우선순위/단일/다중 적용 전략**을 추가.

---

## 6. 테스트 전략 — 세 원칙을 뒷받침

- **KISS**: 짧은 함수 → **단위 테스트** 수월.  
- **DRY**: 공통 규칙은 **계약 테스트**로 여러 구현에 동일 적용.
- **YAGNI**: “필요할 때 구현”의 **증거**는 **실패하는 테스트**.

```java
// 계약 테스트(공유 테스트) 예
abstract class DiscountRuleContract {
    protected abstract DiscountRule sut();

    @Test void apply_when_applicable_changes_price(){
        var ctx = new Context(10000, /*...*/);
        if (sut().applicable(ctx)) assertNotEquals(10000, sut().apply(ctx));
    }
}
```

---

## 7. 도구/실천 목록

- **정적 분석/중복 탐지**: SonarQube, PMD CPD, Detekt(Kotlin), ESLint/TS(sonarjs)  
- **복잡도 지표**: CC, Cognitive Complexity, Duplication %, Maintainability Index  
- **리뷰 체크리스트**  
  - [ ] 함수/클래스가 **하나의 책임**만 수행  
  - [ ] **같은 지식**이 복수 위치에 있지 않음  
  - [ ] 미래 대비 코드/추상화가 **테스트 없이** 추가되지 않음  
  - [ ] **가드절/초기 리턴**으로 분기 평탄화  
  - [ ] 상수/정책이 **중앙 정의**되어 있음

---

## 8. 의사결정 트리(현장 카드)

```
1) 지금 실패하는 테스트/요구가 있는가?
   ├─ 아니오 → YAGNI(보류)
   └─ 예     → 2) 가장 단순한 해법인가? (KISS)
                 ├─ 아니오 → 단순화
                 └─ 예     → 3) 동일 지식이 3곳 이상 반복되는가?
                               ├─ 아니오 → 유지(AHA)
                               └─ 예     → DRY(추출/전략/템플릿)
```

---

## 9. 안티패턴 ↔ 리팩토링

| 악취 | 증상 | 보조 원칙 | 리팩토링 |
|---|---|---|---|
| Speculative Generality | 쓰이지 않는 추상화/팩토리 | YAGNI | 제거/인라인/구현으로 축소 |
| Shotgun Surgery | 작은 변경이 많은 파일에 파급 | DRY | 정책 공통화/모듈화 |
| Over-Abstracted | 이해 어려운 계층/패턴 남발 | KISS | 계층 축소/합성 단순화 |
| Duplicate Knowledge | 정책 로직이 여기저기 | DRY | 정책/룰 엔진·전략 추출 |

---

## 10. 수식으로 보는 위험 직관(간단 모델)

변경 위험을 공개면 수·중복·복잡도에 근사:

$$
\text{Risk} \approx (\text{PublicSurface}) \times (\text{DuplicationRate}) \times (\text{CognitiveComplexity})
$$

- **PublicSurface**: 공개 메서드·DTO 필드 수  
- **DuplicationRate**: 코드/지식 중복 비율  
- **CognitiveComplexity**: 이해 난이도 지표

→ KISS는 **복잡도 감소**, DRY는 **중복 감소**, YAGNI는 **공개면 축소**로 **Risk↓**.

---

## 11. 다양한 언어 스니펫(간단)

### 11.1 Python — KISS 가드절
```python
def purchase(user, item):
    if not user.is_active:  # guard
        return "denied"
    if not item.in_stock():
        return "sold-out"
    return "ok"
```

### 11.2 TypeScript — DRY 상수/타입 중앙화
```ts
export const CURRENCIES = ["KRW","USD","EUR"] as const;
export type Currency = typeof CURRENCIES[number];
```

### 11.3 C# — YAGNI: 인터페이스 남발 금지
```csharp
// Before
public interface IXmlExportable { string ToXml(); } // 당장 필요 없음

// After
// 요구가 생길 때 확장 메서드나 어댑터로 추가
```

---

## 12. 요약 표

| 원칙 | 핵심 | 이점 | 주의 |
|---|---|---|---|
| **DRY** | 지식 중복 제거 | 일관성·유지보수성 | 성급한 추상화 금지(AHA), Rule of Three |
| **KISS** | 단순성 유지 | 이해·변경 용이, 버그↓ | 과도한 패턴/계층 금지 |
| **YAGNI** | 지금 필요한 것만 | 낭비/과설계 방지 | 제거 용이한 경로로 실험 |

---

## 13. 결론

- **DRY**로 **지식**을 한 곳에 모으고, **KISS**로 **형태**를 단순하게 유지하며, **YAGNI**로 **시점**을 통제하라.  
- 적용 순서는 **작동 → 단순화 → 공통화 → 필요 시 확장**.  
- 테스트와 측정(중복·복잡도·공개면)이 **증거**를 제공한다.  
- 결국 팀이 매일 내리는 **수많은 작은 결정**이 코드베이스의 운명을 가른다. 이 세 원칙은 그 결정을 **일관되게** 만들어준다.