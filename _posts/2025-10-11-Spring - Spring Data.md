---
layout: post
title: Spring - Spring Data
date: 2025-10-11 23:25:23 +0900
category: Spring
---
# 5) 데이터 접근 II — Spring Data
> Spring Data JPA의 리포지토리 패턴에서 시작해 **Query Method / `@Query` / QueryDSL 개요**, 그리고 **페이징·정렬·동적 검색·Specification**까지 한 번에 정리합니다.  
> 환경 가정: Spring Boot 3.3+, Spring Data JPA, Hibernate 6.x, Java 21, PostgreSQL 16.

---

## A. Spring Data JPA 리포지토리 패턴

### A-1. 핵심 철학
- **“레포지토리는 도메인 컬렉션”**: 영속성 기술(JPA/SQL)을 감추고 **의미 있는 쿼리 이름**으로 모델링.
- **관점 분리**: 비즈니스 흐름은 **서비스(유즈케이스)**가 담당, 레포지토리는 **데이터 접근 계약**만.

### A-2. `JpaRepository` 상속 구조
```java
public interface MemberRepository extends JpaRepository<Member, Long> {
  // + PagingAndSortingRepository + CrudRepository 기능 포함
}
```
- 제공 메서드: `save`, `findById`, `findAll`, `deleteById`, `count`, `existsById`, `findAll(Pageable)` …
- **트랜잭션**: 기본적으로 **읽기 메서드는 트랜잭션 없이도 가능**하지만,
  - 서비스 계층에서 `@Transactional(readOnly = true)` 권장(성능 힌트·플러시 방지).
  - 변경은 서비스 계층 `@Transactional`에서 수행.

### A-3. 단위 테스트(슬라이스) 기본
```java
@DataJpaTest
class MemberRepositoryTest {

  @Autowired MemberRepository repo;
  @Autowired TestEntityManager em;

  @Test
  void save_and_find() {
    Member m = new Member(null, "alice", "Alice");
    repo.save(m);
    em.flush(); em.clear();
    assertThat(repo.findById(m.getId())).isPresent();
  }
}
```

---

## B. Query Method — 이름으로 쿼리를 표현하기

### B-1. 네이밍 규칙 핵심
- `findBy`, `readBy`, `getBy`, `existsBy`, `countBy`, `deleteBy` …
- 조건 연결: `And`, `Or`, `Between`, `LessThan`, `GreaterThan`, `Like`, `StartingWith`, `Containing`, `In`, `IsNull`, `True`, `Before`, `After`, `IgnoreCase`…
- 정렬: `OrderBy…Asc/Desc`
- 중첩 속성: `findByAddressCity(String city)`

```java
List<Member> findByUsernameAndActiveTrueOrderByCreatedAtDesc(String username);
Optional<Member> findFirstByEmailEndingWith(String domain);
boolean existsByEmail(String email);
long countByActiveFalse();
```

**장점**: 선언만으로 실행.  
**한계**: 복잡한 조인/동적 필터/집계는 **가독성이 급격히 떨어짐** → `@Query`/QueryDSL로 승격.

### B-2. 파라미터 바인딩
```java
List<Member> findByEmailIn(Collection<String> emails);
List<Member> findTop10ByActiveTrueAndCreatedAtAfter(LocalDateTime since);
```

### B-3. Streaming & Slicing
```java
@QueryHints(@QueryHint(name = HINT_FETCH_SIZE, value = "1000"))
Stream<Member> findByActiveTrue();
```
- 대량 처리 시 스트림은 **트랜잭션 경계 내에서만** 사용하고 **반드시 close** 또는 try-with-resources.

---

## C. `@Query` — JPQL/네이티브로 표현력 확장

### C-1. JPQL `@Query`
```java
@Query("""
  select m from Member m
  join fetch m.team t
  where (:active is null or m.active = :active)
    and (:q is null or m.username like concat('%', :q, '%'))
""")
List<Member> search(@Param("active") Boolean active, @Param("q") String q);
```
- **장점**: 조인/조건/집계/DTO 프로젝션 자유롭다.
- **주의**: 엔티티 반환 시 N+1 위험 → fetch join/EntityGraph/배치 페치 크기 고려.

### C-2. DTO 프로젝션
```java
public record MemberView(Long id, String username, String teamName) {}

@Query("""
  select new com.example.MemberView(m.id, m.username, t.name)
  from Member m join m.team t
  where t.id = :teamId
""")
List<MemberView> findViewsByTeam(@Param("teamId") Long teamId);
```
- **장점**: **정확한 셀렉트** + **불필요한 로딩 차단**.

### C-3. 네이티브 쿼리
```java
@Query(value = """
  select m.id, m.username, t.name as team_name
  from members m join team t on t.id = m.team_id
  where t.name = :name
""", nativeQuery = true)
List<Object[]> rawByTeamName(@Param("name") String name);
```
- DB 특화 함수/윈도우/CTE/힌트 사용 가능.  
- **단점**: 이식성↓, 엔티티 상태/변경감지와 동기화 주의. DTO 매핑은 `SqlResultSetMapping` 또는 스프링 `JdbcTemplate`/`RecordClassMapper` 고려.

### C-4. 수정 쿼리(벌크 업데이트)
```java
@Modifying(clearAutomatically = true, flushAutomatically = true)
@Query("update Member m set m.active = false where m.lastLoginAt < :threshold")
int deactivateOld(@Param("threshold") LocalDateTime threshold);
```
- 벌크 업데이트는 **영속성 컨텍스트 무시** → 실행 후 **clear**로 1차 캐시 초기화 필요(`clearAutomatically=true` 권장).

---

## D. 스프링 데이터의 다양한 프로젝션

### D-1. 인터페이스 기반
```java
public interface MemberNameOnly {
  String getUsername();
}

List<MemberNameOnly> findByActiveTrue();
```

### D-2. 클래스 기반(생성자)
```java
public record MemberNameAndTeam(String username, String teamName) {}
@Query("""
  select new com.example.MemberNameAndTeam(m.username, t.name)
  from Member m join m.team t
""")
List<MemberNameAndTeam> list();
```

### D-3. 동적 프로젝션(리턴 타입 제네릭)
```java
<T> List<T> findByTeamId(Long teamId, Class<T> type);

// 사용
repo.findByTeamId(1L, MemberNameOnly.class);
repo.findByTeamId(1L, MemberView.class);
```

---

## E. 페이징(Page) / 슬라이싱(Slice) / 정렬(Sort)

### E-1. API 개요
- `Page<T>`: **총 건수** 포함. 리스트+페이지 메타.  
- `Slice<T>`: 다음 페이지 여부만, **count 쿼리 생략**으로 성능↑.  
- `List<T>`: 페이징 없이 전부.

```java
Page<Member> findByActiveTrue(Pageable pageable);
Slice<Member> findByTeamId(Long teamId, Pageable pageable);
```

### E-2. 요청 예
```java
Pageable pageable = PageRequest.of(0, 20, Sort.by(Sort.Order.desc("createdAt"), Sort.Order.asc("id")));
Page<Member> page = repo.findByActiveTrue(pageable);
```
- 정렬 속성은 **인덱스** 고려.  
- **Nested 정렬** 가능: `Sort.by("team.name").ascending()`(JPA가 조인 필요 → 주의)

### E-3. 커스텀 countQuery 최적화
```java
@Query(
  value = "select m from Member m join m.team t where t.id = :teamId",
  countQuery = "select count(m.id) from Member m where m.team.id = :teamId"
)
Page<Member> pageByTeam(@Param("teamId") Long teamId, Pageable pageable);
```
- 복잡한 fetch join을 **카운트에서는 제거**(불필요 조인 제거로 속도↑).

### E-4. `Slice` 사용 시점
- 모바일 **무한 스크롤** 등 “다음 페이지 존재 여부”만 필요할 때 적합.  
- **대용량 테이블**에서 전체 카운트가 고가일 때.

---

## F. 동적 검색 ① — Example & ExampleMatcher (간단 필터)

### F-1. Example (Query By Example; QBE)
```java
Member probe = new Member();
probe.setActive(true);
probe.setUsername("ali"); // like 검색 하고 싶다면 Matcher 사용

ExampleMatcher matcher = ExampleMatcher.matching()
    .withIgnorePaths("id", "createdAt")
    .withMatcher("username", m -> m.startsWith().ignoreCase());

Example<Member> example = Example.of(probe, matcher);
List<Member> result = repo.findAll(example);
```
- **장점**: 간단한 동적 필터에 빠르게 적용.  
- **한계**: 조인/복잡한 조건/OR 조합은 제한적.

---

## G. 동적 검색 ② — `Specification` (JPA Criteria 기반)

### G-1. `Specification` 인터페이스
```java
public interface MemberRepository extends JpaRepository<Member, Long>, JpaSpecificationExecutor<Member> { }
```

```java
public class MemberSpec {
  public static Specification<Member> active(Boolean v) {
    return (root, query, cb) -> (v == null) ? null : cb.equal(root.get("active"), v);
  }
  public static Specification<Member> usernameContains(String q) {
    return (root, query, cb) -> (q == null || q.isBlank()) ? null :
        cb.like(cb.lower(root.get("username")), "%"+q.toLowerCase()+"%");
  }
  public static Specification<Member> teamNameEquals(String team) {
    return (root, query, cb) -> {
      if (team == null) return null;
      Join<Member, Team> t = root.join("team", JoinType.LEFT);
      return cb.equal(t.get("name"), team);
    };
  }
}
```

```java
Specification<Member> spec = Specification
    .where(MemberSpec.active(true))
    .and(MemberSpec.usernameContains("ali"))
    .and(MemberSpec.teamNameEquals("Core"));

Page<Member> page = repo.findAll(spec, PageRequest.of(0, 20));
```

- **장점**: 조인/AND/OR/중첩 조건을 타입 세이프하게 조립.  
- **단점**: 코드가 장황, 복잡한 케이스는 **QueryDSL**로 넘어가면 가독성↑.

### G-2. 정렬/페이징과 함께
- `JpaSpecificationExecutor`의 `findAll(Specification, Pageable)` 사용.  
- **카운트 쿼리** 자동 생성(복잡하면 `@Query`처럼 countQuery 최적화가 어려움).

---

## H. QueryDSL 개요 — 타입 세이프 JPQL 빌더

> 팀이 **동적 조건**을 많이 사용하거나 쿼리 복잡성이 높다면 **QueryDSL**을 진지하게 고려하세요.

### H-1. 세팅(요약; Gradle Kotlin DSL)
```kotlin
dependencies {
  implementation("com.querydsl:querydsl-jpa:5.1.0:jakarta")
  annotationProcessor("com.querydsl:querydsl-apt:5.1.0:jakarta")
  annotationProcessor("jakarta.persistence:jakarta.persistence-api")
}
```
- 빌드 시 **Q타입**(예: `QMember`) 생성: `build/generated` 하위.

### H-2. 기본 사용
```java
@Repository
@RequiredArgsConstructor
public class MemberQueryDslRepository {
  private final JPAQueryFactory queryFactory; // @Bean으로 주입

  public List<Member> search(Boolean active, String q, String teamName, int page, int size) {
    QMember m = QMember.member;
    QTeam t = QTeam.team;

    BooleanBuilder where = new BooleanBuilder();
    if (active != null) where.and(m.active.eq(active));
    if (q != null && !q.isBlank()) where.and(m.username.containsIgnoreCase(q));
    if (teamName != null) where.and(m.team.name.eq(teamName));

    return queryFactory.selectFrom(m)
        .leftJoin(m.team, t).fetchJoin()
        .where(where)
        .offset((long) page * size)
        .limit(size)
        .orderBy(m.createdAt.desc(), m.id.asc())
        .fetch();
  }
}
```
- **장점**: 컴파일 타임 안전, 동적 조립 간결, DTO 프로젝션(`Projections.constructor/fields/bean`) 쉬움.
- **주의**: `fetchJoin` + 페이징 시 DB 결과 왜곡 가능 → **두 번 조회**(id만 페이징 → IN 조회) 패턴 사용.

### H-3. DTO 프로젝션
```java
public record MemberView(Long id, String username, String teamName) {}

public List<MemberView> viewsByTeam(Long teamId, int page, int size) {
  QMember m = QMember.member;
  QTeam t = QTeam.team;

  return queryFactory
      .select(Projections.constructor(MemberView.class, m.id, m.username, t.name))
      .from(m).join(m.team, t)
      .where(t.id.eq(teamId))
      .offset((long) page * size).limit(size)
      .orderBy(m.createdAt.desc())
      .fetch();
}
```

### H-4. 카운트 최적화
```java
JPAQuery<Long> countQuery = queryFactory
    .select(m.id.count())
    .from(m)
    .where(where);

// Page<T> 반환 헬퍼
return PageableExecutionUtils.getPage(content, pageable, countQuery::fetchOne);
```
- 콘텐츠가 마지막 페이지일 때 **불필요한 카운트 쿼리 생략**.

---

## I. 리포지토리 커스텀 구현(확장 포인트)

### I-1. 커스텀 인터페이스 + Impl
```java
public interface MemberRepositoryCustom {
  List<MemberView> searchViews(Boolean active, String q, Pageable pageable);
}

public class MemberRepositoryImpl implements MemberRepositoryCustom {

  private final JPAQueryFactory queryFactory;

  public MemberRepositoryImpl(JPAQueryFactory queryFactory) { this.queryFactory = queryFactory; }

  @Override
  public List<MemberView> searchViews(Boolean active, String q, Pageable pageable) {
    QMember m = QMember.member;
    QTeam t = QTeam.team;
    BooleanBuilder where = new BooleanBuilder();
    if (active != null) where.and(m.active.eq(active));
    if (q != null && !q.isBlank()) where.and(m.username.containsIgnoreCase(q));

    return queryFactory
        .select(Projections.constructor(MemberView.class, m.id, m.username, t.name))
        .from(m).leftJoin(m.team, t)
        .where(where)
        .offset(pageable.getOffset()).limit(pageable.getPageSize())
        .orderBy(QuerydslUtils.toOrderSpec(pageable.getSort(), m))
        .fetch();
  }
}
```

```java
public interface MemberRepository
    extends JpaRepository<Member, Long>, MemberRepositoryCustom { }
```
- Spring Data는 `MemberRepositoryImpl`(접미사 **Impl**)을 자동 연결.

---

## J. 정렬 고급 & 안전 가이드

### J-1. Sort 유효성 검증
- 클라이언트 입력 정렬 컬럼 화이트리스트 검증:
```java
Set<String> ALLOWED = Set.of("createdAt","id","username");
Sort sanitizeSort(Sort sort) {
  return Sort.by(sort.stream()
    .filter(o -> ALLOWED.contains(o.getProperty()))
    .toList());
}
```

### J-2. 대소문자 무시 정렬
```java
Sort sort = Sort.by(Sort.Order.by("username").ignoreCase().ascending());
```

### J-3. 다국어 정렬(콜레이션)
- DB **collation**(한국어/영문) 또는 **ICU** 설정, 네이티브 `order by username collate ...` 고려.

---

## K. N+1과 페이징의 함정, 그리고 해결책

### K-1. 컬렉션 fetch join + 페이징 금지
- 하이버네이트는 내부에서 메모리 페이징 → 결과 왜곡/성능 악화.
- **해결**:  
  1) **두 번 조회**: id 목록만 페이징 → IN 쿼리로 연관 로딩  
  2) 컬렉션은 배치 페치(`hibernate.default_batch_fetch_size`)로 초기화

```java
// 1) ids 페이징
@Query("select o.id from Order o where o.member.id=:mid")
Page<Long> pageIds(@Param("mid") Long mid, Pageable p);

// 2) 본 조회
@Query("select distinct o from Order o join fetch o.member where o.id in :ids")
List<Order> findWithMemberByIdIn(@Param("ids") List<Long> ids);
```

---

## L. 트랜잭션 & 읽기 전용 최적화

### L-1. 서비스 계층에서 경계 설정
```java
@Service
@RequiredArgsConstructor
public class MemberReadService {
  private final MemberRepository repo;

  @Transactional(readOnly = true)
  public Page<MemberView> search(Boolean active, String q, Pageable pageable) { /* ... */ }
}
```
- 읽기 전용은 플러시 생략, 드라이버/DB 힌트(벤더 따라 다름).

### L-2. 벌크/배치 업데이트는 분리
- `@Modifying` JPQL 또는 QueryDSL `update()` 사용 후 **영속성 컨텍스트 클리어**.

---

## M. 테스트 전략 — Query Method, Spec, QueryDSL

### M-1. 슬라이스로 빠르게
```java
@DataJpaTest
class MemberSearchSpecTest {

  @Autowired MemberRepository repo;

  @Test
  void spec_search_basic() {
    Specification<Member> spec = Specification
        .where(MemberSpec.active(true))
        .and(MemberSpec.usernameContains("ali"));
    Page<Member> page = repo.findAll(spec, PageRequest.of(0, 10));
    assertThat(page.getContent()).allMatch(m -> m.isActive());
  }
}
```

### M-2. Testcontainers로 실제 동작 검증
- LIKE/정렬/인덱스 힌트/윈도우 함수 등 **DB 의존 케이스**는 컨테이너로 통합 테스트.

---

## N. 성능·운영 체크리스트

- [ ] **조회 API**는 **DTO/프로젝션**으로 필요한 컬럼만.  
- [ ] **모든 연관은 LAZY**에서 시작 → 케이스별 `fetch join`/EntityGraph/배치 페치.  
- [ ] **카운트 쿼리 최적화**(Page): 불필요 조인 제거, 마지막 페이지면 생략.  
- [ ] **정렬 컬럼 인덱스** 확보, 화이트리스트 검증.  
- [ ] **벌크 업데이트 후 1차 캐시 초기화**.  
- [ ] **두 번 조회** 패턴으로 컬렉션 + 페이징 처리.  
- [ ] **OSIV 비활성** 시 서비스에서 DTO 변환까지 끝내기.  
- [ ] 복잡한 동적 쿼리는 **Specification → QueryDSL 승격**.

---

## O. 흔한 함정과 트러블슈팅

1) **`InvalidDataAccessApiUsageException: Parameter with that name [x] did not exist`**  
   - `@Param` 이름/플레이스홀더 불일치. 메서드/쿼리 동기화.

2) **`QuerySyntaxException`**  
   - JPQL에서 엔티티/필드 이름 사용(테이블/컬럼 X). DTO 경로 정확히.

3) **`LazyInitializationException`**  
   - 컨트롤러에서 LAZY 접근. 서비스에서 **fetch join**하거나 DTO 변환 후 반환.

4) **느린 count 쿼리**  
   - 조인 제거한 `countQuery` 제공 또는 Slice 사용.

5) **정렬 인젝션 위험**  
   - 허용 컬럼 화이트리스트로 필터링.

6) **ManyToMany + 페이징 이상 동작**  
   - 조인 엔티티로 리팩터링.

---

## P. 미니 레퍼런스(치트시트)

### P-1. 리포지토리 리턴 타입
- 단건: `Optional<T>`/`T`/`@Nullable T`  
- 다건: `List<T>`/`Page<T>`/`Slice<T>`/`Stream<T>`  
- 프로젝션: 인터페이스/레코드/DTO

### P-2. 키워드 스니펫
```
findBy, readBy, getBy, countBy, existsBy
And, Or, Between, LessThan, GreaterThan, Like, In, Not, True, False, IsNull, StartingWith, Containing
OrderBy…Asc/Desc, Distinct
Top1, First10
```

### P-3. JPA 힌트/락
```java
@QueryHints(@QueryHint(name = "org.hibernate.readOnly", value = "true"))
@Lock(LockModeType.PESSIMISTIC_WRITE)
```

---

## Q. 한 페이지 요약

- **Query Method**로 단순 조건은 끝내고, 복잡해지면 **`@Query`** 또는 **QueryDSL**로 승격.  
- **페이징/정렬**은 인덱스·count 최적화·화이트리스트 검증이 관건.  
- **동적 검색**은 간단하면 **Example/Matcher**, 복잡하면 **Specification** → 더 복잡하면 **QueryDSL**.  
- **N+1**은 LAZY + fetch join/배치 페치/DTO 프로젝션으로 차단.  
- **운영**에서는 카운트 비용·정렬 안정성·벌크 작업 이후 컨텍스트 초기화가 핵심.
