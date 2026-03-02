---
layout: post
title: PowerBuilder - SQLCA 이해 및 DBMS 연결 설정
date: 2025-12-13 20:30:23 +0900
category: PowerBuilder
---
# Transaction Object (SQLCA) 이해 및 DBMS 연결 설정 (DB Profile)

PowerBuilder 애플리케이션이 데이터베이스와 통신하려면 **트랜잭션 객체(Transaction Object)**라는 특별한 매개체가 필요합니다. 트랜잭션 객체는 PowerBuilder와 데이터베이스 사이의 **통신 영역(Communications Area)** 역할을 하는 비시각적 객체로, 데이터베이스 연결에 필요한 모든 정보와 최근 SQL 작업의 상태 정보를 담고 있습니다. 이 글에서는 기본 트랜잭션 객체인 SQLCA의 개념부터 DB Profile을 이용한 연결 설정 방법까지 상세히 알아보겠습니다.

---

## 1. Transaction Object란?

### 개념과 역할

트랜잭션 객체는 PowerBuilder 프로그램과 데이터베이스 사이에서 정보를 전달하는 **구조체 변수**입니다. 총 15개의 멤버(속성)로 구성되어 있으며, 다음과 같은 두 가지 주요 역할을 수행합니다:

1. **연결 정보 제공**: 데이터베이스에 연결하기 위해 필요한 DBMS 종류, 서버 이름, 사용자 ID, 비밀번호 등의 정보를 담습니다.
2. **실행 결과 수신**: SQL 문 실행 후 성공/실패 여부, 오류 코드, 오류 메시지 등을 반환받습니다.

### SQLCA (SQL Communications Area)

PowerBuilder는 애플리케이션 실행을 시작할 때 **전역 기본 트랜잭션 객체**인 `SQLCA`를 자동으로 생성합니다. 대부분의 애플리케이션은 하나의 데이터베이스와 통신하므로, 별도의 트랜잭션 객체를 만들지 않고 SQLCA만으로 충분한 경우가 많습니다.

SQLCA는 Application 객체의 Open 이벤트가 실행되기 전에 이미 생성되어 있으므로, 어느 스크립트에서든 바로 사용할 수 있습니다.

```
// SQLCA는 애플리케이션 전체에서 사용 가능한 전역 객체입니다.
CONNECT USING SQLCA;
```

---

## 2. Transaction Object의 속성

트랜잭션 객체는 15개의 속성을 가지고 있으며, 이 중 10개는 데이터베이스 연결을 위해 사용되고, 5개는 최근 SQL 작업의 성공/실패 상태 정보를 받아오는 데 사용됩니다.

### 연결 속성 (Connection Properties)

| 속성 | 데이터 타입 | 설명 | DB Profile 필드 |
|------|------------|------|-----------------|
| **DBMS** | String | 사용할 데이터베이스 관리 시스템의 식별자 (예: "ODBC", "MSS Microsoft SQL Server", "ORA Oracle") | DBMS |
| **Database** | String | 연결할 데이터베이스 이름 | Database Name |
| **UserID** | String | 데이터베이스에 연결하는 사용자 이름 (일부 DBMS에서는 필요 없음) | User ID |
| **DBPass** | String | 데이터베이스 연결용 비밀번호 | Password |
| **Lock** | String | DBMS가 지원하는 경우, 격리 수준(Isolation Level) 설정 | Isolation Level |
| **LogID** | String | 데이터베이스 서버에 로그인하는 사용자 이름 (Oracle, Sybase 등에서 필요) | Login ID |
| **LogPass** | String | 데이터베이스 서버 로그인용 비밀번호 | Login Password |
| **ServerName** | String | 데이터베이스가 위치한 서버 이름 | Server Name |
| **AutoCommit** | Boolean | 트랜잭션 자동 커밋 여부 (True: 각 SQL 문 즉시 커밋, False: 수동 커밋 필요) | AutoCommit Mode |
| **DBParm** | String | DBMS별 특수 연결 파라미터를 지정 (ODBC 연결문자열, 기타 옵션) | DBPARM |

### 상태 정보 속성 (Status Properties)

| 속성 | 데이터 타입 | 설명 |
|------|------------|------|
| **SQLCode** | Long | 최근 SQL 작업 결과 코드 (0: 성공, -1: 오류, 100: 검색된 데이터 없음) |
| **SQLNRows** | Long | 최근 SQL 작업에 영향을 받은 행 수 |
| **SQLDBCode** | Long | 데이터베이스 벤더 고유 오류 코드 |
| **SQLErrText** | String | 오류 코드에 해당하는 오류 메시지 |
| **SQLReturnData** | String | DBMS별 추가 정보 (Informix 등에서 사용) |

### DBMS별 지원 속성

모든 DBMS가 동일한 속성을 사용하는 것은 아닙니다. 다음은 주요 DBMS별로 사용 가능한 트랜잭션 객체 속성입니다.

| DBMS | 사용 가능한 속성 |
|------|-----------------|
| **ODBC** | DBMS, UserID*, LogID+, LogPass+, DBParm, Lock, AutoCommit, SQLReturnData, SQLCode, SQLNRows, SQLDBCode, SQLErrText |
| **MS SQL Server** | DBMS, Database, ServerName, LogID, LogPass, DBParm, Lock, AutoCommit, SQLCode, SQLNRows, SQLDBCode, SQLErrText |
| **Oracle** | DBMS, ServerName, LogID, LogPass, DBParm, SQLReturnData, SQLCode, SQLNRows, SQLDBCode, SQLErrText |
| **Sybase ASE** | DBMS, Database, ServerName, LogID, LogPass, DBParm, Lock, AutoCommit, SQLCode, SQLNRows, SQLDBCode, SQLErrText |

\* UserID는 ODBC에서 선택 사항입니다. 이 속성을 지정하면 ODBC SQLGetInfo 호출이 반환하는 연결의 UserName 속성을 덮어씁니다.  
\+ LogID와 LogPass는 ODBC 드라이버가 SQL 드라이버 CONNECT 호출을 지원하지 않는 경우에만 PowerBuilder가 사용합니다.

---

## 3. DBMS 연결 설정 (DB Profile)

### DB Profile이란?

DB Profile은 PowerBuilder 개발 환경에서 특정 데이터베이스에 연결하기 위한 **연결 설정 정보를 저장해 놓은 것**입니다. 한 번 설정해 놓은 Profile을 통해 쉽게 데이터베이스에 연결할 수 있으며, 이 Profile에 설정한 값들이 트랜잭션 객체의 연결 속성에 매핑됩니다.

### DB Profile 생성 방법

1. **Database Painter 열기**
   - PowerBuilder 도구 모음에서 **Database** 아이콘 클릭
   - 또는 메뉴에서 **Tools → Database Painter** 선택

2. **새 Profile 생성**
   - 연결하려는 DBMS 유형(예: ODBC, MSS Microsoft SQL Server, ORA Oracle)을 마우스 오른쪽 클릭
   - **New Profile** 선택

3. **연결 정보 입력**
   ```
   [Database Profile Setup 대화상자]
   ┌─────────────────────────────────────┐
   │ Profile Name: MyApp_Dev             │
   ├─────────────────────────────────────┤
   │ [Connection] 탭                      │
   │   - DBMS:      MSS Microsoft SQL Server│
   │   - Server:    localhost\SQLExpress  │
   │   - Database:  MyCompanyDB           │
   │   - Login ID:  sa                    │
   │   - Password:  ********               │
   ├─────────────────────────────────────┤
   │ [Preview] 탭                          │
   │   // 자동 생성된 연결 코드 확인 가능   │
   │   SQLCA.DBMS = "MSS Microsoft SQL Server"│
   │   SQLCA.Database = "MyCompanyDB"       │
   │   SQLCA.ServerName = "localhost\SQLExpress"│
   │   SQLCA.LogId = "sa"                   │
   │   SQLCA.LogPass = "********"            │
   └─────────────────────────────────────┘
   ```

4. **연결 테스트**
   - **Preview** 탭 하단의 **Test Connection** 버튼을 클릭하여 연결 확인

5. **저장**
   - **OK** 버튼을 클릭하여 Profile 저장

### Preview 탭의 활용

DB Profile의 **Preview** 탭에는 지금까지 입력한 설정에 해당하는 **PowerScript 코드가 자동으로 생성**됩니다. 이 코드를 복사하여 애플리케이션의 연결 설정 부분에 붙여넣을 수 있어 매우 편리합니다.

```
// Preview 탭에서 생성된 코드 예시 (ODBC 연결)
SQLCA.DBMS = "ODBC"
SQLCA.AutoCommit = False
SQLCA.DBParm = "ConnectString='DSN=MyDB;UID=sa;PWD=1234'"
```

---

## 4. SQLCA를 이용한 데이터베이스 연결 실전

### 기본 연결 절차

PowerBuilder 애플리케이션에서 데이터베이스와 통신하기 위한 기본 단계는 다음과 같습니다.

1. **트랜잭션 객체에 연결 속성 값 할당**
2. **데이터베이스 연결 (CONNECT)**
3. **DataWindow 컨트롤에 트랜잭션 객체 할당 (SetTransObject)**
4. **데이터베이스 처리 (Retrieve/Update)**
5. **데이터베이스 연결 해제 (DISCONNECT)**

### 예제: Application Open 이벤트에서 연결

```powerbuilder
// Application 객체의 Open 이벤트
// SQLCA 속성 설정 (DB Profile Preview 코드 활용)
SQLCA.DBMS = "MSS Microsoft SQL Server"
SQLCA.Database = "MyCompanyDB"
SQLCA.ServerName = "localhost\SQLExpress"
SQLCA.LogId = "sa"
SQLCA.LogPass = "1234"
SQLCA.AutoCommit = False

// 데이터베이스 연결
CONNECT USING SQLCA;

// 연결 오류 확인
IF SQLCA.SQLCode <> 0 THEN
   MessageBox("연결 오류", "코드: " + String(SQLCA.SQLDBCode) + "~r~n" + SQLCA.SQLErrText)
   HALT CLOSE   // 애플리케이션 종료
END IF

// 메인 윈도우 열기
Open(w_main)
```

### 예제: 윈도우에서 DataWindow 연결 및 조회

```powerbuilder
// w_main의 Open 이벤트
// DataWindow에 트랜잭션 객체 할당
dw_employee.SetTransObject(SQLCA)

// 데이터 조회
IF dw_employee.Retrieve() < 0 THEN
   MessageBox("조회 오류", "데이터를 가져올 수 없습니다.")
END IF
```

### 예제: 윈도우 Close 이벤트에서 연결 해제

```powerbuilder
// w_main의 Close 이벤트
DISCONNECT USING SQLCA;

IF SQLCA.SQLCode <> 0 THEN
   MessageBox("연결 해제 오류", SQLCA.SQLErrText)
END IF
```

### 오류 처리 패턴 (임베디드 SQL vs DataWindow)

PowerScript에서 SQL 작업 후 오류를 처리할 때는 **임베디드 SQL**과 **DataWindow**의 경우를 구분해야 합니다.

#### [1] 임베디드 SQL (SELECT INTO, INSERT, UPDATE, DELETE)

임베디드 SQL을 실행한 후에는 SQLCA의 상태 속성(`SQLCode`, `SQLDBCode`, `SQLErrText`)을 확인하여 성공 여부를 판단합니다.

```powerbuilder
// SELECT INTO 예제
String ls_name
SELECT cust_name INTO :ls_name FROM customer WHERE cust_id = 'X999';
IF SQLCA.SQLCode = -1 THEN
    MessageBox("DB 오류", SQLCA.SQLErrText)
    ROLLBACK USING SQLCA;
ELSEIF SQLCA.SQLCode = 100 THEN
    MessageBox("알림", "검색된 데이터가 없습니다.")   // 데이터 없음!
ELSE
    MessageBox("성공", "고객명: " + ls_name)
END IF
```

#### [2] DataWindow Retrieve

DataWindow의 `Retrieve()` 함수는 성공 시 가져온 행 수를 반환하고, 실패 시 음수를 반환합니다. SQLCA의 상태값은 Retrieve 자체의 성공 여부와 무관하게 이전 작업의 영향을 받을 수 있으므로, Retrieve의 결과는 반환값으로 확인해야 합니다.

```powerbuilder
Long ll_rows
ll_rows = dw_employee.Retrieve()

IF ll_rows < 0 THEN
    MessageBox("DB 오류", SQLCA.SQLErrText)   // 실제 오류 발생 시
ELSEIF ll_rows = 0 THEN
    MessageBox("알림", "조회된 조건의 데이터가 없습니다.")   // 데이터 없음!
END IF
```

> **⚠️ 중요**: SQLCA.SQLCode = 100은 **임베디드 SQL**에서만 "검색 결과 없음"을 의미합니다. DataWindow의 Retrieve 결과는 반드시 함수의 반환값(Return Value)으로 확인해야 합니다.

---

## 5. 여러 데이터베이스 연결 (다중 트랜잭션 객체)

애플리케이션이 동시에 여러 데이터베이스에 연결해야 하는 경우, SQLCA 외에 **별도의 트랜잭션 객체를 직접 생성**하여 사용할 수 있습니다.

### 별도 트랜잭션 객체 생성 및 사용

```powerbuilder
// 전역 변수 선언 또는 인스턴스 변수 선언
Transaction trans_oracle

// 연결이 필요한 시점에서
trans_oracle = CREATE Transaction   // 객체 생성

// Oracle 연결 속성 설정
trans_oracle.DBMS = "ORA Oracle"
trans_oracle.LogID = "scott"
trans_oracle.LogPass = "tiger"
trans_oracle.ServerName = "orcl"
trans_oracle.AutoCommit = False

// Oracle 데이터베이스 연결
CONNECT USING trans_oracle;

// DataWindow에 Oracle 트랜잭션 객체 할당
dw_oracle.SetTransObject(trans_oracle)
dw_oracle.Retrieve()

// SQLCA로는 SQL Server 연결 유지 (별도 작업)

// 사용 후 연결 해제 및 객체 파괴
DISCONNECT USING trans_oracle;
DESTROY trans_oracle   // 메모리 해제 필수!
```

### USING 절로 트랜잭션 객체 지정

임베디드 SQL에서도 `USING` 절을 통해 사용할 트랜잭션 객체를 지정할 수 있습니다.

```powerbuilder
// SQL Server (SQLCA)에 INSERT
INSERT INTO employee (id, name) 
VALUES ('1001', '홍길동')
USING SQLCA;

// Oracle (trans_oracle)에 INSERT
INSERT INTO dept (deptno, dname) 
VALUES (10, 'SALES')
USING trans_oracle;
```

---

## 6. 실무 활용 팁

### 1. INI 파일을 이용한 연결 정보 관리

데이터베이스 연결 정보를 코드에 직접 하드코딩하면 환경 변경(개발→테스트→운영) 시 매번 수정해야 합니다. INI 파일을 사용하여 연결 정보를 분리하면 편리합니다.

**config.ini**
```ini
[Database]
DBMS=ODBC
Database=MyDB
DBParm=ConnectString='DSN=MyDB;UID=sa;PWD=1234'
AutoCommit=False
```

**애플리케이션 코드**
```powerbuilder
// INI 파일에서 연결 정보 읽기
SQLCA.DBMS = ProfileString("config.ini", "Database", "DBMS", "")
SQLCA.DBParm = ProfileString("config.ini", "Database", "DBParm", "")
SQLCA.AutoCommit = ProfileString("config.ini", "Database", "AutoCommit", "False") = "True"

CONNECT USING SQLCA;
```

### 2. 트랜잭션 객체 확장 (사용자 정의 객체)

SQLCA의 기능을 확장하여 저장 프로시저 호출 등의 특수 기능을 추가할 수 있습니다. 트랜잭션 객체를 상속받는 표준 클래스 사용자 객체를 만들고, RPCFUNC 키워드로 저장 프로시저를 외부 함수로 선언합니다.

```powerbuilder
// u_trans_database 사용자 객체 (transaction 상속)
// Local External Functions 선언
FUNCTION double give_raise(REF double salary) RPCFUNC

// 애플리케이션에서 SQLCA 타입을 u_trans_database로 지정
// Application 객체의 Additional Properties → Variable Types 탭
SQLCA: u_trans_database
```

### 3. 트랜잭션 풀(Transaction Pool) 활용

다중 사용자 환경에서 데이터베이스 연결을 효율적으로 관리하려면 트랜잭션 풀을 사용할 수 있습니다. `SetTransPool` 함수로 최소/최대 연결 수를 설정합니다.

```powerbuilder
// Application Open 이벤트
// 최소 5개, 최대 10개 연결 유지, 연결 시도 제한시간 10초
SetTransPool(5, 10, 10)
```

### 4. 자주 발생하는 연결 오류와 해결방법

| 오류 증상 | 확인 사항 |
|----------|----------|
| "Database connection failed" | DBMS 식별자(DBMS 속성)가 정확한지 확인 |
| "Login failed" | LogID/LogPass 또는 UserID/DBPass가 올바른지 확인 |
| "Server not found" | ServerName이 정확한지, 네트워크 연결 상태 확인 |
| "Cannot connect to database" | ODBC DSN 설정 또는 데이터베이스 서비스 실행 여부 확인 |

---

## 결론

트랜잭션 객체(SQLCA)는 PowerBuilder 애플리케이션이 데이터베이스와 대화할 수 있게 해주는 핵심 통로입니다. SQLCA는 애플리케이션 시작 시 자동 생성되는 기본 트랜잭션 객체로, 대부분의 경우 이것만으로 충분합니다. 트랜잭션 객체의 15개 속성을 이해하고 올바르게 설정하는 것이 데이터베이스 연동의 첫걸음입니다.

특히 SQL 작업 후 오류 처리 시 **임베디드 SQL과 DataWindow의 차이**를 명확히 이해해야 합니다. 임베디드 SQL은 `SQLCA.SQLCode`로 결과를 확인하지만, DataWindow의 `Retrieve()`는 반환값(Return Value)으로 결과를 확인해야 합니다. 이 차이를 모르면 데이터가 없을 때 잘못된 오류 처리를 하기 쉽습니다.

DB Profile은 개발 환경에서 데이터베이스 연결 설정을 쉽게 저장하고 테스트할 수 있게 해주며, Preview 탭에서 생성되는 코드를 그대로 애플리케이션에 활용할 수 있어 효율적입니다.

실무에서는 INI 파일을 이용한 연결 정보 분리, 트랜잭션 객체 확장, 트랜잭션 풀 활용 등의 기법을 통해 더 안정적이고 유연한 데이터베이스 연결을 구현할 수 있습니다. 특히 여러 데이터베이스를 동시에 사용해야 하는 경우에는 SQLCA 외에 별도의 트랜잭션 객체를 직접 생성하여 사용하는 방법을 숙지해야 합니다.

다음 글에서는 PowerBuilder의 데이터베이스 처리 기능을 더욱 강력하게 만들어 주는 **고급 SQL 기법(동적 SQL, 저장 프로시저 호출 등)**에 대해 알아보겠습니다.