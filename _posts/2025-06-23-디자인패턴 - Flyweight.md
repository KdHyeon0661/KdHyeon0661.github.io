---
layout: post
title: 디자인패턴 - Flyweight
date: 2025-06-23 19:20:23 +0900
category: 디자인패턴
---
# Flyweight (플라이웨이트 패턴)

## ✅ 정의

**플라이웨이트 패턴(Flyweight Pattern)**은 **많은 수의 유사한 객체**를 **공유하여 메모리 사용을 줄이기 위한 구조 패턴**입니다.  
불변이며 공유 가능한 객체를 만들어 **중복 인스턴스를 방지하고 시스템 자원 사용을 최적화**하는 데에 목적이 있습니다.

> “같은 것은 공유하고, 다른 것만 다르게 유지하자”

---

## 🎯 의도 (Intent)

- 대량의 객체가 필요할 때, **중복 데이터를 공유**하여 메모리 사용량을 최소화
- 반복적으로 생성되는 객체를 **공통된 상태(내부 상태)**와 **변하는 상태(외부 상태)**로 분리
- 외부 상태만 바꿔가며 **하나의 객체를 재활용**하는 방식

---

## 📦 구조 (UML)

```
┌────────────┐
│   Client   │
└────┬───────┘
     │
     ▼
┌────────────┐       uses       ┌────────────────┐
│ Flyweight  │◄────────────────│ FlyweightFactory│
└────────────┘                 └────────────────┘
     ▲
     │
┌────────────┐
│ConcreteFly │
└────────────┘
```

- `Flyweight`: 공유 객체가 구현해야 할 인터페이스
- `ConcreteFlyweight`: 공유 가능한 실제 객체
- `FlyweightFactory`: 이미 생성된 객체를 캐싱하고 재사용하도록 제공
- `Client`: 외부 상태를 전달하며 플라이웨이트 객체 사용

---

## 🧑‍💻 구현 예시 (Python)

```python
# Flyweight 클래스
class TreeType:
    def __init__(self, name, color, texture):
        self.name = name          # 내부 상태
        self.color = color        # 내부 상태
        self.texture = texture    # 내부 상태

    def draw(self, x, y):         # 외부 상태는 호출 시 전달
        print(f"[{self.name}] 나무를 위치({x}, {y})에 그림. 색상: {self.color}, 텍스처: {self.texture}")

# Flyweight Factory
class TreeFactory:
    _tree_types = {}

    @classmethod
    def get_tree_type(cls, name, color, texture):
        key = (name, color, texture)
        if key not in cls._tree_types:
            cls._tree_types[key] = TreeType(name, color, texture)
        return cls._tree_types[key]

# 클라이언트
class Tree:
    def __init__(self, x, y, tree_type: TreeType):
        self.x = x
        self.y = y
        self.type = tree_type

    def draw(self):
        self.type.draw(self.x, self.y)

# 사용 예시
forest = []

# 수천 그루의 나무를 그려야 한다면
for i in range(0, 10000, 1000):
    t1 = Tree(i, i, TreeFactory.get_tree_type("자작나무", "초록", "거칠게"))
    t2 = Tree(i+5, i+10, TreeFactory.get_tree_type("소나무", "진초록", "가늘게"))
    forest.extend([t1, t2])

# 그리기
for tree in forest[:4]:
    tree.draw()
```

**출력 예:**
```
[자작나무] 나무를 위치(0, 0)에 그림. 색상: 초록, 텍스처: 거칠게
[소나무] 나무를 위치(5, 10)에 그림. 색상: 진초록, 텍스처: 가늘게
[자작나무] 나무를 위치(1000, 1000)에 그림. 색상: 초록, 텍스처: 거칠게
[소나무] 나무를 위치(1005, 1010)에 그림. 색상: 진초록, 텍스처: 가늘게
```

---

## ✅ 장점

- **메모리 사용량 감소** (공유 가능한 객체 재사용)
- **객체 수 감소**로 GC 부담 줄어듦
- 대규모 객체를 다룰 때 **성능 최적화**에 효과적

---

## ⚠️ 단점

- **내부 상태와 외부 상태를 구분해야 하는 복잡성** 증가
- 공유 객체의 **불변성 보장 필요**
- 코드를 이해하기 어렵고, **설계 난이도 상승**
- 객체의 **식별성이 희생될 수 있음**

---

## 📌 사용 사례

| 사례 | 설명 |
|------|------|
| **게임에서의 타일, 나무, 건물** | 수천 개의 동일 리소스를 한 객체로 공유 |
| **문자 렌더링 시스템 (예: 글꼴)** | 같은 글자는 동일 객체로 공유하여 메모리 절약 |
| **아이콘 관리** | 동일 아이콘을 반복적으로 사용할 때 |
| **IDE의 문법 강조** | "같은 단어는 같은 색상 스타일"을 공유 객체로 처리 |
| **브라우저 DOM 요소 렌더링** | 유사한 엘리먼트를 공유하고 좌표만 다르게 적용 |

---

## 💡 Flyweight vs Singleton vs Pool

| 패턴 | 목적 | 공유 방식 | 특징 |
|------|------|-----------|------|
| **Flyweight** | 메모리 절약 | 내부 상태 공유 | 대량의 객체 공유 |
| **Singleton** | 유일 인스턴스 보장 | 단일 전역 객체 | 공유라기보단 고정 |
| **Object Pool** | 재사용성 극대화 | 사용 후 반납 | 상태 변경 허용됨 |

---

## 🧠 마무리

**플라이웨이트 패턴은 "공유를 통한 최적화"에 중점을 둔 패턴**입니다.  
객체 수가 많은 시스템에서 동일한 속성을 가진 객체들을 공유함으로써, **메모리 효율성**을 획기적으로 높일 수 있습니다.

다만, **외부 상태를 잘 분리하고, 불변성**을 잘 유지해야 하는 설계의 어려움이 있기 때문에,  
실제로는 **게임 엔진, 그래픽 렌더링, 텍스트 렌더러, 캐시 시스템** 등에서 전략적으로 사용됩니다.