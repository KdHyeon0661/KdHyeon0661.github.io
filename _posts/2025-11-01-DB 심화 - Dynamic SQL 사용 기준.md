---
layout: post
title: DB 심화 - Dynamic SQL 사용 기준
date: 2025-11-01 14:25:23 +0900
category: DB 심화
---
# Dynamic SQL 사용 기준

> **핵심 원칙**
> - **값에 대한 동적 처리는 바인드 변수로 해결**하고, **구조 변경이 필요한 경우에만 Dynamic SQL을 사용**한다.
> - 선택적 검색 조건 처리는 **성능, 가독성, 안전성**을 종합적으로 고려하여 **정적 SQL의 복잡성과 동적 SQL의 유연성 사이에서 최적의 균형점을 찾아야 한다**.

---

## 정의와 범위

- **정적 SQL(Static SQL)**: 애플리케이션 코드 내에 SQL 문장이 완전히 고정되어 있는 형태. 컴파일 시점에 구문 검증이 가능하다.
- **동적 SQL(Dynamic SQL)**: 런타임에 문자열을 조합하여 SQL 문장을 생성하고 실행하는 형태. `EXECUTE IMMEDIATE` 또는 애플리케이션 레벨의 문자열 템플릿을 사용한다.
- **선택적 검색 조건**: 사용자의 입력값(파라미터)이 NULL일 경우 해당 조건을 필터에서 제외하고, 값이 있을 때만 적용하는 쿼리 패턴이다. 예: `region`, `date_from`, `date_to`, `status` 등.

> **핵심 문제**: Static SQL과 Dynamic SQL의 선택 문제는 근본적으로 **바인드 변수의 적절한 사용 여부**와 직결된다. 쿼리의 성능과 보안을 위해 값은 반드시 바인드 변수로 처리해야 한다.

---

## Dynamic SQL 사용의 기본 원칙

### 원칙 1 — 값은 바인드 변수를 사용한다
런타임에 변화하는 값(파라미터)을 SQL 문자열에 직접 결합하지 않는다. 이는 SQL Injection 위험을 초래하고, 커서 공유를 방해하여 성능을 저하시킨다. 모든 값은 바인드 변수(`USING` 절)를 통해 전달해야 한다.

### 원칙 2 — 구조 변경 시에만 동적 SQL을 고려한다
테이블명, 컬럼명, 조인 조건, 정렬 기준, 힌트, 파티션 프루닝 조건 등 SQL의 구조적 요소가 런타임에 결정될 때 Dynamic SQL 사용을 검토한다. 이때도 사용자 입력값을 직접 사용하지 않고, 반드시 사전 정의된 화이트리스트를 통해 검증해야 한다.

### 원칙 3 — 인덱스 사용 가능성(SARGability)을 보장한다
`WHERE` 절에 `TRUNC(column)`, `NVL(column, ...)` 같은 컬럼 변형 함수를 사용하거나, 비상수 값과의 비교를 하면 인덱스 사용이 제한된다. 가능하면 컬럼 자체를 그대로 비교하는 조건(`col >= :d1 AND col < :d2`)으로 쿼리를 설계해야 한다.

### 원칙 4 — 실행 계획의 안정성을 최우선한다
데이터 분포가 치우친 스큐 컬럼에는 히스토그램 통계와 Adaptive Cursor Sharing(ACS)을 활용하여 카디널리티 추정의 정확도를 높인다. 핵심 비즈니스 로직의 SQL은 SQL Plan Baseline(SPM)을 통해 최적의 실행 계획을 고정시켜 예상치 못한 계획 변경을 방지한다.

### 원칙 5 — 가독성과 유지보수성을 고려한다
조건이 많아질수록 동적 SQL 문자열 조합 로직은 복잡해지고 디버깅이 어려워진다. 경우에 따라 `UNION ALL`을 이용한 명시적인 분기 방식이 더 직관적이고 테스트하기 쉬울 수 있다. 작성한 SQL은 단위 테스트 케이스를 쉽게 구성할 수 있어야 한다.

---

## 선택적 검색 조건 처리: 현실적인 도전과 해결 패턴

사용자 입력 파라미터에 따라 조건이 유동적으로 변하는 쿼리는 흔하지만, 성능 문제를 일으키기 쉽다. 단순한 `OR :p IS NULL` 패턴은 편리하지만 인덱스 사용을 방해하고 실행 계획을 불안정하게 만든다.

```sql
-- 흔한 안티패턴: 간단하지만 성능 위험이 있음
WHERE (region = :r OR :r IS NULL)
  AND (order_dt >= :d1 OR :d1 IS NULL)
  AND (order_dt <  :d2 OR :d2 IS NULL)
```

아래는 다양한 상황에 맞춰 선택할 수 있는 설계 패턴이다. 데이터 규모, 선택도, 인덱스 구성, 유지보수 비용을 종합적으로 평가하여 선택해야 한다.

### 패턴 1: 정적 `OR + IS NULL` (단순 유지)
정적 SQL을 유지하면서 선택적 조건을 처리하는 가장 기본적인 방법이다.
- **장점**: 구현이 매우 간단하며, 커서 공유가 쉽다.
- **단점**: 옵티마이저가 최적의 실행 계획을 수립하기 어려워 인덱스를 효율적으로 사용하지 못할 수 있다. 대량 데이터 처리 시 성능 저하 가능성이 높다.
- **적절한 사용처**: 데이터량이 적거나, 임시 검증용(PoC) 쿼리, 성능이 중요하지 않은 보조적인 조회 기능.

### 패턴 2: `UNION ALL` 분기 (명시적 계획 분리)
경우의 수를 미리 정의하고, 각 경우에 최적화된 정적 SQL을 `UNION ALL`로 결합한다.
```sql
-- region 필터만 있는 경우
SELECT ... FROM t WHERE region = :r AND :d1 IS NULL
UNION ALL
-- region과 기간 필터가 모두 있는 경우
SELECT ... FROM t WHERE region = :r AND order_dt >= :d1 AND order_dt < :d2
UNION ALL
-- 필터가 전혀 없는 경우
SELECT ... FROM t WHERE :r IS NULL AND :d1 IS NULL
```
- **장점**: 각 분기별로 독립적이고 최적의 실행 계획을 가질 수 있어 성능이 안정적이다.
- **단점**: 조건의 조합이 많아지면 SQL 문장이 기하급수적으로 늘어나 관리가 복잡해진다.
- **적절한 사용처**: 핵심 트랜잭션 경로의 쿼리, 조건별 데이터 선택도 차이가 극명하여 다른 실행 계획이 필요한 경우.

### 패턴 3: 동적 WHERE 절 조립 (권장 패턴)
실행 시점에 실제로 필요한 조건만으로 WHERE 절을 동적으로 구성한다. **값은 반드시 바인드 변수로 전달한다.**
```plsql
v_sql := 'SELECT ... FROM t WHERE 1=1';
IF v_region IS NOT NULL THEN
    v_sql := v_sql || ' AND region = :reg';
END IF;
IF v_date_from IS NOT NULL THEN
    v_sql := v_sql || ' AND order_dt >= :dt_from';
END IF;
-- EXECUTE IMMEDIATE v_sql USING ...;
```
- **장점**: 불필요한 조건이 제거되어 인덱스를 최대한 활용할 수 있다. 패턴 2에 비해 관리해야 할 SQL 템플릿이 하나로 집중된다.
- **단점**: 동적 SQL 실행을 위한 코드가 필요하며, 모든 파라미터 조합에 대한 테스트가 필요하다.
- **적절한 사용처**: 대용량 테이블의 핵심 조회, 선택도가 높거나 스큐가 심한 컬럼을 포함하는 필터 조건.

### 패턴 4: 배열 바인드 (다중 값 필터)
하나의 파라미터로 여러 값을 선택해야 하는 경우(예: 여러 지역) 사용한다.
```plsql
TYPE t_vc IS TABLE OF VARCHAR2(10);
v_regions t_vc := t_vc('APAC', 'EMEA');
...
SELECT ... FROM t
WHERE region IN (SELECT COLUMN_VALUE FROM TABLE(:v_regions));
```
- **장점**: 애플리케이션에서 컬렉션을 바인드하여 안전하게 다중 값 IN 조건을 처리할 수 있다.
- **단점**: 바인드 배열의 크기가 수백~수천 개를 넘어서면 성능이 저하될 수 있다.
- **적절한 사용처**: UI의 멀티 선택 필터, 사용자 권한에 따른 데이터 접근 제어 목록.

### 패턴 5: 전역 임시 테이블(GTT) 조인 (대량 키 필터)
필터링할 키 값이 매우 많을 때, 해당 키들을 먼저 임시 테이블에 저장하고 메인 쿼리와 조인한다.
```sql
-- 1. 세션별 임시 테이블에 키 적재
INSERT INTO my_gtt (filter_key) VALUES (?);
-- 2. 메인 쿼리에서 조인
SELECT t.* FROM main_table t JOIN my_gtt g ON t.key = g.filter_key;
```
- **장점**: 수만, 수십만 개의 키에 대한 필터링을 효율적으로 처리할 수 있다. 임시 테이블에 인덱스를 생성할 수도 있다.
- **단점**: 임시 테이블 관리 오버헤드와 추가적인 디스크 I/O가 발생한다.
- **적절한 사용처**: 대규모 배치 작업, 매우 많은 값을 기준으로 하는 리포트 조회.

### 패턴 6: JSON 파라미터 활용 (유연한 인터페이스)
많은 수의 선택적 파라미터를 JSON 형식 하나로 받아서 처리한다.
```sql
SELECT ...
FROM t
WHERE ( :j_param.region IS NULL OR region = JSON_VALUE(:j_param, '$.region') )
  AND ( :j_param.date_from IS NULL OR order_dt >= JSON_VALUE(:j_param, '$.date_from') );
```
- **장점**: 인터페이스가 단순해지고, 파라미터 개수 변경에 유연하게 대응할 수 있다.
- **단점**: JSON 함수 사용으로 인해 인덱스 활용이 제한될 수 있으며, 가상 컬럼이나 함수 기반 인덱스 등 추가 최적화가 필요할 수 있다.
- **적절한 사용처**: 파라미터 구조가 자주 변경될 수 있는 유연한 API 백엔드.

---

## 언어별 안전한 구현 예시

### Java (JDBC)
```java
StringBuilder sql = new StringBuilder("SELECT * FROM emp WHERE 1=1");
List<Object> params = new ArrayList<>();

if (region != null) {
    sql.append(" AND region = ?");
    params.add(region);
}
if (startDate != null) {
    sql.append(" AND hiredate >= ?");
    params.add(startDate);
}

try (PreparedStatement pstmt = connection.prepareStatement(sql.toString())) {
    for (int i = 0; i < params.size(); i++) {
        pstmt.setObject(i + 1, params.get(i));
    }
    ResultSet rs = pstmt.executeQuery();
    // 결과 처리
}
```

### Python (oracledb)
```python
sql_parts = ["SELECT * FROM emp WHERE 1=1"]
params = {}
if region is not None:
    sql_parts.append(" AND region = :region")
    params["region"] = region
if start_date is not None:
    sql_parts.append(" AND hiredate >= :start_date")
    params["start_date"] = start_date

final_sql = " ".join(sql_parts)
cursor.execute(final_sql, params)
```

---

## 성능 비교 접근법

이론적인 장단점 외에 실제 환경에서의 성능을 비교 평가해야 한다. 다음 항목을 체크한다.

1.  **실행 계획 분석**: `DBMS_XPLAN.DISPLAY_CURSOR`를 사용하여 실제 실행 계획을 확인한다. `Access Predicates`(인덱스를 통해 필터링)가 적절하게 사용되는지, `Filter Predicates`(테이블 접근 후 필터링)가 많지는 않은지 본다.
2.  **리소스 사용량 측정**: `AUTOTRACE` 또는 성능 뷰를 통해 `Consistent Gets`(논리적 읽기), `Physical Reads`(물리적 읽기), `Elapsed Time`을 비교한다.
3.  **다양한 데이터 분포 테스트**:
    *   **선택도 높은 조건**: `region = 'RARE_VALUE'`
    *   **선택도 낮은 조건**: `region = 'COMMON_VALUE'`
    *   **NULL 파라미터**: 조건 자체가 적용되지 않는 경우
    *   **복합 조건**: 여러 조건이 결합된 경우

---

## 패턴 선택 가이드라인

| 상황 | 권장 패턴 | 이유 |
| :--- | :--- | :--- |
| 소량 데이터, 빠른 구현 필요 | 패턴 1 (정적 `OR`) | 단순성 우선, 성능 영향 미미 |
| 핵심 OLTP, 조건별 최적 계획 필요 | 패턴 2 (`UNION ALL`) | 실행 계획 안정성 최고 |
| 대용량 테이블, 다양한 조건 조합 | 패턴 3 (동적 WHERE) | 유연성과 성능의 밸런스 |
| UI 멀티 선택 필터 (값 수십 개) | 패턴 4 (배열 바인드) | 구현이 깔끔하고 안전함 |
| 배치 작업, 필터 키 수만 개 이상 | 패턴 5 (GTT 조인) | 대량 데이터 조인에 적합 |
| 파라미터 구조가 매우 유동적 | 패턴 6 (JSON) | 인터페이스 통합 용이 |

---

## 결론

Dynamic SQL 사용 여부는 단순한 기술 선호도의 문제가 아니다. **값은 무조건 바인드 변수로 처리한다**는 보안과 성능의 기본 원칙을 준수한 상태에서, **비즈니스 요구사항과 데이터 특성에 맞춰 가장 예측 가능하고 효율적인 실행 계획을 유도할 수 있는 설계를 선택**해야 한다.

선택적 검색 조건 처리에는 정답이 없다. 각 패턴은 Trade-off가 존재한다. 개발자는 자신의 시스템에서 가장 중요한 것이 무엇인지(절대적인 성능, 코드 유지보수성, 구현 속도, 실행 계획 안정성)를 명확히 하고, 그에 따라 `동적 WHERE 조립`이나 `UNION ALL 분기`와 같은 패턴을 적절히 선택하거나 혼용해야 한다. 모든 결정은 실제 데이터와 워크로드에 대한 성능 테스트를 근거로 해야 한다.