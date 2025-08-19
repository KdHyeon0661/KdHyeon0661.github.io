---
layout: post
title: Java - Garbage Collection
date: 2025-08-04 21:20:23 +0900
category: Java
---
# Garbage Collection (GC) 개요

Garbage Collection(GC)은 **Java Virtual Machine(JVM)**에서 자동으로 수행되는 **메모리 관리 메커니즘**이다.  
개발자가 직접 메모리를 해제하지 않아도, JVM이 더 이상 사용되지 않는 객체를 찾아 힙(Heap) 메모리에서 제거하여 메모리 누수와 프로그램 충돌을 방지한다.

---

## 1. 왜 Garbage Collection이 필요한가?

- Java는 **자동 메모리 관리**를 지향하며, 프로그래머가 직접 `malloc`/`free` 같은 메모리 할당 해제를 하지 않는다.
- 명시적 해제 없이도 **안전하고 효율적인 메모리 사용**을 보장하기 위해서다.
- 그렇지 않으면 메모리 누수, dangling pointer, 이중 해제 같은 문제 발생 가능.

---

## 2. GC의 기본 원리

1. **참조 분석(Reachability Analysis)**  
   JVM은 객체가 프로그램에서 더 이상 참조되지 않는 시점을 찾는다.  
   즉, 어떤 객체가 **루트(Root)**(스택, static 변수, CPU 레지스터 등)로부터 참조 불가능해지면 가비지(더 이상 필요 없는 객체)로 간주한다.

2. **가비지 객체 탐지 및 회수**  
   참조되지 않는 객체들을 힙에서 제거하여 메모리를 확보한다.

3. **힙 메모리 정리 및 단편화 방지**  
   일부 GC 알고리즘은 객체를 이동(Compaction)시켜 메모리 단편화를 줄이고 할당 속도를 높인다.

---

## 3. JVM 힙 영역과 GC 대상

- JVM 힙은 크게 **Young Generation**과 **Old Generation (Tenured Generation)**으로 나뉜다.
- 대부분의 객체가 Young Gen에서 생성되고 짧은 시간 내에 사라진다.
- 오래 살아남는 객체는 Old Gen으로 승격(promote)된다.
- GC는 주로 Young Gen에서 자주 발생하는 `Minor GC`와 Old Gen에서 발생하는 `Major(Full) GC`로 구분됨.

---

## 4. 주요 Garbage Collection 알고리즘

### 4.1 Serial GC
- 단일 스레드 기반 GC.
- 소규모, 단일 CPU 환경에서 주로 사용.
- 간단하지만, GC 중에는 애플리케이션 실행이 중단됨(Stop-the-World).

### 4.2 Parallel GC (Throughput Collector)
- 멀티스레드를 사용해 Young Gen GC를 병렬로 처리.
- 처리량 극대화에 초점.
- Full GC도 병렬 처리.

### 4.3 CMS (Concurrent Mark Sweep)
- Old Gen 영역을 **동시(Concurrent)**로 마킹하고 수집하여 Pause 시간을 줄임.
- 낮은 지연 시간 환경에 적합했으나, 복잡성과 단편화 문제로 현재는 G1 GC로 대체 추세.

### 4.4 G1 GC (Garbage First)
- 힙을 여러 개의 Region(작은 영역)으로 분할하여 GC 수행.
- Young과 Old Gen 영역이 동적으로 할당됨.
- Pause 시간 예측이 가능하도록 설계되어 응답 시간 제어에 적합.
- 현대 JVM의 기본 GC로 자리 잡음.

### 4.5 ZGC, Shenandoah
- 초저지연(very low pause) GC로, 수백 GB 규모 힙도 짧은 중단 시간으로 처리 가능.
- 최근 Java 버전에서 도입.

---

## 5. GC 동작 단계 (가장 일반적인 Mark-and-Sweep 방식)

1. **Mark(마킹)**  
   루트에서 도달 가능한 모든 객체를 표시.

2. **Sweep(스윕, 제거)**  
   마킹되지 않은 객체들을 메모리에서 해제.

3. **Compact(압축, 선택적)**  
   메모리 단편화를 줄이기 위해 살아남은 객체들을 한쪽으로 몰아 배치.

---

## 6. GC 트리거(발생 조건)

- Young Gen 영역이 부족할 때 Minor GC 발생.
- Old Gen 영역이 부족하거나 시스템 메모리가 부족할 때 Major GC 발생.
- 직접 호출은 권장하지 않지만, `System.gc()`는 GC 실행을 요청하는 메서드.

---

## 7. Stop-the-World 이벤트

- GC 수행 시 모든 애플리케이션 스레드가 일시 중단되는 현상.
- Pause 시간 감소가 GC 알고리즘의 중요한 목표.

---

## 8. GC와 성능

- GC가 자주, 오래 걸리면 애플리케이션 지연과 처리량 저하로 이어짐.
- GC 튜닝은 JVM 옵션과 힙 사이즈 조절, 적절한 GC 알고리즘 선택으로 가능.
- GC 로그 분석은 병목점 식별에 필수.

---

## 9. 간단한 GC 로그 예시 (Java 8 G1GC)
```
[GC pause (G1 Evacuation Pause) (young)
Desired survivor size 524288 bytes, new threshold 1 (max 15)
- age   1:    64000 bytes,    64000 total
- age   2:    204800 bytes,   268800 total
[Parallel Time: 0.4 ms, GC Workers: 4]
   [Eden: 1024K(1024K)->0B(512K) Survivors: 512K->512K Heap: 10M(15M)->9M(15M)]
```

---

## 10. 결론

- Java GC는 메모리 누수 방지와 안정적인 메모리 관리를 위한 필수 기능.
- JVM과 애플리케이션 요구사항에 따라 다양한 GC 알고리즘 선택과 튜닝이 필요.
- GC 원리와 동작 방식을 이해하면 성능 최적화 및 문제 해결에 큰 도움이 됨.
