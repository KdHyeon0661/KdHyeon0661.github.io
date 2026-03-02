---
layout: post
title: PowerBuilder - Stored Procedure 호출 및 DB ErrorCode 처리
date: 2025-12-13 22:30:23 +0900
category: PowerBuilder
---
# Stored Procedure 호출 및 DB ErrorCode 처리

데이터베이스 기반 애플리케이션에서 **저장 프로시저(Stored Procedure)**는 성능 향상, 보안 강화, 비즈니스 로직 집중화 등의 장점으로 널리 사용됩니다. PowerBuilder는 저장 프로시저를 호출하는 다양한 방법을 제공하며, 동시에 발생할 수 있는 **데이터베이스 오류**를 체계적으로 처리할 수 있는 기능을 갖추고 있습니다. 특히 `DBError` 이벤트를 활용하면 예외 상황에서도 사용자에게 친숙한 메시지를 보여주고, 데이터 무결성을 유지할 수 있습니다. 이 글에서는 저장 프로시저 호출 방법과 DB 오류 처리 기법을 상세히 알아보겠습니다.

---

## 1. Stored Procedure 호출

### 1.1 저장 프로시저란?

저장 프로시저는 데이터베이스 서버에 저장된 미리 컴파일된 SQL 코드 블록입니다. 다음과 같은 장점이 있습니다.

- **성능 향상**: 최초 실행 시 컴파일되어 캐시에 저장되므로 반복 실행 시 빠름
- **네트워크 트래픽 감소**: 여러 SQL 문을 하나의 호출로 처리
- **보안 강화**: 테이블 직접 접근 대신 프로시저 권한만 부여
- **비즈니스 로직 중앙화**: 여러 애플리케이션에서 동일한 로직 공유

### 1.2 PowerBuilder에서 저장 프로시저 호출 방법

PowerBuilder는 저장 프로시저 호출을 위해 세 가지 주요 방법을 제공합니다.

#### 1.2.1 DECLARE와 EXECUTE를 이용한 호출 (표준 방식)

가장 기본적인 방법으로, 먼저 프로시저를 선언(DECLARE)한 후 실행(EXECUTE)합니다. 이 방식은 파라미터를 명시적으로 바인딩할 수 있고, 출력 파라미터를 받을 수 있습니다.

**문법**:
```powerbuilder
DECLARE 프로시저명 PROCEDURE FOR 저장프로시저명
    ( :변수1, :변수2, ... )   {USING 트랜잭션객체};

EXECUTE 프로시저명;

// 상태 확인
IF SQLCA.SQLCode = 0 THEN
    COMMIT USING SQLCA;
ELSE
    ROLLBACK USING SQLCA;
END IF

CLOSE 프로시저명;   // 리소스 해제
```

**예제**: 부서별 급여 총액을 계산하는 프로시저 호출 (출력 파라미터 포함)

```powerbuilder
// 프로시저: calculate_dept_total (IN dept_id, OUT total)
Long ll_dept_id = 100
Double ld_total

// 1. 프로시저 선언 (파라미터는 순서대로 나열, 출력은 OUTPUT 키워드)
DECLARE dept_total PROCEDURE FOR calculate_dept_total
    (:ll_dept_id, :ld_total OUTPUT)
    USING SQLCA;

// 2. 실행
EXECUTE dept_total;

// 3. 결과 확인
IF SQLCA.SQLCode = 0 THEN
    MessageBox("부서 총액", "총액: " + String(ld_total))
ELSE
    MessageBox("오류", SQLCA.SQLErrText)
END IF

// 4. 리소스 해제
CLOSE dept_total;
```

출력 파라미터는 `OUTPUT` 키워드로 표시하며, 프로시저 실행 후 변수에 결과가 저장됩니다.

#### 1.2.2 DataWindow에서 저장 프로시저를 데이터 소스로 사용

DataWindow 생성 시 데이터 소스로 **Stored Procedure**를 선택할 수 있습니다. 이 방법을 사용하면 프로시저의 결과 집합을 DataWindow가 자동으로 처리합니다.

**DataWindow 생성 단계**:
1. New → DataWindow → 원하는 프레젠테이션 스타일 선택
2. 데이터 소스 선택 화면에서 **Stored Procedure** 선택
3. 프로시저 목록에서 원하는 프로시저 선택
4. 필요한 경우 파라미터 정의

**예제**: 고객 목록을 반환하는 프로시저로 DataWindow 생성 후 조회

```powerbuilder
// 프로시저: get_customers (IN grade)
dw_customer.SetTransObject(SQLCA)
dw_customer.Retrieve("GOLD")   // 'GOLD' 등급 고객 조회
```

DataWindow가 자동으로 프로시저를 호출하고 결과 집합을 표시합니다. 업데이트 기능을 사용하려면 프로시저가 업데이트 가능한 결과 집합을 반환해야 하며, Update Properties 설정이 필요합니다.

#### 1.2.3 RPCFUNC를 이용한 트랜잭션 객체 확장 (실무 표준 권장)

> **💡 실무 핵심 팁**: RPCFUNC 방식은 저장 프로시저를 마치 객체의 메서드처럼 호출할 수 있어 코드가 가장 간결하고 유지보수가 쉽습니다. 따라서 실무에서는 이 방식이 사실상 표준(Best Practice)으로 사용됩니다.

트랜잭션 객체에 **RPCFUNC** 키워드로 저장 프로시저를 외부 함수처럼 선언하면, 마치 객체의 메서드처럼 프로시저를 호출할 수 있습니다. DECLARE/EXECUTE 문법에 비해 코드가 훨씬 간결하고, 출력 파라미터 처리도 자연스럽습니다.

**단계**:
1. 트랜잭션 객체를 상속받는 사용자 객체(예: `u_trans_database`) 생성
2. 사용자 객체의 **Local External Functions** 영역에 프로시저 선언
3. 애플리케이션에서 SQLCA의 타입을 이 사용자 객체로 변경
4. 코드에서 `SQLCA.프로시저명()` 형태로 호출

**선언 예**:
```powerbuilder
// u_trans_database의 Local External Functions
FUNCTION double give_raise (REF double salary) RPCFUNC
FUNCTION int sp_save_customer (string id, string name, string phone, string grade, string mode) RPCFUNC
```

**사용 예**:
```powerbuilder
// Application Open 이벤트에서 SQLCA 타입 변경
// Application → Properties → Variable Types 탭 → SQLCA: u_trans_database

// 이후 코드에서
Double ld_sal = 50000
SQLCA.give_raise(ld_sal)   // 프로시저 호출, ld_sal은 출력 파라미터로 변경 가능

// 저장 프로시저 호출 (리턴값으로 성공 여부 확인 가능)
int li_rtn
li_rtn = SQLCA.sp_save_customer("C001", "홍길동", "02-1234", "G", "I")
IF li_rtn = 0 THEN
    COMMIT USING SQLCA;
ELSE
    ROLLBACK USING SQLCA;
END IF
```

이 방법은 프로시저를 내장 함수처럼 호출할 수 있어 코드가 간결하고 직관적입니다. 또한 반환값이나 출력 파라미터를 통해 프로시저의 실행 결과를 쉽게 받을 수 있습니다.

### 1.3 입력/출력 파라미터 처리

저장 프로시저의 파라미터는 방향에 따라 다음과 같이 처리합니다.

| 파라미터 종류 | PowerBuilder에서 표현 |
|--------------|----------------------|
| **IN** | 일반 변수를 값으로 전달 (DECLARE 시 변수만 나열) |
| **OUT** | `OUTPUT` 키워드로 표시 (DECLARE 시 변수에 OUTPUT) |
| **INOUT** | `OUTPUT` 키워드로 표시하고 초기값 설정 |

**예제**: INOUT 파라미터를 가진 프로시저 (DECLARE 방식)

```powerbuilder
// 프로시저: adjust_salary (INOUT emp_id, IN percentage)
Long ll_emp_id = 123
Double ld_percent = 10.0

DECLARE adjust_sal PROCEDURE FOR adjust_salary
    (:ll_emp_id OUTPUT, :ld_percent)
    USING SQLCA;

EXECUTE adjust_sal;

IF SQLCA.SQLCode = 0 THEN
    // ll_emp_id는 프로시저 내에서 변경된 값이 반영됨
    MessageBox("변경된 ID", String(ll_emp_id))
END IF

CLOSE adjust_sal;
```

### 1.4 결과 집합을 반환하는 프로시저 처리

결과 집합을 반환하는 프로시저는 주로 DataWindow를 통해 처리하는 것이 간편합니다. 그러나 스크립트에서 직접 처리할 수도 있습니다.

**예제**: 결과 집합을 커서로 처리 (DECLARE PROCEDURE 커서)

```powerbuilder
// 프로시저: get_employees (IN dept_id)
// 결과 집합: emp_id, emp_name, salary

Long ll_dept_id = 10

DECLARE emp_cur PROCEDURE FOR get_employees
    (:ll_dept_id)
    USING SQLCA;

// 프로시저 실행 (결과 집합을 반환하는 경우 OPEN으로 시작)
OPEN emp_cur;

Long ll_id
String ls_name
Double ld_salary

FETCH emp_cur INTO :ll_id, :ls_name, :ld_salary;
DO WHILE SQLCA.SQLCode = 0
    // 각 행 처리
    FETCH emp_cur INTO :ll_id, :ls_name, :ld_salary;
LOOP

CLOSE emp_cur;
```

---

## 2. DB ErrorCode 처리

데이터베이스 작업 중에는 다양한 오류가 발생할 수 있습니다. PowerBuilder는 오류 정보를 확인할 수 있는 속성과 이벤트를 제공합니다.

### 2.1 SQLCA 상태 속성 이해

트랜잭션 객체(SQLCA)는 최근 SQL 작업의 결과를 다음 속성에 저장합니다.

| 속성 | 데이터 타입 | 설명 |
|------|------------|------|
| **SQLCode** | Long | 0: 성공, -1: 오류, 100: 검색된 데이터 없음 |
| **SQLDBCode** | Long | DBMS 고유 오류 코드 |
| **SQLErrText** | String | 오류 메시지 |
| **SQLNRows** | Long | 영향을 받은 행 수 |

**기본 오류 처리 패턴**:
```powerbuilder
CONNECT USING SQLCA;
IF SQLCA.SQLCode <> 0 THEN
    MessageBox("DB 연결 오류", "코드: " + String(SQLCA.SQLDBCode) + "~r~n" + SQLCA.SQLErrText)
    HALT
END IF
```

### 2.2 DataWindow/DataStore의 DBError 이벤트

DataWindow나 DataStore에서 데이터베이스 작업 중 오류가 발생하면 **DBError** 이벤트가 발생합니다. 이 이벤트를 활용하면 표준 오류 메시지 대신 사용자 정의 처리를 할 수 있습니다.

**DBError 이벤트 파라미터**:

| 파라미터 | 설명 |
|---------|------|
| **sqldbcode** | DBMS 오류 코드 |
| **sqlerrtext** | 오류 메시지 |
| **sqlsyntax** | 실행하려던 SQL 문 |
| **buffer** | 오류 발생 시점의 버퍼 (Primary!, Delete!, Filter!, etc.) |
| **row** | 오류 발생 행 번호 |

**반환값**:
- **0** (기본값): PowerBuilder가 기본 오류 메시지 박스를 표시
- **1**: 오류 메시지를 표시하지 않고 계속 진행 (직접 처리했음을 의미)

**DBError 이벤트 활용 예**:
```powerbuilder
// dw_customer의 DBError 이벤트
String ls_msg

CHOOSE CASE sqldbcode
    CASE 2627   // SQL Server 중복 키 오류
        ls_msg = "이미 존재하는 고객 ID입니다."
    CASE 547    // 참조 무결성 오류
        ls_msg = "다른 테이블에서 참조 중인 데이터는 삭제할 수 없습니다."
    CASE ELSE
        ls_msg = "데이터베이스 오류가 발생했습니다.~r~n" + sqlerrtext
END CHOOSE

MessageBox("오류", ls_msg)
RETURN 1   // 기본 메시지 박스 표시 안 함
```

### 2.3 실전 오류 처리 패턴

#### 2.3.1 임베디드 SQL 오류 처리

모든 임베디드 SQL 문 다음에는 SQLCA 상태를 확인하는 것이 좋습니다.

```powerbuilder
DELETE FROM employee WHERE emp_id = :ll_id USING SQLCA;
IF SQLCA.SQLCode = -1 THEN
    ROLLBACK USING SQLCA;
    MessageBox("삭제 오류", SQLCA.SQLErrText)
    RETURN
ELSEIF SQLCA.SQLCode = 100 THEN
    // 해당 데이터 없음 - 무시하거나 메시지 표시
END IF
```

#### 2.3.2 DataWindow Update 오류 처리

DataWindow의 `Update()` 함수는 성공 시 1을 반환하고 실패 시 -1을 반환합니다. 실패 원인은 SQLCA에서 확인합니다.

```powerbuilder
IF dw_1.Update() = 1 THEN
    COMMIT USING SQLCA;
ELSE
    ROLLBACK USING SQLCA;
    MessageBox("업데이트 오류", "코드: " + String(SQLCA.SQLDBCode) + "~r~n" + SQLCA.SQLErrText)
END IF
```

#### 2.3.3 저장 프로시저 오류 처리

프로시저 호출 후 SQLCA 상태를 확인합니다.

```powerbuilder
// RPCFUNC 방식을 사용하는 경우
int li_rtn = SQLCA.sp_update_inventory(ll_product_id, ll_quantity)
IF li_rtn < 0 OR SQLCA.SQLCode <> 0 THEN
    ROLLBACK USING SQLCA;
    MessageBox("재고 업데이트 실패", SQLCA.SQLErrText)
ELSE
    COMMIT USING SQLCA;
END IF
```

### 2.4 DBError 이벤트 상세 활용

DBError 이벤트는 특히 DataWindow에서 사용자 입력 오류를 우아하게 처리하는 데 유용합니다.

**예제: 오류 로깅 및 사용자 정의 메시지**

```powerbuilder
// dw_orders의 DBError 이벤트
String ls_log
Long ll_row = row

// 오류 로그 파일에 기록
ls_log = String(Today(), "yyyy-mm-dd") + " " + String(Now()) + " "
ls_log += "오류 코드: " + String(sqldbcode) + "~r~n"
ls_log += "메시지: " + sqlerrtext + "~r~n"
ls_log += "SQL: " + sqlsyntax + "~r~n~r~n"

// 로그 파일에 추가 (예: error.log)
FileWrite(FileOpen("error.log", LineMode!, Write!, LockWrite!, Append!), ls_log)

// 사용자에게 친숙한 메시지 표시
IF sqldbcode = 8152 THEN   // 문자열 잘림 오류 (SQL Server)
    MessageBox("입력 오류", "입력한 값이 너무 깁니다. 컬럼 길이를 확인하세요.")
ELSE
    MessageBox("데이터베이스 오류", sqlerrtext)
END IF

RETURN 1   // 기본 오류 메시지 방지
```

### 2.5 트랜잭션 관리와 오류 처리 결합

오류 발생 시 트랜잭션을 적절히 롤백해야 데이터 무결성이 유지됩니다. 일반적인 패턴은 다음과 같습니다.

```powerbuilder
// 여러 DataWindow 업데이트
IF dw_master.Update() = 1 AND dw_detail.Update() = 1 THEN
    COMMIT USING SQLCA;
ELSE
    ROLLBACK USING SQLCA;
    // 추가 오류 처리 (DBError 이벤트가 이미 처리했을 수 있음)
END IF
```

---

## 3. 종합 예제: 안전한 CRUD 구현 (RPCFUNC 방식 권장)

다음은 저장 프로시저를 호출하고 오류를 처리하는 완전한 예제입니다. 시나리오는 고객 정보를 저장 프로시저를 통해 입력/수정하는 화면입니다. 여기서는 실무 표준인 **RPCFUNC 방식**을 사용합니다.

### 3.1 윈도우 구성
- DataWindow 컨트롤: `dw_customer` (Freeform 스타일, `d_customer` 객체 연결)
- 버튼: `cb_save` (저장)

### 3.2 저장 프로시저 (가상)
```sql
-- SQL Server 예
CREATE PROCEDURE sp_save_customer
    @cust_id    VARCHAR(10),
    @cust_name  VARCHAR(50),
    @phone      VARCHAR(20),
    @grade      CHAR(1),
    @mode       CHAR(1),   -- 'I': 입력, 'U': 수정
    @result     INT OUTPUT -- 0: 성공, 1: 실패
AS
BEGIN
    IF @mode = 'I'
        INSERT INTO customer (cust_id, cust_name, phone, grade)
        VALUES (@cust_id, @cust_name, @phone, @grade)
    ELSE
        UPDATE customer
        SET cust_name = @cust_name,
            phone = @phone,
            grade = @grade
        WHERE cust_id = @cust_id
    
    IF @@ERROR = 0
        SET @result = 0
    ELSE
        SET @result = 1
END
```

### 3.3 RPCFUNC 선언

먼저 트랜잭션 객체를 확장하는 사용자 객체(`u_trans`)를 만들고, 저장 프로시저를 선언합니다.

```powerbuilder
// u_trans 사용자 객체 (transaction 상속)
// Local External Functions
FUNCTION int sp_save_customer (string id, string name, string phone, string grade, string mode, REF int result) RPCFUNC
```

애플리케이션 속성에서 SQLCA의 타입을 `u_trans`로 변경합니다.

### 3.4 윈도우 스크립트

**윈도우 인스턴스 변수**
```powerbuilder
String is_mode   // 'I' 또는 'U'
```

**cb_save의 Clicked 이벤트 (RPCFUNC 방식)**
```powerbuilder
// DataWindow에서 현재 행의 데이터 가져오기
String ls_id, ls_name, ls_phone, ls_grade
ls_id = dw_customer.GetItemString(1, "cust_id")
ls_name = dw_customer.GetItemString(1, "cust_name")
ls_phone = dw_customer.GetItemString(1, "phone")
ls_grade = dw_customer.GetItemString(1, "grade")

// 모드 결정 (간단히 ID 존재 여부로 판단)
is_mode = "I"   // 실제로는 조회 로직 필요

// 저장 프로시저 호출 (RPCFUNC)
int li_proc_result, li_rtn
li_rtn = SQLCA.sp_save_customer(ls_id, ls_name, ls_phone, ls_grade, is_mode, li_proc_result)

IF SQLCA.SQLCode = 0 AND li_proc_result = 0 THEN
    COMMIT USING SQLCA;
    MessageBox("성공", "저장되었습니다.")
    is_mode = "U"   // 이후 수정 모드
ELSE
    ROLLBACK USING SQLCA;
    MessageBox("저장 실패", "코드: " + String(SQLCA.SQLDBCode) + "~r~n" + SQLCA.SQLErrText)
END IF
```

**DECLARE 방식을 사용한다면 (비교)**
```powerbuilder
// DECLARE 방식 (파라미터 순서대로 나열)
DECLARE sp_save PROCEDURE FOR sp_save_customer
    (:ls_id, :ls_name, :ls_phone, :ls_grade, :is_mode, :li_proc_result OUTPUT)
    USING SQLCA;

EXECUTE sp_save;

IF SQLCA.SQLCode = 0 AND li_proc_result = 0 THEN
    COMMIT USING SQLCA;
    MessageBox("성공", "저장되었습니다.")
ELSE
    ROLLBACK USING SQLCA;
    MessageBox("저장 실패", SQLCA.SQLErrText)
END IF

CLOSE sp_save;
```

RPCFUNC 방식이 훨씬 간결하고 가독성이 좋습니다.

### 3.5 DataWindow Update()와 DBError 활용

만약 저장 프로시저 대신 DataWindow의 내장 Update 기능을 사용한다면 DBError 이벤트가 핵심 역할을 합니다.

```powerbuilder
// cb_save Clicked 이벤트 (DataWindow Update 방식)
IF dw_customer.Update() = 1 THEN
    COMMIT USING SQLCA;
    MessageBox("성공", "저장되었습니다.")
ELSE
    ROLLBACK USING SQLCA;
    // DBError 이벤트에서 이미 오류 메시지를 표시했을 수 있으므로
    // 여기서는 추가 처리 없음 (또는 공통 오류 메시지)
END IF
```

**dw_customer DBError 이벤트** (앞서 예시와 동일)

---

## 4. 결론

저장 프로시저 호출과 데이터베이스 오류 처리는 PowerBuilder 애플리케이션의 안정성과 사용자 경험을 좌우하는 중요한 요소입니다.

- **저장 프로시저**는 복잡한 비즈니스 로직을 데이터베이스에 캡슐화하여 성능과 보안을 향상시킵니다. PowerBuilder는 DECLARE/EXECUTE, DataWindow 데이터 소스, RPCFUNC 등 다양한 호출 방식을 제공합니다. **실무에서는 RPCFUNC 방식이 가장 권장**되는데, 코드가 간결하고 유지보수가 쉽기 때문입니다.
- **오류 처리**는 모든 데이터베이스 작업 후에 SQLCA 상태를 확인하는 기본 원칙을 지키고, 특히 DataWindow에서는 **DBError 이벤트**를 적극 활용하여 사용자에게 친숙한 메시지를 제공하고 오류를 로깅해야 합니다.

실무에서는 다음과 같은 추가 팁을 기억하세요.
- DBMS별 오류 코드를 표로 관리하여 일관된 메시지 제공
- 중요한 오류는 로그 파일이나 데이터베이스에 기록
- 사용자 작업 재시도 가능한 경우 재시도 로직 구현
- 트랜잭션 경계를 명확히 하여 부분 업데이트 방지

이러한 기법들을 적용하면 예외 상황에서도 데이터 무결성을 유지하고 사용자 신뢰도를 높일 수 있습니다.