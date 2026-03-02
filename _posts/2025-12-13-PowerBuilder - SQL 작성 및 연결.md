---
layout: post
title: PowerBuilder - SQL 작성 및 연결
date: 2025-12-13 16:30:23 +0900
category: PowerBuilder
---
# SQL 작성 및 연결 (Retrieve, Update, Insert, Delete)

DataWindow의 진정한 힘은 데이터베이스와의 연동에서 발휘됩니다. DataWindow는 내부적으로 SQL을 생성하고 실행하여 데이터를 검색(Retrieve)하고, 변경된 내용을 데이터베이스에 반영(Update, Insert, Delete)합니다. 개발자는 복잡한 SQL 문을 직접 작성할 필요 없이 DataWindow의 함수와 속성을 통해 데이터를 쉽게 조작할 수 있습니다. 이 글에서는 DataWindow를 데이터베이스에 연결하는 방법부터 CRUD 작업의 구체적인 구현까지 상세히 알아보겠습니다.

---

## 1. DataWindow와 데이터베이스 연결

DataWindow가 데이터베이스와 통신하려면 **트랜잭션 객체(Transaction Object)**를 연결해야 합니다. PowerBuilder는 기본 트랜잭션 객체로 `SQLCA`를 제공합니다. SQLCA는 애플리케이션 시작 시 자동 생성되며, 데이터베이스 연결 정보를 담고 있습니다.

### SetTransObject vs SetTrans

DataWindow에 트랜잭션 객체를 연결하는 방법은 두 가지입니다.

| 함수 | 설명 |
|------|------|
| **SetTransObject ( transaction )** | 이미 연결된 트랜잭션 객체를 사용합니다. 데이터베이스 연결은 별도로 수행해야 합니다. 가장 일반적인 방법입니다. |
| **SetTrans ( transaction )** | 트랜잭션 객체의 연결 정보만 복사해두고, DataWindow가 필요할 때 자동으로 연결/해제합니다. 성능상 불리하므로 특별한 경우 외에는 사용하지 않습니다.

일반적으로 `SetTransObject`를 사용하며, 데이터베이스 연결은 애플리케이션 시작 시 한 번 수행하고 트랜잭션 객체를 계속 재사용합니다.

```powerbuilder
// 데이터베이스 연결 (보통 Application Open 이벤트)
SQLCA.DBMS = "ODBC"
SQLCA.DBParm = "ConnectString='DSN=MyDB;UID=sa;PWD=1234'"
CONNECT USING SQLCA;
IF SQLCA.SQLCode <> 0 THEN
   MessageBox("오류", "데이터베이스 연결 실패: " + SQLCA.SQLErrText)
   HALT
END IF

// 윈도우 Open 이벤트에서 DataWindow에 트랜잭션 객체 연결
dw_customer.SetTransObject(SQLCA)
```

이제 DataWindow는 SQLCA를 통해 데이터베이스와 통신할 준비가 되었습니다.

---

## 2. 데이터 검색 (Retrieve)

`Retrieve()` 함수는 DataWindow에 정의된 SELECT 문을 실행하여 데이터를 가져옵니다.

### 기본 Retrieve

```powerbuilder
Long ll_rows
ll_rows = dw_customer.Retrieve()
IF ll_rows < 0 THEN
   MessageBox("오류", "데이터 검색 실패")
END IF
```

`Retrieve()`는 성공 시 가져온 행 수를 반환하고, 실패 시 음수를 반환합니다.

### 인자가 있는 Retrieve

DataWindow의 SELECT 문에 WHERE 절의 파라미터를 정의했다면, `Retrieve()` 함수에 인자를 전달할 수 있습니다.

```powerbuilder
// DataWindow SQL: SELECT * FROM customer WHERE cust_id = :id
ll_rows = dw_customer.Retrieve("C001")
```

여러 개의 인자는 순서대로 전달합니다.

```powerbuilder
// SQL: SELECT * FROM orders WHERE cust_id = :id AND order_date >= :date
ll_rows = dw_orders.Retrieve("C001", 2025-01-01)
```

### Retrieve 관련 유용한 함수

| 함수 | 설명 |
|------|------|
| `SetTransObject()` | 트랜잭션 객체 연결 |
| `GetTransObject()` | 현재 연결된 트랜잭션 객체 반환 |
| `Retrieve()` | 데이터 조회 |
| `Reset()` | DataWindow의 모든 행 삭제 (데이터베이스에는 영향 없음) |

---

## 3. 데이터 삽입 (Insert)

DataWindow에 새로운 행을 추가하려면 `InsertRow()` 함수를 사용합니다. 이 함수는 DataWindow 내부에 빈 행을 추가할 뿐, 데이터베이스에는 아직 저장되지 않습니다. 실제 저장은 `Update()` 함수로 수행합니다.

### InsertRow 사용법

```powerbuilder
Long ll_new_row
ll_new_row = dw_customer.InsertRow(0)   // 0은 마지막 행 다음에 추가
dw_customer.ScrollToRow(ll_new_row)     // 새 행으로 스크롤
```

`InsertRow()`의 인자는 행을 삽입할 위치입니다.
- `0`: 마지막 행 다음에 추가
- 양수: 해당 행 앞에 추가

### 데이터 입력 후 저장

사용자가 새 행에 데이터를 입력한 후 저장 버튼을 누르면 `Update()`를 호출합니다.

```powerbuilder
// 저장 버튼 Clicked 이벤트
Integer li_rtn
li_rtn = dw_customer.Update()
IF li_rtn = 1 THEN
   COMMIT USING SQLCA;   // 트랜잭션 커밋
   MessageBox("성공", "저장되었습니다.")
ELSE
   ROLLBACK USING SQLCA; // 오류 시 롤백
   MessageBox("오류", "저장 실패: " + SQLCA.SQLErrText)
END IF
```

`Update()` 함수는 DataWindow에서 변경된 내용(삽입, 수정, 삭제)을 데이터베이스에 반영합니다. 성공 시 1을 반환합니다.

---

## 4. 데이터 갱신 (Update)

DataWindow에서 기존 행의 데이터를 수정하면 자동으로 변경 상태가 추적됩니다. `Update()`를 호출하면 변경된 내용이 데이터베이스에 반영됩니다.

### Update 과정

1. 사용자가 DataWindow에서 값을 수정합니다.
2. `Update()` 호출 시 DataWindow는 수정된 행에 대해 UPDATE 문을 생성하여 실행합니다.
3. 성공적으로 반영되면 내부 변경 플래그를 초기화합니다.

### Update() 함수의 인자 이해

`Update()` 함수는 선택적으로 두 개의 Boolean 인자를 받을 수 있습니다.

- **첫 번째 인자 (`accepttext`)**: `TRUE`이면 업데이트를 수행하기 전에 현재 입력 중인 셀의 내용을 DataWindow 버퍼에 강제로 적용(AcceptText)합니다. 기본값은 `TRUE`입니다.
- **두 번째 인자 (`resetflag`)**: `TRUE`이면 업데이트 성공 후 DataWindow 내부의 변경 상태 플래그(Item Status)를 모두 `NotModified!`로 초기화합니다. 기본값은 `TRUE`입니다.

일반적으로는 인자 없이 `Update()`를 호출하는 것으로 충분합니다. 만약 더 세밀한 제어가 필요하다면 아래와 같이 사용할 수 있습니다.

```powerbuilder
// 첫 번째 인자 TRUE: 저장 전 현재 입력 중인 값을 버퍼에 적용
// 두 번째 인자 FALSE: 업데이트 후 내부 플래그를 초기화하지 않음 (추가 작업 후 직접 초기화할 경우)
dw_customer.Update(TRUE, FALSE)
```

### Update()와 트랜잭션

`Update()`는 자동으로 COMMIT을 수행하지 않습니다. 따라서 반드시 COMMIT 또는 ROLLBACK을 명시적으로 실행해야 합니다. 일반적인 패턴은 다음과 같습니다.

```powerbuilder
IF dw_customer.Update() = 1 THEN
   COMMIT USING SQLCA;
ELSE
   ROLLBACK USING SQLCA;
END IF
```

---

## 5. 데이터 삭제 (Delete)

DataWindow에서 행을 삭제하려면 `DeleteRow()` 함수를 사용합니다. 이 함수는 DataWindow 내부에서 행을 삭제 표시만 하고, 실제 데이터베이스 삭제는 `Update()`가 호출될 때 수행됩니다.

### DeleteRow 사용법

```powerbuilder
Long ll_current_row
ll_current_row = dw_customer.GetRow()   // 현재 선택된 행
dw_customer.DeleteRow(ll_current_row)    // 해당 행 삭제 표시
```

전체 행을 삭제하려면 반복문을 사용하거나 `RowsDelete()` 함수를 고려할 수 있습니다.

### 삭제 후 저장

삭제 표시된 행을 실제 데이터베이스에서 삭제하려면 `Update()`를 호출합니다.

```powerbuilder
// 삭제 버튼 Clicked 이벤트
IF MessageBox("확인", "삭제하시겠습니까?", Question!, YesNo!) = 1 THEN
   dw_customer.DeleteRow(0)   // 0은 현재 행을 의미
   IF dw_customer.Update() = 1 THEN
      COMMIT;
      dw_customer.Retrieve()   // 데이터 다시 조회 (선택)
   ELSE
      ROLLBACK;
   END IF
END IF
```

---

## 6. 트랜잭션 처리

PowerBuilder는 데이터베이스 트랜잭션을 명시적으로 관리해야 합니다. DataWindow의 `Update()` 후 반드시 `COMMIT` 또는 `ROLLBACK`을 수행해야 합니다.

### COMMIT

```powerbuilder
COMMIT USING SQLCA;
```

### ROLLBACK

```powerbuilder
ROLLBACK USING SQLCA;
```

트랜잭션 격리 수준 등은 데이터베이스 연결 시 설정하거나 SQLCA의 AutoCommit 속성으로 제어할 수 있습니다.

---

## 7. 오류 처리

DataWindow 작업 중 오류가 발생하면 `SQLCA.SQLCode`에 오류 코드가 설정되고, `SQLCA.SQLErrText`에 오류 메시지가 저장됩니다.

```powerbuilder
IF dw_customer.Update() = 1 THEN
   COMMIT USING SQLCA;
ELSE
   ROLLBACK USING SQLCA;
   MessageBox("DB 오류", "코드: " + String(SQLCA.SQLCode) + "~r~n" + SQLCA.SQLErrText)
END IF
```

또한 DataWindow 자체의 오류 이벤트(`dberror`)를 처리할 수도 있습니다.

---

## 8. Update Properties 설정

DataWindow가 어떤 테이블에, 어떤 컬럼을, 어떻게 업데이트할지는 **Update Properties**에서 설정합니다. DataWindow Painter에서 **Rows → Update Properties** 메뉴를 선택합니다.

```
[Update Properties 대화상자]
┌─────────────────────────────────┐
│ Allow Updates: [√]               │
│ Table to Update: customer        │
│ Where Clause for Update/Delete:  │
│   ( ) Key Columns                 │
│   ( ) Key and Updateable Columns │
│   ( ) Key and Modified Columns   │
│ Key Modification:                 │
│   ( ) Use Delete then Insert      │
│   ( ) Use Update                  │
│ Updateable Columns:               │
│   [√] cust_id                     │
│   [√] cust_name                   │
│   [√] phone                       │
│   [ ] grade                       │
│ Unique Key Column(s):             │
│   [√] cust_id                     │
└─────────────────────────────────┘
```

### 주요 설정 항목

| 항목 | 설명 |
|------|------|
| **Allow Updates** | 업데이트 허용 여부 |
| **Table to Update** | 업데이트할 테이블 이름 (조인한 경우 반드시 지정) |
| **Where Clause for Update/Delete** | UPDATE/DELETE 시 WHERE 절에 포함할 조건 (동시성 제어) |
| **Key Modification** | 키 컬럼 수정 시 처리 방법 (Update 또는 Delete 후 Insert) |
| **Updateable Columns** | 수정 가능한 컬럼 지정 |
| **Unique Key Column(s)** | 고유 키 컬럼 (기본키) 지정 |

이 설정이 올바르지 않으면 `Update()` 실패의 주요 원인이 됩니다. 특히 조인된 DataWindow의 경우 업데이트할 테이블과 키 컬럼을 정확히 지정해야 합니다.

---

## 9. 실전 예제: 고객 관리 CRUD

간단한 고객 관리 화면을 통해 CRUD를 구현해 보겠습니다.

### 윈도우 구성
- DataWindow 컨트롤: `dw_customer` (Grid 스타일, d_customer 연결)
- 버튼: `cb_retrieve` (조회), `cb_insert` (추가), `cb_delete` (삭제), `cb_save` (저장)

### 9.1 조회 (Retrieve)

```powerbuilder
// cb_retrieve의 Clicked 이벤트
dw_customer.SetTransObject(SQLCA)   // 보통 Open 이벤트에서 한 번만 수행
IF dw_customer.Retrieve() < 0 THEN
   MessageBox("오류", "데이터 조회 실패")
END IF
```

### 9.2 추가 (Insert)

```powerbuilder
// cb_insert의 Clicked 이벤트
Long ll_row
ll_row = dw_customer.InsertRow(0)
dw_customer.ScrollToRow(ll_row)
dw_customer.SetFocus()   // 입력 가능하도록 포커스 이동
```

### 9.3 삭제 (Delete)

```powerbuilder
// cb_delete의 Clicked 이벤트
Long ll_current
ll_current = dw_customer.GetRow()
IF ll_current > 0 THEN
   IF MessageBox("확인", "선택한 고객을 삭제하시겠습니까?", Question!, YesNo!) = 1 THEN
      dw_customer.DeleteRow(ll_current)
      // 실제 삭제는 저장 버튼에서 Update()로 처리
   END IF
ELSE
   MessageBox("알림", "삭제할 행을 선택하세요.")
END IF
```

### 9.4 저장 (Save) - Update

> **💡 실무 팁: 저장 전엔 반드시 AcceptText()를 호출하라!**  
> 사용자가 셀에 값을 입력하고 엔터나 탭을 치지 않은 상태(커서가 깜빡이는 상태)에서 마우스로 '저장' 버튼을 누르면, 마지막에 입력한 값은 DataWindow 버퍼로 들어가지 않습니다. (Edit Control에만 머물러 있음). 따라서 모든 저장 로직의 첫 줄에는 무조건 **`AcceptText()`**를 호출하여 입력 중인 값을 버퍼로 강제로 밀어 넣어야 합니다.

```powerbuilder
// cb_save의 Clicked 이벤트
// 1. 현재 편집 중인 데이터를 강제로 버퍼에 적용
IF dw_customer.AcceptText() < 0 THEN
    RETURN // 유효성 검사 실패 시 저장 중단
END IF

// 2. Update() 실행
Integer li_rtn
li_rtn = dw_customer.Update()
IF li_rtn = 1 THEN
   COMMIT USING SQLCA;
   MessageBox("성공", "저장되었습니다.")
ELSE
   ROLLBACK USING SQLCA;
   MessageBox("오류", "저장 실패: " + SQLCA.SQLErrText)
END IF
```

### 9.5 윈도우 Open 이벤트

```powerbuilder
// 트랜잭션 연결 및 초기 데이터 조회
dw_customer.SetTransObject(SQLCA)
dw_customer.Retrieve()
```

---

## 10. DataWindow가 생성하는 SQL 확인

DataWindow가 실제로 어떤 SQL을 실행하는지 확인하려면 `Describe()` 함수를 사용하거나, **SQLPreview** 이벤트에서 확인할 수 있습니다. 또는 데이터베이스 모니터링 도구를 사용할 수도 있습니다.

```powerbuilder
// dw_customer의 SQLPreview 이벤트
// 이 이벤트는 DataWindow가 SQL을 실행하기 직전에 발생하며,
// sqlsyntax 인자에 실행될 SQL 문이 전달됩니다.
String ls_sql
ls_sql = sqlsyntax
MessageBox("실행 SQL", ls_sql)
```

이를 통해 DataWindow가 생성한 정확한 INSERT, UPDATE, DELETE 문을 확인할 수 있습니다.

---

## 결론

DataWindow를 통한 SQL 작업은 매우 직관적이고 강력합니다. `Retrieve()`, `Update()`, `InsertRow()`, `DeleteRow()` 같은 간단한 함수 호출만으로 복잡한 데이터베이스 연동을 구현할 수 있습니다. 다만, 트랜잭션 관리와 Update Properties 설정을 정확히 해주어야 예상치 못한 오류를 방지할 수 있습니다. 또한 저장 버튼에서는 `AcceptText()`를 먼저 호출하여 입력 중인 데이터를 안전하게 처리하는 습관이 필요합니다.

이러한 CRUD 기본기를 바탕으로 필터링, 정렬, 그룹화, 동적 DataWindow 등 고급 기능으로 확장해 나갈 수 있습니다. 다음 글에서는 DataWindow의 고급 기능인 필터, 정렬, 그룹화, 계산 컬럼 등에 대해 자세히 알아보겠습니다.