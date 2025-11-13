---
layout: post
title: Spring - JPA 기본
date: 2025-10-11 22:25:23 +0900
category: Spring
---
# 4. 데이터 접근 I — JPA 기본
> 영속성 컨텍스트/엔티티·값 타입/연관관계, 지연·즉시 로딩과 N+1/캐시, 트랜잭션 전파·격리·롤백을 **실행 가능한 예제**와 함께 정리한다.
> 환경 가정: Spring Boot 3.3+, Hibernate ORM 6.x, Java 21, Gradle, PostgreSQL 16 (RDB 특성은 DB마다 조금씩 다를 수 있음).

---

## A. 영속성 컨텍스트(Persistence Context)

### A-1. 개념
- **영속성 컨텍스트** = 엔티티 인스턴스를 **1차 캐시(Identity Map)**에 저장하고 **변경 감지(Dirty Checking)**를 통해 트랜잭션 종료 시점(or flush 시점)에 DB와 동기화하는 **세션 범위의 컨텍스트**.
- 단일 트랜잭션 동안 같은 `@Id`(PK)를 가진 엔티티는 **동일한 자바 객체**(동일성 `==`)를 반환한다.

### A-2. 상태 전이(State Transition)
1) **비영속(new/transient)**: 컨텍스트와 무관한 새 객체
2) **영속(managed)**: 컨텍스트가 관리(1차 캐시에 등록)
3) **준영속(detached)**: 컨텍스트에서 떨어진 상태(더 이상 변경 감지 X)
4) **삭제(removed)**: 삭제 예약 상태

```java
// A-2 예제: 상태 전이
var m = new Member("alice"); // 비영속
em.persist(m);               // 영속 (1차 캐시에 등록, INSERT는 flush 시점)
em.detach(m);                // 준영속 (변경 감지 대상에서 제외)
em.remove(m2);               // 삭제 예약 (flush 시 DELETE)
```

### A-3. 1차 캐시, 동일성 보장, 쓰기 지연(Transactional Write-Behind)
- **1차 캐시**: 동일 트랜잭션에서 같은 엔티티를 조회할 때 **조회 쿼리 최소화** + **동일 객체 보장**.
- **쓰기 지연**: `persist()` 시점에는 즉시 INSERT를 날리지 않고, **flush**에서 SQL 배치로 한 번에 보냄(성능 ↑).
- **Flush 트리거**:
  - 트랜잭션 커밋 직전(`commit → flush → SQL → commit`)
  - JPQL/Criteria 실행 직전
  - 명시적 `em.flush()` 호출

```java
// A-3 예제: 동일성 보장
Member m1 = em.find(Member.class, 1L);
Member m2 = em.find(Member.class, 1L);
assert m1 == m2; // true (동일성 보장)
```

### A-4. 변경 감지(Dirty Checking)와 Flush
- 영속 엔티티의 필드 변경 → Flush 시점에 **변경된 필드만 UPDATE**.
- 하이버네이트는 스냅샷을 보관하여 변경 여부를 판단.

```java
// A-4 예제: 변경 감지
Member m = em.find(Member.class, 1L);
m.changeName("Alice");   // setter or 도메인 메서드
// em.update() 같은 호출은 없음
// 트랜잭션 커밋 시 UPDATE SQL 자동 전송
```

> **주의**: 영속/준영속 구분이 중요하다. **준영속(detached)** 상태에서 필드 변경은 DB에 반영되지 않는다. 이때는 `em.merge()`로 다시 영속화해야 한다(단, 병합은 “새 엔티티 인스턴스”를 반환함).

---

## B. 엔티티/값 타입

### B-1. 엔티티 vs 값 타입
- **엔티티**: 식별자(@Id)로 구분되는 **생명주기**가 있는 객체. 독립적으로 저장/삭제.
- **값 타입(Value Type)**: **식별자 없음**, 불변(권장), **소유자 엔티티에 종속**. JPA에서는 `@Embeddable`로 표현.

### B-2. 값 타입(임베디드)
```java
// B-2-1) 값 타입 정의
@Embeddable
public record Address(String city, String street, String zip) {}

// B-2-2) 엔티티 내에서 사용
@Entity
@Table(name="member")
public class Member {
  @Id @GeneratedValue private Long id;
  private String username;

  @Embedded
  private Address address;  // city/street/zip 컬럼으로 내장됨

  // 생성자/게터/도메인 메서드...
}
```

- **불변성**: 값 타입은 가급적 **불변 객체**로 설계(`record` 적합). 변경은 **새 인스턴스 교체**로.

### B-3. 컬렉션 값 타입(권장X, 대안 제시)
- JPA의 `@ElementCollection`은 별도 테이블에 **소유자 PK + 값**으로 저장한다.
- 변경 추적/삭제 전략이 까다롭고, 실무에서는 **엔티티**로 승격하는 편이 안정적.

```java
@ElementCollection
@CollectionTable(name="member_tags", joinColumns=@JoinColumn(name="member_id"))
@Column(name="tag")
private Set<String> tags = new HashSet<>();
```

> **실무 팁**: 태그/속성 같은 값 모음은 **별도 엔티티**(ex. `MemberTag`)로 승격해 PK/인덱스/감사 등을 확보하자.

### B-4. 열거형(ENUM)
- `@Enumerated(EnumType.STRING)`을 권장: **ORDINAL(숫자)**는 변경·삽입 시 의미 붕괴 위험.

```java
public enum OrderStatus { CREATED, PAID, CANCELED }

@Enumerated(EnumType.STRING)
private OrderStatus status;
```

---

## C. 연관관계 매핑

### C-1. 방향과 소유
- **단방향** vs **양방향**: JPA는 **객체 그래프**와 **테이블 관계**를 매핑해야 한다.
- **연관관계의 주인(owning side)**: 외래키를 가진 쪽(일반적으로 `@ManyToOne`)이 주인.
- 양방향에서는 **주인만이 외래키를 관리**한다(`mappedBy`는 읽기 전용).

### C-2. 다대일(N:1) — 가장 흔함
```java
@Entity
class Order {
  @Id @GeneratedValue Long id;

  @ManyToOne(fetch = FetchType.LAZY) // 기본은 LAZY 권장
  @JoinColumn(name="member_id")      // FK
  private Member member;

  // ...
}
```

```java
@Entity
class Member {
  @Id @GeneratedValue Long id;
  private String username;

  @OneToMany(mappedBy = "member", orphanRemoval = true, cascade = CascadeType.ALL)
  private List<Order> orders = new ArrayList<>();
}
```

- **주인**: `Order.member` (`@ManyToOne`)
- `Member.orders`는 **거울(비주인)** — `mappedBy="member"`

> **양방향 연관관계 편의 메서드**
```java
public void addOrder(Order o){
  orders.add(o);
  o.setMember(this);
}
```

### C-3. 일대일(1:1)
- 외래키를 어느 테이블에 둘지 **업무 특성**에 맞게 결정.
- JPA는 LAZY 프록시 최적화가 까다롭다(하이버네이트는 대체 프록시 전략 사용).

```java
@Entity
class MemberProfile {
  @Id private Long id;

  @MapsId // Member PK를 FK로 공유
  @OneToOne(fetch=FetchType.LAZY)
  @JoinColumn(name="member_id")
  private Member member;

  private String bio;
}
```

### C-4. 일대다(1:N) 단방향 (권장X)
- `@OneToMany` 단방향은 **조인 테이블** or **업데이트 2번** 발생으로 비효율적.
- 가능하면 **다대일 양방향**으로 모델링.

### C-5. 다대다(M:N) (권장X, 조인 엔티티 권장)
- 순수 `@ManyToMany`는 **추가 컬럼**(수량/가격/상태)이 필요해지면 곧 **파괴**됨.
- **조인 엔티티**(예: `OrderItem`)로 **명시적** 매핑.

```java
@Entity
class OrderItem {
  @Id @GeneratedValue Long id;

  @ManyToOne(fetch = FetchType.LAZY) @JoinColumn(name="order_id")
  private Order order;

  @ManyToOne(fetch = FetchType.LAZY) @JoinColumn(name="product_id")
  private Product product;

  private int quantity;
  private long price;
}
```

### C-6. 고아 객체 제거 & Cascade
- `orphanRemoval = true`: 컬렉션에서 제거되면 DB에서 **DELETE**.
- `cascade = CascadeType.ALL`: 부모 영속/삭제에 자식도 전이.
  - **주의**: 생명주기 동일한 **Aggregate 내부**에서만 사용(DDD 관점).

---

## D. 지연 로딩(LAZY) / 즉시 로딩(EAGER), N+1, 페치 전략

### D-1. LAZY vs EAGER
- **LAZY**: 연관 엔티티를 **프록시**로 두고 실제 접근 시 쿼리 실행(권장 기본값).
- **EAGER**: 엔티티 조회 시 연관 엔티티도 즉시 로딩(예상치 못한 쿼리 폭발/N+1).

```java
@ManyToOne(fetch = FetchType.LAZY)
private Member member;
```

> **규칙**: 모든 연관은 **LAZY**로 시작하고, 조회 케이스별 **Fetch Join/EntityGraph**로 필요한 것만 묶어 가져온다.

### D-2. N+1 문제
- **설명**: 루트 엔티티 **N건 조회** + 각 엔티티의 연관을 개별 조회 → 총 1+N 쿼리.
- **발생 패턴**: 컬렉션/연관을 **루프에서 접근**할 때.

```java
// N+1 예시 (반패턴)
List<Order> orders = em.createQuery(
  "select o from Order o", Order.class).getResultList();
for (Order o : orders) {
  System.out.println(o.getMember().getUsername()); // 각 o마다 member 조회 쿼리 발생
}
```

### D-3. 해결 전략
1) **Fetch Join**(JPQL)
```java
List<Order> orders = em.createQuery(
  "select o from Order o join fetch o.member", Order.class)
  .getResultList();
```
- **주의**: 컬렉션(fetch join) + 페이징(`setFirstResult/setMaxResults`)은 결과 왜곡 가능 → **배치 페치** 또는 **서브쿼리 페치** 전략 고려.

2) **EntityGraph**
```java
@EntityGraph(attributePaths={"member"})
@Query("select o from Order o")
List<Order> findAllWithMember();
```

3) **배치 사이즈 설정**
- 하이버네이트: `hibernate.default_batch_fetch_size=100` or 개별 `@BatchSize(size=100)`
- 프록시 초기화 시 **IN 쿼리**로 한꺼번에 가져옴.

```yaml
spring:
  jpa:
    properties:
      hibernate.default_batch_fetch_size: 100
```

4) **DTO Projection** (쿼리와 화면/응답 맞춤)
```java
public record OrderView(Long id, String memberName, long total) {}
List<OrderView> list = em.createQuery(
  "select new path.OrderView(o.id, m.username, o.total) " +
  "from Order o join o.member m", OrderView.class)
.getResultList();
```

> **요약**: “모든 연관 LAZY + 조회 케이스별 Fetch Join/EntityGraph/DTO”가 안전한 기본이다.

---

## E. 캐시 기초 — 1차 캐시 / 2차 캐시 / 쿼리 캐시

### E-1. 1차 캐시(영속성 컨텍스트)
- 트랜잭션 범위에서 **동일성 보장**, 반복 조회 최적화.
- flush 전까지 같은 엔티티 재조회 시 **쿼리 없이 반환**.

### E-2. 2차 캐시(세션팩토리 레벨, 선택)
- **엔티티 캐시**: 읽기 빈도 높고 변경 적은 엔티티에 적합(코드표/권역 데이터 등).
- **캐시 전략**(Hibernate): `READ_ONLY`, `NONSTRICT_READ_WRITE`, `READ_WRITE`, `TRANSACTIONAL`
- 활성화(예: Ehcache, Caffeine, Infinispan 연동)

```yaml
spring:
  jpa:
    properties:
      hibernate.cache.use_second_level_cache: true
      hibernate.cache.use_query_cache: true
      hibernate.cache.region.factory_class: org.hibernate.cache.jcache.JCacheRegionFactory
```

```java
@Entity
@Cacheable
@org.hibernate.annotations.Cache(usage = CacheConcurrencyStrategy.READ_ONLY)
class Region { @Id Long id; String name; }
```

### E-3. 쿼리 캐시(주의)
- **입력 파라미터 + 쿼리 문자열** 기반 캐시.
- 트랜잭션 격리/동기화가 어렵고, 실무에서는 **2차 캐시 + 애플리케이션 캐시(Redis 등)**를 선호.

```java
List<Region> r = em.createQuery("select r from Region r", Region.class)
  .setHint("org.hibernate.cacheable", true)
  .getResultList();
```

> **실무 권장**: 2차 캐시는 **정적/준정적 데이터**에 한해 신중히. 대부분은 **Spring Cache + Redis**로 어플리케이션 계층 캐시를 구성.

---

## F. 트랜잭션: 전파(Propagation), 격리(Isolation), 롤백

### F-1. ACID와 스프링 트랜잭션 추상화
- **ACID**: Atomicity, Consistency, Isolation, Durability
- 스프링은 `@Transactional`로 **플랫폼 트랜잭션 매니저**를 감싸 RDB/JPA/메시징 등 일관된 프로그래밍 모델 제공.

```java
@Service
public class OrderService {
  private final OrderRepository repo;
  public OrderService(OrderRepository repo){ this.repo = repo; }

  @Transactional
  public Long place(Order o) { repo.save(o); return o.getId(); }
}
```

### F-2. 전파(Propagation)
- **REQUIRED(기본)**: 트랜잭션 있으면 참여, 없으면 새로 시작.
- **REQUIRES_NEW**: **항상 새 트랜잭션**(기존 트랜잭션 **일시 중단**, 별도 커밋/롤백).
- **MANDATORY**: 반드시 기존 트랜잭션 필요(없으면 예외).
- **SUPPORTS**: 있으면 참여, 없으면 비트랜잭션.
- **NOT_SUPPORTED**: 트랜잭션 **중단**하고 비트랜잭션으로 실행.
- **NEVER**: 트랜잭션 있으면 예외.
- **NESTED**: 동일 트랜잭션 내 **세이브포인트**. 일부만 롤백 가능(DB/드라이버 지원 필요).

```java
@Transactional(propagation = Propagation.REQUIRES_NEW)
public void audit(AuditLog log) { auditRepo.save(log); }
```

> **패턴**: 본거래 실패 시에도 **감사 로그는 남겨야** 하면 `REQUIRES_NEW`로 분리.

### F-3. 격리 수준(Isolation)
- **DEFAULT**: 드라이버/DB 기본(보통 READ COMMITTED)
- **READ_UNCOMMITTED**: Dirty Read 허용(권장X)
- **READ_COMMITTED**: **커밋된 데이터만** 읽음(가장 많이 사용, PostgreSQL 기본)
- **REPEATABLE_READ**: 반복 조회 **같은 값 보장**(InnoDB는 GAP Lock/Phantom 처리 다름)
- **SERIALIZABLE**: 완전 직렬화(성능 비용 큼)

```java
@Transactional(isolation = Isolation.REPEATABLE_READ)
public void doWork() { /* ... */ }
```

> **DB 차이**: PostgreSQL은 REPEATABLE READ가 **스냅샷 격리**. MySQL(InnoDB)의 REPEATABLE READ는 Phantom Read 방지 여부/락 동작이 다르니 주의.

### F-4. 롤백 전략
- 기본: **런타임 예외(언체크)** 발생 시 롤백, **체크 예외**는 커밋.
- 조정: `rollbackFor`, `noRollbackFor`

```java
@Transactional(rollbackFor = IOException.class, noRollbackFor = BusinessWarning.class)
public void process() throws IOException { /* ... */ }
```

- **읽기 전용 최적화**: `@Transactional(readOnly = true)` → 플러시 최적화/힌트(드라이버 의존)

### F-5. 트랜잭션 경계와 LAZY
- **서비스 계층**에서 트랜잭션 시작/종료.
- 컨트롤러에서 **LAZY 필드 접근** 시 이미 트랜잭션이 끝나면 `LazyInitializationException`.
- 해결: 서비스 계층에서 DTO로 변환 후 반환(또는 OSIV 정책 고려하되 **권장X**).

### F-6. 예시: 전파/격리/롤백 시나리오

```java
@Service
public class PayService {
  private final OrderService orderService;
  private final PaymentGateway gateway;
  private final AuditService audit;

  public PayService(OrderService orderService, PaymentGateway gateway, AuditService audit) {
    this.orderService = orderService; this.gateway = gateway; this.audit = audit;
  }

  @Transactional(isolation = Isolation.READ_COMMITTED)
  public Long pay(String userId, long amount) {
    Long orderId = orderService.create(userId, amount); // REQUIRED
    try {
      gateway.charge(userId, amount); // 외부 API
      audit.success(orderId);         // REQUIRES_NEW — 본거래와 분리
      return orderId;
    } catch (PaymentException e) {
      audit.fail(orderId, e.getCode()); // REQUIRES_NEW — 실패 로그 남김
      throw e; // 런타임 예외 → 전체 롤백
    }
  }
}

@Service
class AuditService {
  @Transactional(propagation = Propagation.REQUIRES_NEW)
  public void success(Long orderId) { /* insert audit success */ }
  @Transactional(propagation = Propagation.REQUIRES_NEW)
  public void fail(Long orderId, String code) { /* insert audit fail */ }
}
```

> **해설**: 결제 성공/실패 **감사 로그는 항상 남는다**(별도 트랜잭션). 결제 자체는 실패 시 전체 롤백.

---

## G. 실전 예제 — “주문 도메인”으로 종합

### G-1. 모델
- `Member(1) - (N) Order`
- `Order(1) - (N) OrderItem`
- `OrderItem(N) - (1) Product`
- `Member`는 내장 값 `Address` 보유

```java
// G-1-1) Embeddable
@Embeddable
public record Address(String city, String street, String zip) {}

// G-1-2) Member
@Entity
class Member {
  @Id @GeneratedValue Long id;
  String username;

  @Embedded Address address;

  @OneToMany(mappedBy="member", cascade=CascadeType.ALL, orphanRemoval=true)
  List<Order> orders = new ArrayList<>();

  public void addOrder(Order o){ orders.add(o); o.setMember(this); }
}

// G-1-3) Product
@Entity
class Product {
  @Id @GeneratedValue Long id;
  String name;
  long price;
}

// G-1-4) Order + OrderItem
@Entity @Table(name="orders")
class Order {
  @Id @GeneratedValue Long id;

  @ManyToOne(fetch=FetchType.LAZY) @JoinColumn(name="member_id")
  Member member;

  @OneToMany(mappedBy="order", cascade=CascadeType.ALL, orphanRemoval=true)
  List<OrderItem> items = new ArrayList<>();

  @Enumerated(EnumType.STRING)
  OrderStatus status = OrderStatus.CREATED;

  public void addItem(OrderItem i){ items.add(i); i.setOrder(this); }
  public long total(){ return items.stream().mapToLong(it -> it.price * it.quantity).sum(); }
}

@Entity
class OrderItem {
  @Id @GeneratedValue Long id;

  @ManyToOne(fetch=FetchType.LAZY) @JoinColumn(name="order_id")
  Order order;

  @ManyToOne(fetch=FetchType.LAZY) @JoinColumn(name="product_id")
  Product product;

  int quantity;
  long price;
}
```

### G-2. 리포지토리 + N+1 케이스
```java
public interface OrderRepository extends JpaRepository<Order, Long> {

  // N+1 위험: order만 가져오고 member는 LAZY
  @Query("select o from Order o")
  List<Order> findAllBasic();

  // N+1을 줄이기: member fetch join
  @Query("select o from Order o join fetch o.member")
  List<Order> findAllWithMember();

  // 컬렉션 fetch join + distinct (중복 제거), 페이징 주의
  @Query("select distinct o from Order o join fetch o.member join fetch o.items i join fetch i.product")
  List<Order> findGraphEagerAll();
}
```

```java
@Service
class OrderQuery {
  private final OrderRepository repo;
  public OrderQuery(OrderRepository repo){ this.repo = repo; }

  @Transactional(readOnly = true)
  public List<OrderView> listBasic(){ // N+1 가능
    return repo.findAllBasic().stream()
      .map(o -> new OrderView(o.getId(), o.getMember().getUsername(), o.total()))
      .toList();
  }

  @Transactional(readOnly = true)
  public List<OrderView> listOptimized(){
    return repo.findAllWithMember().stream()
      .map(o -> new OrderView(o.getId(), o.getMember().getUsername(), o.total()))
      .toList();
  }
}

public record OrderView(Long id, String memberName, long total){}
```

> **로그로 확인**: `listBasic()`은 `select * from orders` + 각 row마다 `select * from member where id=?` → **N+1**.
> `listOptimized()`는 한 번의 **join fetch**로 `member`까지 로딩.

### G-3. 배치 페치 사이즈로 컬렉션 최적화
```yaml
spring:
  jpa:
    properties:
      hibernate.default_batch_fetch_size: 100
```
- `orders`를 먼저 불러오고, 루프에서 `o.getItems()` 접근 시 **IN(...)**으로 100개씩 한 번에 가져옴.

---

## H. 테스트 전략 — 슬라이스/통합 + 고립된 트랜잭션

### H-1. `@DataJpaTest`로 JPA 슬라이스
```java
@DataJpaTest
class OrderRepositoryTest {
  @Autowired OrderRepository repo;
  @Autowired TestEntityManager em;

  @Test
  void fetchJoinWorks(){
    List<Order> orders = repo.findAllWithMember();
    assertThat(orders).isNotEmpty();
    // LazyInitializationException 없이 member 접근 가능(이미 로딩됨)
    orders.forEach(o -> o.getMember().getUsername());
  }
}
```

### H-2. Testcontainers로 실DB 통합
```java
@SpringBootTest
@Testcontainers
class TxIsolationTest {
  static PostgreSQLContainer<?> PG = new PostgreSQLContainer<>("postgres:16");
  static { PG.start(); }

  @DynamicPropertySource
  static void props(DynamicPropertyRegistry r){
    r.add("spring.datasource.url", PG::getJdbcUrl);
    r.add("spring.datasource.username", PG::getUsername);
    r.add("spring.datasource.password", PG::getPassword);
  }

  @Autowired OrderService orderService;

  @Test
  void rollbackOnRuntimeException(){
    assertThatThrownBy(() -> orderService.payAndFail("u1", 1000L))
      .isInstanceOf(RuntimeException.class);
    // 데이터 일관성 검증...
  }
}
```

---

## I. 운영 체크리스트 & 베스트 프랙티스

### I-1. 매핑/모델링
- [ ] **모든 연관 LAZY** 기본.
- [ ] **다대다 금지** → 조인 엔티티로 모델링.
- [ ] 값 타입(`@Embeddable`)은 **불변**으로.
- [ ] `equals/hashCode`는 **식별자 불변성** 고려(영속 전/후 주의).

### I-2. 쿼리/성능
- [ ] 조회 API는 **fetch join/EntityGraph/DTO**로 N+1 차단.
- [ ] **배치 페치 사이즈** 활용.
- [ ] 페이징 + 컬렉션 fetch join은 지양(필요 시 **두 번 조회** 또는 서브쿼리 패턴).

### I-3. 트랜잭션
- [ ] 서비스 계층에서 **경계를 명확히** (`@Transactional`).
- [ ] 복잡한 시나리오는 **REQUIRES_NEW**로 분리(감사/알림).
- [ ] 읽기는 `readOnly=true`.
- [ ] **OSIV 비활성** 시(권장) 컨트롤러에서 LAZY 접근 금지 → DTO 변환을 서비스에서.

### I-4. 캐시
- [ ] 2차 캐시는 **정적 데이터**에 한정.
- [ ] 일반 데이터 캐시는 **Spring Cache + Redis**로.
- [ ] 캐시 무효화 시나리오(동시성/일관성) 문서화.

### I-5. 장애/데드락
- [ ] **락 타임아웃/재시도** 설계(낙관적 락 버전 필드, 비관적 락 시 대기 제한).
- [ ] 트랜잭션 시간 최소화(외부 호출 포함 금지).

---

## J. 자주 보는 오류 & 트러블슈팅

1) **`LazyInitializationException`**
   - 트랜잭션 밖에서 LAZY 접근 → 서비스에서 **DTO 변환** 후 반환 / 필요 시 페치 전략 명시.

2) **`MultipleBagFetchException`**(Hibernate)
   - 한 쿼리에 **여러 Bag(중복 허용 컬렉션)** fetch join → 컬렉션을 `Set`으로 바꾸거나 **한 번에 하나**만 fetch join.

3) **`TransientObjectException`**
   - 연관된 엔티티가 영속화되지 않은 상태에서 참조 → **cascade persist** 또는 **명시적 persist** 필요.

4) **`NonUniqueResultException`**
   - 단건 조회 쿼리에서 복수 결과 → 제약조건/where 확인, `getSingleResult()` 대신 `getResultList().stream().findFirst()`.

5) **식별자 생성 전략 미스매치**
   - PostgreSQL은 `IDENTITY` 대신 `SEQUENCE`가 일반적. `GenerationType.IDENTITY`는 배치 삽입에 불리.

```java
@Id
@GeneratedValue(strategy = GenerationType.SEQUENCE, generator = "order_seq")
@SequenceGenerator(name="order_seq", sequenceName="order_seq", allocationSize=50)
Long id;
```
- `allocationSize`로 시퀀스 호출 최적화.

---

## K. 확장 토픽(예고)

- **고급 매핑**: 상속 전략(JOINED/SINGLE_TABLE/TABLE_PER_CLASS), 복합키(`@EmbeddedId`, `@IdClass`)
- **잠금**: 낙관적/비관적 락, 버전 필드
- **쿼리 언어**: JPQL/Criteria/QueryDSL 실전 패턴
- **배치 처리**: 수십만 건 대량 INSERT/UPDATE의 전략(하이버네이트 배치/StatelessSession/파티셔닝)

---

## L. 한 페이지 요약

- **영속성 컨텍스트**는 1차 캐시/변경 감지/쓰기 지연으로 개발 생산성과 성능을 제공한다.
- **엔티티/값 타입**을 구분하고, 값 타입은 **불변**으로 설계한다.
- **연관관계**는 **다대일 양방향**이 기본, 컬렉션은 주인 쪽에서만 관리. 다대다는 **조인 엔티티**로.
- **로딩 전략**은 **LAZY**가 기본, 조회 케이스별 **fetch join/EntityGraph/DTO**로 최적화한다.
- **캐시**는 1차 캐시가 기본, 2차/쿼리 캐시는 신중히.
- **트랜잭션**은 서비스 계층에서 전파/격리/롤백을 명확히 설계하고, OSIV에 의존하지 않는다.
