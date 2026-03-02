---
layout: post
title: PowerBuilder - Commit, Rollback 및 트랜잭션 관리와 동적 SQL
date: 2025-12-13 21:30:23 +0900
category: PowerBuilder
---
# Commit / Rollback 및 트랜잭션 관리와 동적 SQL

데이터베이스 애플리케이션에서 **트랜잭션 관리**는 데이터 무결성을 보장하는 핵심 요소입니다. PowerBuilder는 명시적인 `COMMIT`과 `ROLLBACK`을 통해 트랜잭션을 제어하며, 경우에 따라 **동적 SQL**을 사용하여 실행 시점에 SQL 문을 구성하고 실행해야 할 필요가 있습니다. 이 글에서는 트랜잭션의 개념부터 실제 관리 방법, 그리고 동적 SQL의 네 가지 형식과 활용법까지 상세히 알아보겠습니다.

---

## 1. Commit / Rollback 및 트랜잭션 관리

### 트랜잭션(Transaction)이란?

트랜잭션은 **하나의 논리적 작업 단위**를 구성하는 데이터베이스 연산들의 집합입니다. 트랜잭션은 **ACID** 특성을 가져야 합니다.

- **Atomicity (원자성)**: 모두 성공하거나 모두 실패해야 함
- **Consistency (일관성)**: 트랜잭션 전후 데이터베이스가 일관된 상태여야 함
- **Isolation (고립성)**: 동시에 실행되는 트랜잭션 간 간섭이 없어야 함
- **Durability (지속성)**: 성공한 트랜잭션은 영구적으로 반영되어야 함

### COMMIT과 ROLLBACK

PowerBuilder에서 트랜잭션은 다음 두 명령어로 제어합니다.

| 명령어 | 설명 |
|--------|------|
| **COMMIT** | 현재 트랜잭션의 모든 변경사항을 데이터베이스에 영구히 저장하고 트랜잭션을 종료합니다. |
| **ROLLBACK** | 현재 트랜잭션의 모든 변경사항을 취소하고 트랜잭션을 종료합니다. |

**기본 문법:**
```powerbuilder
COMMIT USING 트랜잭션객체;
ROLLBACK USING 트랜잭션객체;
```

트랜잭션 객체를 생략하면 기본적으로 SQLCA를 사용합니다.

```powerbuilder
COMMIT;           // SQLCA 사용
COMMIT USING SQLCA;  // 명시적 표현
```

### AutoCommit 속성

트랜잭션 객체의 **AutoCommit** 속성은 각 SQL 문이 자동으로 커밋될지 여부를 결정합니다.

- **FALSE (기본값)**: 수동 커밋 모드. 명시적인 COMMIT 또는 ROLLBACK이 필요합니다.
- **TRUE**: 자동 커밋 모드. 각 SQL 문 실행 직후 자동으로 COMMIT됩니다.

```powerbuilder
// 자동 커밋 모드 설정
SQLCA.AutoCommit = TRUE
EXECUTE IMMEDIATE "DELETE FROM temp_table" USING SQLCA;
// 별도의 COMMIT 없이 즉시 반영됨

// 수동 커밋 모드로 변경
SQLCA.AutoCommit = FALSE
```

**주의**: DataWindow를 사용할 때는 일반적으로 AutoCommit = FALSE로 설정하고, `Update()` 후 성공 시 COMMIT, 실패 시 ROLLBACK하는 패턴을 사용합니다.

### 트랜잭션 경계 설정

트랜잭션은 일반적으로 다음과 같은 패턴으로 관리됩니다.

```powerbuilder
// 트랜잭션 시작 (첫 SQL 문 실행 시 자동 시작)
// 여러 작업 수행
...
IF 성공 THEN
    COMMIT USING SQLCA;
ELSE
    ROLLBACK USING SQLCA;
END IF
```

### DataWindow Update와 트랜잭션

DataWindow의 `Update()` 함수는 내부적으로 여러 개의 INSERT, UPDATE, DELETE 문을 실행할 수 있습니다. 이 모든 작업은 하나의 트랜잭션으로 처리되어야 합니다.

```powerbuilder
// DataWindow 저장 버튼 Clicked 이벤트
IF dw_1.Update() = 1 THEN
    COMMIT USING SQLCA;
    MessageBox("성공", "저장되었습니다.")
ELSE
    ROLLBACK USING SQLCA;
    MessageBox("오류", "저장 실패: " + SQLCA.SQLErrText)
END IF
```

### Savepoint를 이용한 부분 롤백

복잡한 트랜잭션에서는 중간에 **저장점(Savepoint)**을 설정하여 해당 지점까지만 롤백할 수 있습니다.

```powerbuilder
// 저장점 설정
EXECUTE IMMEDIATE "SAVEPOINT sp_point1" USING SQLCA;

// 여러 작업 수행
...
IF 오류_발생 THEN
    // 저장점까지만 롤백
    EXECUTE IMMEDIATE "ROLLBACK TO SAVEPOINT sp_point1" USING SQLCA;
ELSE
    COMMIT USING SQLCA;
END IF
```

**주의**: SAVEPOINT는 모든 DBMS에서 지원하는 것은 아니며, Oracle, PostgreSQL 등에서 지원합니다. SQL Server는 `SAVE TRANSACTION`을 사용합니다.

### 실전 예제: 다중 테이블 업데이트

여러 테이블을 동시에 업데이트해야 하는 경우, 모든 작업이 성공하거나 모두 실패해야 합니다.

```powerbuilder
// 주문 테이블과 재고 테이블 동시 업데이트
Long ll_rtn

// 주문 테이블에 데이터 삽입 (dw_order에 새 행 추가)
dw_order.SetItem(1, "order_date", Today())
dw_order.SetItem(1, "cust_id", "C001")
dw_order.SetItem(1, "amount", 100000)

// 재고 테이블 업데이트 (dw_stock에서 수량 감소)
dw_stock.SetItem(1, "stock_qty", dw_stock.GetItemDecimal(1, "stock_qty") - 10)

// 두 DataWindow를 순서대로 업데이트 (하나의 트랜잭션)
IF dw_order.Update() = 1 AND dw_stock.Update() = 1 THEN
    COMMIT USING SQLCA;
    MessageBox("성공", "주문이 처리되었습니다.")
ELSE
    ROLLBACK USING SQLCA;
    MessageBox("오류", "처리 중 오류가 발생했습니다.")
END IF
```

---

## 2. 동적 SQL (Dynamic SQL)

동적 SQL은 **실행 시점에 SQL 문을 문자열로 구성하여 실행**하는 기법입니다. 다음과 같은 상황에서 유용합니다.

- 사용자 입력에 따라 검색 조건이 동적으로 변하는 경우
- 테이블 이름이나 컬럼 이름이 실행 시점에 결정되는 경우
- 데이터베이스 객체(테이블, 인덱스 등)를 동적으로 생성/삭제하는 DDL 문 실행
- PowerBuilder가 직접 지원하지 않는 DBMS 고유 SQL 문 실행

### 동적 SQL의 네 가지 형식

PowerBuilder는 동적 SQL을 위한 네 가지 형식을 제공합니다. 각 형식은 SQL 문의 복잡성과 결과 집합의 유무에 따라 선택합니다.

| 형식 | 설명 | 사용 시기 |
|------|------|----------|
| **Format 1** | 커서 없음, 입력/출력 파라미터 없음 | DDL 문, 단순 DML (행 반환 없음) |
| **Format 2** | 커서 없음, 입력 파라미터 있음 | 파라미터가 있는 DML (INSERT, UPDATE, DELETE) |
| **Format 3** | 커서 사용, 입력 파라미터 있음 | 다중 행 결과 집합 조회 (SELECT) |
| **Format 4** | 동적 커서, 입력/출력 파라미터 있음 | 컬럼 목록을 모를 때 (가장 유연) |

### Format 1: EXECUTE IMMEDIATE

입력 파라미터도 없고 결과 집합도 없는 SQL 문을 실행합니다. 주로 DDL(CREATE, DROP, ALTER)이나 단순 DML에 사용합니다.

**문법:**
```powerbuilder
EXECUTE IMMEDIATE SQL문장 {USING 트랜잭션객체};
```

**예제:**
```powerbuilder
// 테이블 생성
EXECUTE IMMEDIATE "CREATE TABLE temp (id INT, name VARCHAR(20))" USING SQLCA;
IF SQLCA.SQLCode <> 0 THEN
    MessageBox("오류", SQLCA.SQLErrText)
END IF

// 데이터 삭제
EXECUTE IMMEDIATE "DELETE FROM temp WHERE id = 100" USING SQLCA;
```

### Format 2: PREPARE + EXECUTE (DML 전용)

입력 파라미터가 있는 INSERT, UPDATE, DELETE 문을 실행할 때 사용합니다. 결과 집합을 반환하지 않습니다.

**문법:**
```powerbuilder
PREPARE SQLSA FROM SQL문장 {USING 트랜잭션객체};
EXECUTE SQLSA USING {파라미터변수목록};
```

**예제:**
```powerbuilder
String ls_sql
ls_sql = "UPDATE employee SET salary = salary * ? WHERE dept_id = ?"

PREPARE SQLSA FROM ls_sql USING SQLCA;
EXECUTE SQLSA USING :ld_multiplier, :ls_dept_id;

IF SQLCA.SQLCode = 0 THEN
    COMMIT USING SQLCA;
ELSE
    ROLLBACK USING SQLCA;
END IF
```

> **⚠️ 중요**: Format 2는 SELECT 문에 사용할 수 없습니다. SELECT 문을 동적으로 실행하려면 반드시 Format 3(커서)를 사용해야 합니다.

### Format 3: 커서를 이용한 다중 행 조회

여러 행을 반환하는 동적 SELECT 문을 처리할 때 사용합니다. 단일 행만 반환하는 경우에도 커서를 사용해야 합니다.

**문법:**
```powerbuilder
DECLARE 커서이름 DYNAMIC CURSOR FOR SQLSA;
PREPARE SQLSA FROM SQL문장 {USING 트랜잭션객체};
OPEN DYNAMIC 커서이름 {USING 파라미터변수목록};
FETCH 커서이름 INTO 변수목록;
CLOSE 커서이름;
```

**예제: 부서별 사원 조회**
```powerbuilder
String ls_sql, ls_dept_id
Integer li_id
String ls_name
Double ld_salary

// 1. 커서 선언 (스크립트 상단, 변수 선언부에 위치)
DECLARE emp_cur DYNAMIC CURSOR FOR SQLSA;

ls_dept_id = "SALES"
ls_sql = "SELECT emp_id, name, salary FROM employee WHERE dept_id = ?"

// 2. SQL 준비
PREPARE SQLSA FROM ls_sql; // USING SQLCA는 생략 가능 (기본값)

// 3. 커서 열기 (파라미터 전달)
OPEN DYNAMIC emp_cur USING :ls_dept_id;

// 4. 데이터 페치 루프
DO WHILE True
    FETCH emp_cur INTO :li_id, :ls_name, :ld_salary;
    // FETCH 결과가 0이 아니면 (100: 데이터 없음, -1: 에러) 루프 종료
    IF SQLCA.SQLCode <> 0 THEN EXIT;
    
    // 화면에 표시하거나 처리
    lb_list.AddItem(String(li_id) + " : " + ls_name + " : " + String(ld_salary))
LOOP

// 5. 커서 닫기
CLOSE emp_cur;
```

### Format 4: 동적 커서 (컬럼 목록을 모를 때)

가장 유연한 형식으로, 실행 시점에 컬럼의 개수와 타입이 결정됩니다. `DESCRIBE`를 사용하여 결과 집합의 구조를 파악한 후 처리합니다.

**문법:**
```powerbuilder
DECLARE 커서이름 DYNAMIC CURSOR FOR SQLSA;
PREPARE SQLSA FROM SQL문장 {USING 트랜잭션객체};
DESCRIBE SQLSA INTO DynamicDescriptionArea;
OPEN DYNAMIC 커서이름 USING DESCRIPTOR DynamicDescriptionArea;
FETCH 커서이름 USING DESCRIPTOR DynamicDescriptionArea;
CLOSE 커서이름;
```

**DynamicDescriptionArea** 객체를 통해 컬럼의 수, 타입, 값을 동적으로 가져올 수 있습니다.

**예제: 어떤 테이블이든 동적으로 조회**
```powerbuilder
String ls_tbl, ls_sql
ls_tbl = "employee"   // 사용자 입력으로 받았다고 가정
ls_sql = "SELECT * FROM " + ls_tbl

DECLARE dyn_cur DYNAMIC CURSOR FOR SQLSA;
PREPARE SQLSA FROM ls_sql USING SQLCA;

// 결과 집합 구조 파악
DynamicDescriptionArea lda_dyn
DESCRIBE SQLSA INTO lda_dyn;

// 커서 열기
OPEN DYNAMIC dyn_cur USING DESCRIPTOR lda_dyn;

// 컬럼 수 확인
Integer li_col_cnt = lda_dyn.NumOutputs
String ls_col_value

// 데이터 페치 (각 컬럼의 값을 동적으로 읽기)
FETCH dyn_cur USING DESCRIPTOR lda_dyn;
DO WHILE SQLCA.SQLCode = 0
    FOR i = 1 TO li_col_cnt
        CHOOSE CASE lda_dyn.OutParmType[i]
            CASE TypeString!
                ls_col_value = lda_dyn.GetDynamicString(i)
            CASE TypeInteger!
                ls_col_value = String(lda_dyn.GetDynamicNumber(i))
            // ... 기타 타입 처리
        END CHOOSE
        // 값 처리 (예: 리스트박스에 추가)
    NEXT
    FETCH dyn_cur USING DESCRIPTOR lda_dyn;
LOOP

CLOSE dyn_cur;
```

**참고**: Format 4는 복잡하고 성능 부담이 크므로, 가능하면 Format 2나 3을 사용하는 것이 좋습니다.

### 동적 SQL과 SQL Injection

동적 SQL 사용 시 가장 주의해야 할 점은 **SQL 인젝션 공격**입니다. 사용자 입력을 직접 SQL 문자열에 연결하면 보안 위험이 발생할 수 있습니다.

**취약한 코드:**
```powerbuilder
// 사용자 입력을 직접 연결 (위험!)
String ls_id = sle_id.Text
String ls_sql = "SELECT * FROM users WHERE id = '" + ls_id + "'"
EXECUTE IMMEDIATE ls_sql;   // 위험!
```

**안전한 코드 (파라미터 사용):**
```powerbuilder
String ls_sql = "SELECT * FROM users WHERE id = ?"
PREPARE SQLSA FROM ls_sql USING SQLCA;
EXECUTE SQLSA USING :ls_id;
```

항상 **파라미터화된 쿼리**를 사용하여 SQL 인젝션을 방지해야 합니다.

> **💡 실무 팁: 동적 SQL의 남용을 피하라!**  
> 파워빌더에서는 복잡한 동적 SELECT 문을 스크립트로 길게 작성하는 것보다, **숨겨진 DataStore(비시각적 DataWindow)**를 하나 만들고 `SetSQLSelect()` 함수를 이용해 런타임에 DataWindow의 쿼리를 동적으로 변경하여 `Retrieve()` 하는 방식이 훨씬 안전하고 코드가 간결합니다. 동적 SQL은 DataWindow로 처리하기 까다로운 동적 DDL이나 복잡한 배치 작업(주로 Format 1, 2)에 제한적으로 사용하는 것이 좋습니다.

---

## 3. 트랜잭션과 동적 SQL 결합 실전 예제

다음은 동적 SQL을 사용하여 여러 테이블을 업데이트하면서 트랜잭션을 관리하는 예제입니다.

### 시나리오: 동적 테이블에 데이터 삽입

사용자가 입력한 테이블명과 데이터를 받아 해당 테이블에 레코드를 삽입하는 기능입니다.

```powerbuilder
// 윈도우 함수: 동적 테이블에 데이터 삽입
FUNCTION Boolean uf_insert_dynamic(String as_table, String as_id, String as_name)
    String ls_sql
    Integer li_rtn
    
    // 트랜잭션 시작 (AutoCommit = FALSE 가정)
    ls_sql = "INSERT INTO " + as_table + " (id, name) VALUES (?, ?)"
    
    // 동적 SQL 준비 및 실행
    PREPARE SQLSA FROM ls_sql USING SQLCA;
    EXECUTE SQLSA USING :as_id, :as_name;
    
    IF SQLCA.SQLCode = 0 THEN
        COMMIT USING SQLCA;
        RETURN TRUE
    ELSE
        ROLLBACK USING SQLCA;
        MessageBox("오류", SQLCA.SQLErrText)
        RETURN FALSE
    END IF
END FUNCTION
```

**주의**: 위 코드는 테이블명을 직접 연결하므로, 테이블명은 사전에 허용된 목록에서 선택하도록 제한하는 것이 안전합니다.

---

## 결론

트랜잭션 관리는 데이터 무결성을 보장하는 핵심 요소로, PowerBuilder에서는 `COMMIT`과 `ROLLBACK`을 명시적으로 호출하여 트랜잭션 경계를 설정합니다. DataWindow 작업 후에는 반드시 `Update()` 결과에 따라 적절히 커밋 또는 롤백해야 합니다.

동적 SQL은 실행 시점에 SQL 문을 구성해야 하는 유연한 상황에서 필수적입니다. PowerBuilder의 네 가지 동적 SQL 형식을 이해하고, 각 상황에 맞게 선택하여 사용하면 됩니다. 특히 Format 2는 DML 전용이며, SELECT 문은 반드시 Format 3(커서)를 사용해야 합니다. 보안 측면에서 SQL 인젝션을 방지하기 위해 항상 파라미터화된 쿼리를 사용하는 습관을 들여야 합니다.

또한, 가능하다면 복잡한 동적 SELECT 문 대신 DataWindow의 `SetSQLSelect()` 등을 활용하는 것이 더 간결하고 유지보수하기 쉽습니다. 동적 SQL은 필요한 곳에만 적절히 사용하는 것이 좋습니다.

트랜잭션 관리와 동적 SQL은 함께 사용될 수 있으며, 복잡한 비즈니스 로직을 구현할 때 큰 힘을 발휘합니다. 이 두 개념을 마스터하면 PowerBuilder 애플리케이션의 데이터 처리 능력을 한 단계 높일 수 있습니다.

다음 글에서는 **DataWindow의 고급 기능**인 동적 DataWindow, 데이터 공유(ShareData), 그리고 다중 행 처리 기법에 대해 자세히 알아보겠습니다.