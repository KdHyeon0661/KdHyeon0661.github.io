---
layout: post
title: PowerBuilder - DataWindow의 핵심 원리
date: 2025-12-13 17:30:23 +0900
category: PowerBuilder
---
# DataWindow의 핵심 원리: 4가지 버퍼와 상태값

PowerBuilder 개발자라면 누구나 한 번쯤 경이로움을 느꼈을 것입니다. 복잡한 SQL 문을 직접 작성하지 않았는데, `dw_1.Update()` 한 줄만으로 데이터베이스가 정확하게 갱신되는 모습을 보면서 말이죠. 이 마법 같은 현상은 DataWindow 내부의 정교한 **버퍼(Buffer)** 시스템과 **아이템 상태(Item Status)** 관리 메커니즘 덕분입니다. 이 글에서는 DataWindow가 메모리에서 데이터를 어떻게 관리하고, 변경 사항을 추적하며, 최종적으로 데이터베이스에 반영하는지 그 핵심 원리를 상세히 알아보겠습니다.

---

## 1. DataWindow의 4가지 버퍼 이해하기

DataWindow는 데이터베이스에서 가져온 데이터를 메모리에 로드할 때 단순히 하나의 영역에 저장하지 않고, **4개의 논리적 버퍼(Buffer)**로 분류하여 관리합니다. 각 버퍼는 서로 다른 목적과 수명을 가지고 있습니다.

### 1.1 Primary! 버퍼 (기본 버퍼)

**Primary!** 버퍼는 현재 DataWindow 컨트롤에 **표시되고 있는 데이터**가 위치하는 주 메모리 영역입니다.

- 사용자가 화면에서 보고 있는 모든 행은 이 버퍼에 존재합니다.
- 스크롤, 편집, 선택 등 모든 사용자 상호작용의 대상이 됩니다.
- DataWindow에 연결된 DataStore도 동일하게 Primary! 버퍼를 가집니다.

### 1.2 Delete! 버퍼 (삭제 버퍼)

**Delete!** 버퍼는 삭제된 행이 임시로 보관되는 **휴지통** 같은 영역입니다.

- `DeleteRow()` 함수를 호출하면 해당 행이 Primary! 버퍼에서 즉시 사라지는 것처럼 보이지만, 실제로는 Delete! 버퍼로 이동합니다.
- 사용자는 화면에서 삭제된 행을 볼 수 없지만, DataWindow 내부에서는 이 행을 계속 추적하고 있습니다.
- `Update()` 함수가 호출되면 Delete! 버퍼에 있는 행들을 대상으로 DELETE 문이 생성됩니다.
- `Rollback()`이 발생하면 Delete! 버퍼의 행들은 다시 Primary! 버퍼로 복원될 수 있습니다.

### 1.3 Filter! 버퍼 (필터 버퍼)

**Filter!** 버퍼는 필터 조건에 의해 **화면에서 숨겨진 행**들이 보관되는 영역입니다.

- `SetFilter()`와 `Filter()` 함수를 사용하면 조건에 맞지 않는 행들이 Primary! 버퍼에서 Filter! 버퍼로 이동합니다.
- 사용자는 이 행들을 볼 수 없지만, DataWindow 내부 데이터에서는 여전히 존재합니다.
- 필터를 해제하면(`SetFilter("")` 후 `Filter()` 호출) Filter! 버퍼의 행들이 다시 Primary! 버퍼로 돌아옵니다.
- 필터는 단지 화면 표시만 제어할 뿐, 데이터 자체가 삭제된 것은 아닙니다.

### 1.4 Original! 버퍼 (원본 버퍼)

**Original!** 버퍼는 가장 중요한 버퍼 중 하나로, 데이터베이스에서 처음 조회된 **원본 값**을 그대로 보관하는 백업 영역입니다.

- `Retrieve()` 함수가 실행될 때 가져온 모든 데이터는 Primary! 버퍼와 동시에 Original! 버퍼에도 저장됩니다.
- 사용자가 데이터를 수정해도 Original! 버퍼의 값은 절대 변하지 않습니다.
- `Update()` 실행 시 변경된 컬럼을 식별하기 위해 Primary! 버퍼의 현재 값과 Original! 버퍼의 원본 값을 비교합니다.
- 이를 통해 실제로 변경된 컬럼만 UPDATE 문에 포함시킬 수 있습니다.

### 1.5 버퍼 간 데이터 흐름 다이어그램

```
[데이터베이스]
    │
    │ Retrieve()
    ▼
[Original! 버퍼] ◄─────────────────┐ (원본 백업)
    │                              │
    │ (복사)                       │
    ▼                              │
[Primary! 버퍼] ──────────────────┤ (변경 추적)
    │            (변경 시 비교)     │
    │                              │
    ├─ DeleteRow() ──→ [Delete! 버퍼]
    │                              │
    ├─ Filter() ──────→ [Filter! 버퍼]
    │                              │
    │ Update()                    │ Update()
    ▼                              ▼
[데이터베이스 UPDATE]        [데이터베이스 DELETE]
```

---

## 2. 아이템 상태값 (Item Status)

각 버퍼에 존재하는 모든 행(Row)과 개별 컬럼은 현재 자신의 상태를 나타내는 **상태값(Status)**을 가지고 있습니다. 이 상태값은 `GetItemStatus()` 함수로 확인할 수 있으며, Update() 함수가 어떤 SQL 문을 생성할지 결정하는 핵심 기준이 됩니다.

### 2.1 행(Row) 상태값 종류

| 상태값 | 설명 | Update() 동작 |
|--------|------|---------------|
| **NotModified!** | 조회된 후 한 번도 수정되지 않은 상태 | 아무 SQL도 생성하지 않음 |
| **DataModified!** | 기존 행의 데이터가 수정된 상태 | UPDATE 문 생성 |
| **New!** | `InsertRow()`로 추가된 빈 행 (아직 값 입력 없음) | 아무 SQL도 생성하지 않음 |
| **NewModified!** | 새 행에 사용자가 값을 입력한 상태 | INSERT 문 생성 |

### 2.2 컬럼(Column) 상태값

행 상태 외에도 개별 컬럼 수준에서도 상태를 관리합니다. 컬럼 상태는 `GetItemStatus()`에 컬럼 번호를 지정하여 확인할 수 있습니다.

- **NotModified!**: 해당 컬럼이 원본에서 변경되지 않음
- **DataModified!**: 해당 컬럼이 사용자에 의해 수정됨

### 2.3 상태값 변화 과정

```powerbuilder
// 1. 데이터 조회 직후
// 모든 행: NotModified!, 모든 컬럼: NotModified!

// 2. 새 행 추가
dw_1.InsertRow(0)
// 새 행: New! (모든 컬럼: NotModified!)

// 3. 새 행에 데이터 입력
dw_1.SetItem(1, "cust_name", "홍길동")
// 행 상태: NewModified! (해당 컬럼: DataModified!)

// 4. 기존 행 데이터 수정
dw_1.SetItem(2, "phone", "02-123-4567")
// 행 상태: DataModified! (해당 컬럼: DataModified!)

// 5. 행 삭제
dw_1.DeleteRow(3)
// 행이 Delete! 버퍼로 이동, 상태는 삭제 전 상태 유지

// 6. Update() 실행 후
// 성공 시: NewModified! → NotModified!, DataModified! → NotModified!
// Delete! 버퍼의 행들은 제거됨
```

---

## 3. Update() 함수의 마법 같은 메커니즘

이제 위에서 배운 버퍼와 상태값이 실제 `Update()` 함수에서 어떻게 활용되는지 살펴보겠습니다.

### 3.1 Update() 호출 시 내부 처리 순서

`dw_1.Update()` 한 줄이 실행되면 DataWindow 엔진은 다음과 같은 순서로 자동 SQL 문을 생성하고 실행합니다.

#### 1단계: DELETE 문 생성
- **Delete! 버퍼**를 검사합니다.
- Delete! 버퍼에 행이 존재하면, 해당 행에 대한 DELETE 문을 생성하여 실행합니다.
- 이때 WHERE 절에는 원본 키 값(Original! 버퍼 기준)이 사용됩니다.

#### 2단계: INSERT 문 생성
- **Primary! 버퍼**의 행들을 검사합니다.
- 상태가 **NewModified!** 인 행을 찾습니다.
- 해당 행의 모든 컬럼(또는 Update Properties에서 지정한 컬럼)을 대상으로 INSERT 문을 생성하여 실행합니다.

#### 3단계: UPDATE 문 생성
- **Primary! 버퍼**의 행들을 다시 검사합니다.
- 상태가 **DataModified!** 인 행을 찾습니다.
- 각 행에 대해 변경된 컬럼만 식별합니다. (Primary! 버퍼 값과 Original! 버퍼 값을 비교)
- 변경된 컬럼만 포함하는 UPDATE 문을 생성하여 실행합니다.
- WHERE 절에는 원본 키 값(Original! 버퍼 기준)과 낙관적 락(Optimistic Lock)을 위한 조건이 포함될 수 있습니다.

#### 4단계: 상태값 초기화
- 모든 SQL 문이 성공적으로 실행되면 **즉시** DataWindow 내부의 상태값이 초기화됩니다.
  - NewModified! 행 → NotModified!로 변경
  - DataModified! 행 → NotModified!로 변경
  - Delete! 버퍼 비움
  - Original! 버퍼를 현재 Primary! 버퍼 값으로 업데이트

### 3.2 상태값 확인 및 디버깅

실제 코드에서 상태값을 확인하여 디버깅에 활용할 수 있습니다.

```powerbuilder
// 특정 행의 상태 확인
dw_status dws_1
dws_1 = dw_1.GetItemStatus(1, Primary!)
IF dws_1 = NewModified! THEN
   MessageBox("상태", "새로 추가된 행입니다.")
END IF

// 특정 컬럼의 상태 확인
dw_status dws_col
dws_col = dw_1.GetItemStatus(1, 2, Primary!)  // 1행, 2번째 컬럼
IF dws_col = DataModified! THEN
   MessageBox("상태", "해당 컬럼이 수정되었습니다.")
END IF

// 모든 행의 상태를 루프로 확인
Long ll_row, ll_count
ll_count = dw_1.RowCount()
FOR ll_row = 1 TO ll_count
   dws_1 = dw_1.GetItemStatus(ll_row, Primary!)
   CHOOSE CASE dws_1
      CASE NotModified!
         // 변경 없음
      CASE DataModified!
         // 수정됨
      CASE New!
         // 빈 새 행
      CASE NewModified!
         // 값이 입력된 새 행
   END CHOOSE
NEXT
```

### 3.3 Update()와 트랜잭션, 그리고 상태값 초기화의 함정

`dw_1.Update()`를 기본값(인자 없이)으로 호출하면, 데이터베이스에 쿼리를 성공적으로 던진 직후 **COMMIT 실행 여부와 상관없이 즉시 DataWindow의 상태값을 NotModified!로 초기화**합니다.

```powerbuilder
IF dw_1.Update() = 1 THEN
   // ⚠️ 이미 이 시점에서 상태값은 NotModified!로 초기화됨
   COMMIT USING SQLCA; 
ELSE
   ROLLBACK USING SQLCA;
END IF
```

이로 인해 다음과 같은 위험한 상황이 발생할 수 있습니다.

- **Update()는 성공했지만, 그 다음 줄의 COMMIT에서 네트워크 오류나 제약 조건 위반 등으로 실패하여 ROLLBACK된 경우**
- 데이터베이스는 롤백되었지만, DataWindow의 상태값은 이미 초기화되었으므로 사용자가 다시 '저장' 버튼을 눌러도 `Update()`는 "변경된 데이터가 없다"고 판단하여 아무 쿼리도 실행하지 않습니다. 결과적으로 **데이터 증발** 현상이 발생합니다.

#### 다중 DataWindow 업데이트 시 ResetUpdate 패턴

특히 하나의 트랜잭션에서 여러 개의 DataWindow(예: 마스터-디테일)를 동시에 저장해야 하는 경우, 중간에 실패하면 전체를 롤백해야 합니다. 이때 상태값 초기화를 지연시키는 기법이 필요합니다.

`Update()` 함수의 두 번째 인자(`resetflag`)를 `FALSE`로 주면, 데이터베이스 업데이트는 성공해도 상태값을 초기화하지 않습니다. 그리고 모든 업데이트가 성공한 후 수동으로 `ResetUpdate()` 함수를 호출하여 상태값을 초기화합니다.

```powerbuilder
// 헤더와 디테일을 동시에 저장하는 실무 패턴
IF dw_header.Update(TRUE, FALSE) = 1 THEN   // 상태 초기화 안 함
    IF dw_detail.Update(TRUE, FALSE) = 1 THEN   // 상태 초기화 안 함
        COMMIT USING SQLCA;
        // 커밋까지 모두 성공한 후 수동으로 상태값 초기화!
        dw_header.ResetUpdate()
        dw_detail.ResetUpdate()
    ELSE
        ROLLBACK USING SQLCA;   // 디테일 실패: 상태값이 살아있으므로 재시도 가능
    END IF
ELSE
    ROLLBACK USING SQLCA;       // 헤더 실패
END IF
```

이 패턴의 장점:
- 모든 업데이트가 성공하고 COMMIT까지 완료된 후에야 상태값을 초기화하므로 데이터 정합성이 보장됩니다.
- 중간에 실패하면 상태값이 그대로 유지되어 사용자가 다시 저장을 시도할 수 있습니다.

---

## 4. 실전 활용 예제

### 4.1 변경된 행만 추출하여 로그 남기기

```powerbuilder
// 저장 버튼 클릭 이벤트
Long ll_row, ll_count
String ls_log

ll_count = dw_1.RowCount()
FOR ll_row = 1 TO ll_count
   CHOOSE CASE dw_1.GetItemStatus(ll_row, Primary!)
      CASE DataModified!
         ls_log = "수정: 행 " + String(ll_row)
         // 변경된 컬럼 찾기
         Integer i, col_count = Long(dw_1.Describe("DataWindow.Column.Count"))
         FOR i = 1 TO col_count
            IF dw_1.GetItemStatus(ll_row, i, Primary!) = DataModified! THEN
               ls_log += ", 컬럼 " + String(i) + " 변경"
            END IF
         NEXT
         f_log_write(ls_log)
         
      CASE NewModified!
         ls_log = "추가: 행 " + String(ll_row)
         f_log_write(ls_log)
   END CHOOSE
NEXT

// 저장 실행 (단일 DataWindow이므로 일반 Update 사용)
IF dw_1.Update() = 1 THEN
   COMMIT;
   MessageBox("성공", "저장되었습니다.")
ELSE
   ROLLBACK;
   MessageBox("오류", "저장 실패")
END IF
```

### 4.2 낙관적 락(Optimistic Lock) 구현

DataWindow의 Update Properties에서 **Where Clause for Update/Delete**를 설정하면 동시성 제어가 가능합니다.

- **Key Columns**: 키 컬럼만 WHERE 절에 포함 (충돌 위험 높음)
- **Key and Updateable Columns**: 키와 업데이트 가능한 컬럼 포함 (중간)
- **Key and Modified Columns**: 키와 실제 수정된 컬럼 포함 (충돌 감지)

```powerbuilder
// 다른 사용자가 같은 데이터를 수정한 경우
// Update() 실행 시 SQLCA.SQLCode = -1, SQLDBCode = 특정 충돌 코드
IF dw_1.Update() = 1 THEN
   COMMIT;
ELSE
   IF SQLCA.SQLDBCode = 532 THEN // SQL Server 업데이트 충돌 코드
      MessageBox("충돌", "다른 사용자가 이미 데이터를 변경했습니다. 다시 조회하세요.")
      dw_1.Retrieve() // 데이터 다시 로드
   ELSE
      ROLLBACK;
   END IF
END IF
```

---

## 5. 결론

DataWindow의 4가지 버퍼와 상태값 관리 시스템은 PowerBuilder를 강력한 데이터 중심 개발 도구로 만든 핵심 기술입니다. 개발자는 복잡한 SQL 문을 직접 관리할 필요 없이, 단순히 데이터를 화면에 표시하고 사용자 입력을 받은 후 `Update()` 한 번만 호출하면 됩니다.

- **Primary! 버퍼**는 현재 데이터, **Delete! 버퍼**는 삭제된 데이터, **Filter! 버퍼**는 숨겨진 데이터, **Original! 버퍼**는 원본 데이터를 관리합니다.
- **NotModified!, DataModified!, New!, NewModified!** 상태값을 통해 각 행과 컬럼의 변경 이력을 추적합니다.
- `Update()`는 이 정보를 바탕으로 DELETE, INSERT, UPDATE 문을 자동 생성하고 순서대로 실행합니다.
- **중요한 점**: `Update()` 성공 시 상태값은 즉시 초기화되므로, COMMIT 실패 시 데이터 불일치가 발생할 수 있습니다. 다중 DataWindow 업데이트 시에는 `Update(TRUE, FALSE)`와 `ResetUpdate()` 패턴을 활용해야 합니다.

이러한 메커니즘을 이해하면 단순히 DataWindow를 사용하는 것을 넘어, 문제 발생 시 정확한 원인을 진단하고, 복잡한 비즈니스 로직에서도 DataWindow를 효과적으로 활용할 수 있습니다. 또한 상태값을 직접 확인하여 변경 내역을 추적하거나, 동시성 제어를 구현하는 등 고급 기능을 구현할 수 있습니다.
