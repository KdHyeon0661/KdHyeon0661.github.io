---
layout: post
title: PowerBuilder - DataStore
date: 2025-12-13 18:30:23 +0900
category: PowerBuilder
---
# DataStore (데이터스토어)의 개념과 활용

PowerBuilder 개발에서 DataWindow는 화면에 데이터를 표시하는 핵심 컨트롤이지만, 때로는 **화면에 보이지 않으면서 데이터를 처리해야 하는 상황**이 발생합니다. 예를 들어, 백그라운드에서 집계 계산을 하거나, 여러 DataWindow 간에 데이터를 공유하거나, 보고서를 생성할 때 UI에 표시할 필요가 없는 데이터 처리가 필요합니다. 이때 사용하는 것이 바로 **DataStore**입니다.

---

## DataStore란?

DataStore는 DataWindow와 거의 동일한 기능을 가지지만 **시각적이지 않은(Non-Visual)** 객체입니다. 즉, 화면에 보이지 않고, 사용자 입력을 받지 않으며, 오직 데이터 처리와 조작을 목적으로 합니다.

### DataWindow vs DataStore 비교

| 특징 | DataWindow | DataStore |
|------|------------|-----------|
| 화면 표시 | O (컨트롤로 배치) | X (비시각적) |
| 사용자 입력 | O | X |
| 데이터 처리 | O | O |
| Retrieve/Update | O | O |
| 공유(ShareData) | 대상 가능 | 대상 가능 |
| 이벤트 | 모든 시각적 이벤트 | 제한적 (일부 이벤트만) |
| 메모리 | 윈도우 핸들 필요 | 경량 |

DataStore는 DataWindow와 동일한 DataWindow 객체(DataObject)를 사용하므로, 이미 디자인한 DataWindow 객체를 그대로 활용할 수 있습니다.

---

## DataStore 생성 및 기본 사용법

### 1. DataStore 변수 선언 및 생성

DataStore는 비시각적 객체이므로 `CREATE` 문으로 직접 생성해야 합니다. 사용 후에는 반드시 `DESTROY`로 소멸시켜 메모리 누수를 방지합니다.

```powerbuilder
DataStore lds_customer
lds_customer = CREATE DataStore
lds_customer.DataObject = "d_customer_list"   // 미리 만든 DataWindow 객체
lds_customer.SetTransObject(SQLCA)            // 트랜잭션 연결
```

### 2. 데이터 조회 (Retrieve)

```powerbuilder
Long ll_rows
ll_rows = lds_customer.Retrieve()
IF ll_rows < 0 THEN
   MessageBox("오류", "조회 실패")
END IF
```

### 3. 데이터 접근

DataStore에 저장된 데이터는 DataWindow와 동일한 방식으로 접근합니다.

```powerbuilder
// 특정 행, 컬럼 값 읽기
String ls_name = lds_customer.GetItemString(1, "cust_name")
Decimal ld_price = lds_customer.GetItemDecimal(1, "price")

// 값 변경 (메모리 상에서만 변경, DB 업데이트는 Update() 호출 필요)
lds_customer.SetItem(1, "cust_name", "홍길동")
```

### 4. 데이터 업데이트 (Update)

DataStore의 변경 내용을 데이터베이스에 반영하려면 `Update()` 함수를 호출합니다.

```powerbuilder
IF lds_customer.Update() = 1 THEN
   COMMIT USING SQLCA;
ELSE
   ROLLBACK USING SQLCA;
END IF
```

### 5. DataStore 소멸

```powerbuilder
DESTROY lds_customer
```

---

## DataStore의 주요 활용 사례

### 1. 백그라운드 집계 계산

화면에 표시되는 DataWindow는 간단한 목록만 보여주고, 통계 정보는 별도의 DataStore로 계산하여 화면의 StaticText나 Compute Field에 표시할 수 있습니다.

**예제: 주문 목록의 총 합계 계산**

```powerbuilder
// DataStore 생성 및 주문 데이터 조회
DataStore lds_orders
lds_orders = CREATE DataStore
lds_orders.DataObject = "d_orders"
lds_orders.SetTransObject(SQLCA)
lds_orders.Retrieve()

// 총 합계 계산 (DataStore의 Compute Field 활용 또는 직접 계산)
Decimal ld_total = 0
Long i, ll_count = lds_orders.RowCount()
FOR i = 1 TO ll_count
   ld_total += lds_orders.GetItemDecimal(i, "amount")
NEXT

// 화면의 StaticText에 표시
st_total.Text = "총 매출: " + String(ld_total, "#,##0")

// 사용 후 소멸
DESTROY lds_orders
```

### 2. DataWindow 간 데이터 공유 (ShareData)

여러 DataWindow(또는 DataStore)가 동일한 데이터를 공유해야 할 때 `ShareData()` 함수를 사용합니다. 이렇게 하면 하나의 DataWindow/DataStore가 데이터를 가지고 있으면 다른 DataWindow/DataStore는 동일한 데이터를 화면에 표시만 하거나 별도로 가공할 수 있습니다. 데이터의 일관성을 유지하고 메모리를 절약할 수 있습니다.

**예제: 하나의 DataStore를 두 개의 DataWindow에서 공유**

```powerbuilder
// 공통 데이터를 가진 DataStore
DataStore lds_common
lds_common = CREATE DataStore
lds_common.DataObject = "d_employee"
lds_common.SetTransObject(SQLCA)
lds_common.Retrieve()

// 두 개의 DataWindow 컨트롤에 데이터 공유
dw_list.ShareData(lds_common)
dw_detail.ShareData(lds_common)   // 같은 데이터를 다른 형식으로 표시 가능

// 주의: dw_list와 dw_detail은 데이터를 공유하므로 한쪽에서 데이터를 수정하면 다른 쪽에도 반영됨
// 정렬이나 필터는 각 DataWindow마다 독립적으로 적용 가능
dw_list.SetSort("name A")
dw_list.Sort()
dw_detail.SetFilter("dept = '영업'")
dw_detail.Filter()
```

> **💡 핵심 팁: ShareData의 강력함**  
> 서로 다른 DataObject라도 데이터를 공유할 수 있습니다. 단, **내부의 결과 집합(Result Set: 컬럼의 개수, 순서, 데이터 타입)이 완벽하게 동일해야만** 에러 없이 공유됩니다. 이 특징 덕분에 백그라운드에 DataStore 하나만 띄워놓고, 화면에는 **Grid 스타일(목록용)**, **Freeform 스타일(상세용)**, **Graph 스타일(차트용)** 등 서로 다른 디자인의 DataWindow에 데이터를 쫙 뿌려주는 진정한 MVC(Model-View-Controller) 패턴 구현이 가능합니다.

### 3. 데이터 내보내기 (Export)

DataStore는 화면에 보이지 않으므로 파일 내보내기(Excel, CSV, 텍스트 등)에 매우 적합합니다.

```powerbuilder
// CSV 파일로 저장
lds_customer.SaveAs("C:\temp\customer.csv", CSV!, TRUE)

// 최신 엑셀 파일(.xlsx)로 저장 (PowerBuilder 12.5 이상)
lds_customer.SaveAs("C:\temp\customer.xlsx", XLSX!, TRUE)

// 구형 엑셀 파일(.xls)로 저장 (호환성 필요 시)
lds_customer.SaveAs("C:\temp\customer.xls", Excel8!, TRUE)
```

### 4. 데이터 복사 및 이동

DataStore 간에 데이터를 복사하거나 이동할 수 있습니다. `RowsCopy()`, `RowsMove()` 함수를 사용합니다.

```powerbuilder
// lds_source의 1~10행을 lds_target의 끝에 복사
lds_source.RowsCopy(1, 10, Primary!, lds_target, lds_target.RowCount() + 1, Primary!)
```

### 5. 보고서 생성

프린트하거나 PDF로 출력할 때 화면에 표시할 필요 없이 DataStore로 데이터를 준비하고 `Print()` 함수를 사용할 수 있습니다.

```powerbuilder
// DataStore의 내용을 프린터로 직접 출력
lds_report.Print()
```

---

## DataStore와 UI 연동 실전 예제

### 시나리오
- 고객 목록을 보여주는 메인 화면(`w_customer`)이 있습니다.
- 화면에는 DataWindow 컨트롤(`dw_list`)로 고객 목록이 표시됩니다.
- 백그라운드에서 선택된 고객의 주문 내역을 집계하여 화면 아래에 표시하려고 합니다.
- 또한, 고객 등급별 통계를 별도로 계산하여 다른 컨트롤에 표시합니다.

### 1. 윈도우 오픈 시 DataStore 준비

```powerbuilder
// 윈도우 인스턴스 변수 선언
DataStore ids_orders

// w_customer의 Open 이벤트
ids_orders = CREATE DataStore
ids_orders.DataObject = "d_orders"
ids_orders.SetTransObject(SQLCA)

// 메인 DataWindow 조회
dw_list.SetTransObject(SQLCA)
dw_list.Retrieve()
```

### 2. 고객 선택 시 해당 고객의 주문 집계

```powerbuilder
// dw_list의 Clicked 이벤트 (또는 RowFocusChanged)
Long ll_sel_row = dw_list.GetRow()
IF ll_sel_row > 0 THEN
   String ls_cust_id = dw_list.GetItemString(ll_sel_row, "cust_id")
   
   // DataStore에서 해당 고객의 주문 조회
   ids_orders.Retrieve(ls_cust_id)   // d_orders에 파라미터 cust_id가 있다고 가정
   
   // 주문 건수 및 총 금액 계산
   Long ll_order_count = ids_orders.RowCount()
   Decimal ld_total = 0
   FOR i = 1 TO ll_order_count
      ld_total += ids_orders.GetItemDecimal(i, "amount")
   NEXT
   
   // 화면에 표시
   st_order_info.Text = "주문 건수: " + String(ll_order_count) + "건, 총 금액: " + String(ld_total, "#,##0")
END IF
```

### 3. 고객 등급별 통계 (백그라운드에서 미리 계산)

```powerbuilder
// 윈도우 오픈 이벤트에서 추가로 등급별 통계 DataStore 생성 및 계산
DataStore lds_stats
lds_stats = CREATE DataStore
lds_stats.DataObject = "d_customer_stats"   // 등급별 통계를 위한 DataWindow
lds_stats.SetTransObject(SQLCA)
lds_stats.Retrieve()   // SQL에서 GROUP BY 등으로 통계를 가져옴

// 화면에 통계 표시 (예: 리스트박스나 StaticText)
// 간단히 첫 번째 등급의 고객 수 표시
IF lds_stats.RowCount() > 0 THEN
   st_gold_count.Text = "골드: " + String(lds_stats.GetItemLong(1, "gold_count"))
END IF

DESTROY lds_stats   // 사용 후 정리
```

### 4. 윈도우 닫힐 때 DataStore 정리

```powerbuilder
// w_customer의 Close 이벤트
DESTROY ids_orders
```

---

## DataStore 사용 시 주의사항

### 1. 메모리 관리
DataStore는 `CREATE`로 생성한 후 반드시 `DESTROY`로 해제해야 합니다. 그렇지 않으면 메모리 누수가 발생하여 애플리케이션 성능 저하의 원인이 됩니다.

### 2. 트랜잭션 객체 공유
DataStore도 DataWindow와 동일하게 `SetTransObject()`로 트랜잭션 객체를 연결해야 합니다. 보통 SQLCA를 사용합니다.

### 3. 이벤트 처리
DataStore는 시각적 이벤트(Clicked 등)가 없지만, `DBError`, `RetrieveStart`, `RetrieveEnd`, `RetrieveRow`, `SQLPreview` 등의 이벤트는 발생합니다. 필요한 경우 DataStore 변수를 선언한 객체에서 이벤트를 처리할 수 있습니다.

### 4. ShareData 사용 시 제약
`ShareData()`로 연결된 DataWindow/DataStore는 각자 독립적인 필터, 정렬, 포맷을 가질 수 있지만, 데이터 수정은 공유되므로 한쪽에서 업데이트하면 모두 변경됩니다. 또한, 앞서 언급했듯이 **서로 다른 DataObject도 결과 집합의 구조(컬럼 순서, 타입, 개수)가 동일하다면 공유 가능합니다.**

### 5. 다중 행 조작
DataStore는 화면에 보이지 않지만, 내부적으로 행을 가지고 있으므로 `InsertRow()`, `DeleteRow()`, `Update()` 등 DataWindow의 모든 행 조작 함수를 사용할 수 있습니다.

---

## 실무 꿀팁

### 1. 전역 DataStore 사용
애플리케이션 전체에서 공통으로 사용할 코드성 데이터(공통코드, 부서목록 등)는 전역 DataStore로 관리하면 편리합니다. 애플리케이션 오픈 시 미리 조회해두고 필요할 때마다 사용합니다.

### 2. DataStore로 엑셀 다운로드 구현
사용자가 조회 조건을 선택하면 화면의 DataWindow에 결과를 보여주고, "엑셀 다운로드" 버튼을 누르면 동일한 DataObject를 가진 DataStore를 새로 생성하여 전체 데이터를 조회한 후 `SaveAs`로 저장합니다. 이때 화면의 DataWindow는 페이징되거나 필터링된 상태일 수 있으므로, DataStore로 전체 데이터를 다시 조회하는 것이 안전합니다.

### 3. 백그라운드 계산 전용 DataStore
복잡한 집계가 필요할 때, 화면의 DataWindow에 Compute Field를 많이 넣으면 성능이 저하될 수 있습니다. 대신, DataStore에서 데이터를 조회한 후 PowerScript로 계산하고 결과만 화면에 표시하는 방식이 더 효율적일 수 있습니다.

---

## 결론

DataStore는 PowerBuilder에서 **눈에 보이지 않는 데이터 처리 엔진**으로, DataWindow의 모든 데이터 처리 능력을 그대로 유지하면서도 UI의 제약에서 자유롭습니다. 화면에 표시할 필요가 없는 데이터 집계, 다중 DataWindow 간 데이터 공유, 파일 내보내기, 백그라운드 계산 등 다양한 실무 상황에서 필수적으로 활용됩니다.

DataStore를 적절히 사용하면 코드의 중복을 줄이고, 성능을 최적화하며, 사용자 인터페이스와 비즈니스 로직을 명확히 분리할 수 있습니다. 특히 `ShareData()`를 이용한 데이터 공유 기법은 여러 DataWindow가 동일한 데이터를 다양한 방식으로 보여줘야 할 때 매우 유용하며, 서로 다른 DataObject 간 공유가 가능하다는 점은 PowerBuilder의 강력한 유연성을 보여줍니다.

이제 화면 UI와 DataStore를 결합하여 데이터를 자유자재로 가공하는 능력을 갖추었다면, PowerBuilder를 이용한 고급 비즈니스 애플리케이션 개발에 한 걸음 더 다가갈 수 있을 것입니다.
