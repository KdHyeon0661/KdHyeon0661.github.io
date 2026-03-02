---
layout: post
title: PowerBuilder - DDDW
date: 2025-12-13 19:30:23 +0900
category: PowerBuilder
---
# 파워빌더 UI의 꽃: DropDownDataWindow (DDDW) 완벽 가이드

기업용 애플리케이션을 개발하다 보면 데이터베이스에는 `01`, `02` 같은 코드(Code)를 저장하지만, 사용자 화면에는 `인사부`, `영업부` 같은 명칭(Name)으로 보여줘야 하는 경우가 많습니다. 또한 사용자가 코드 대신 명칭을 선택하면 해당 코드가 저장되어야 합니다. 이런 요구사항을 완벽히 충족시키는 파워빌더의 기능이 바로 **DropDownDataWindow(이하 DDDW)**입니다.

DDDW는 단순한 콤보박스를 넘어, DataWindow 안에 또 다른 DataWindow를 내장하는 강력한 개념입니다. 이 글에서는 DDDW의 개념부터 실무에서 가장 많이 사용되는 연관 콤보박스(Cascading DDDW) 구현까지 상세히 알아보겠습니다.

---

## 1. DDDW란 무엇인가?

DDDW는 이름 그대로 **"DataWindow 안의 특정 컬럼에 또 다른 DataWindow를 드롭다운(펼침) 형태로 집어넣은 것"**입니다. 즉, 메인 DataWindow의 특정 컬럼을 클릭하면 마치 콤보박스처럼 목록이 펼쳐지고, 그 목록은 별도의 DataWindow 객체(자식 DataWindow)로 구성됩니다.

### 왜 DDDW를 사용해야 할까?

- **코드-명칭 분리**: 데이터베이스에는 코드를 저장하고, 화면에는 명칭을 표시할 수 있습니다.
- **동적 데이터**: 목록이 데이터베이스에서 직접 조회되므로, 코드 테이블에 값이 추가/변경되면 프로그램 수정 없이 즉시 반영됩니다.
- **다중 컬럼 표시**: 일반 콤보박스는 한 줄만 표시되지만, DDDW는 그리드 형태로 여러 컬럼(코드, 명칭, 설명 등)을 동시에 보여줄 수 있습니다.

---

## 2. DropDownListBox(DDLB)와의 차이점

파워빌더에는 일반적인 콤보박스인 `DropDownListBox(DDLB)`도 존재합니다. 하지만 실무에서는 거의 모든 경우 DDDW를 사용합니다.

| 항목 | DropDownListBox (DDLB) | DropDownDataWindow (DDDW) |
|------|------------------------|---------------------------|
| **데이터 소스** | 개발자가 속성창이나 스크립트로 직접 항목을 추가 (`AddItem()`) | 데이터베이스 쿼리(SELECT)로 조회 |
| **유지보수성** | 부서나 공통코드가 추가되면 프로그램을 수정하고 재배포해야 함 | DB에 코드만 추가하면 화면에 자동 반영 |
| **표시 형태** | 한 줄의 텍스트만 표시 가능 | 여러 컬럼(코드, 명칭, 비고 등)을 그리드 형태로 표시 가능 |
| **필터링** | 직접 구현해야 함 | 자식 DataWindow에 WHERE 조건을 추가하여 동적 필터링 가능 |

결론: **DDDW가 훨씬 유연하고 강력합니다.**

---

## 3. 디자인 타임 설정: 알맹이와 껍데기 연결하기

DDDW를 구현하려면 먼저 **드롭다운용 자식 DataWindow(알맹이)**를 만들고, 이를 **메인 DataWindow(껍데기)**의 특정 컬럼에 연결해야 합니다.

### 3.1 자식(Child) DataWindow 생성

1. **파일 → New → DataWindow**에서 원하는 프레젠테이션 스타일(주로 Grid 또는 Tabular)을 선택합니다.
2. 데이터 소스로는 `department`와 같은 코드 테이블을 선택합니다.
   ```sql
   SELECT dept_id, dept_name, sort_order FROM department ORDER BY sort_order
   ```
3. 적절히 디자인한 후 이름을 `d_dddw_dept`로 저장합니다.

> 💡 **Tip**: 자식 DataWindow는 보통 코드와 명칭 두 컬럼만 있어도 충분하지만, 정렬 순서나 비고 컬럼을 추가하면 더 유용합니다.

### 3.2 메인 DataWindow에 DDDW 속성 설정

1. 메인 화면에서 사용할 DataWindow 객체(예: `d_employee`)를 엽니다.
2. 부서 코드가 저장될 컬럼(`dept_id`)을 클릭합니다.
3. 속성 창에서 **Edit** 탭을 선택합니다.
4. **Style Type**을 `DropDownDW`로 변경합니다.
5. 아래 세 가지 핵심 속성을 매핑합니다.

| 속성 | 설명 | 예시 값 |
|------|------|---------|
| **DataWindow** | 앞서 만든 자식 DataWindow 객체 | `d_dddw_dept` |
| **Display Column** | 사용자에게 보여줄 명칭 컬럼 | `dept_name` |
| **Data Column** | 실제 DB에 저장될 코드 컬럼 | `dept_id` |

6. 기타 옵션:
   - **AutoRetrieve**: 메인 DataWindow가 데이터를 조회할 때 자식 DataWindow도 자동으로 조회할지 여부. (보통 FALSE로 두고 코드에서 명시적으로 처리)
   - **Lines in DropDown**: 드롭다운 목록에 표시할 행 수
   - **Width of DropDown**: 드롭다운 목록의 너비 (0이면 컬럼 너비와 동일)

---

## 4. [핵심] 런타임 제어와 `GetChild` 함수

디자인 타임에 아무리 잘 설정해도, 화면을 실행하면 드롭다운 목록이 텅 비어 있는 경우가 많습니다. 그 이유는 자식 DataWindow도 데이터를 가져오려면 별도로 `Retrieve()`를 호출해야 하기 때문입니다.

이때 필요한 함수가 바로 **`GetChild()`**입니다. `GetChild()`는 메인 DataWindow에 내장된 자식 DataWindow의 참조를 얻어오는 함수입니다.

### 4.1 기본 사용법

```powerbuilder
// 메인 윈도우의 Open 이벤트

DataWindowChild ldwc_dept   // 자식 DataWindow를 담을 변수 (반드시 DataWindowChild 타입)
Integer li_rtn

// 1. 메인 DataWindow(dw_emp)의 "dept_id" 컬럼에 연결된 DDDW 참조 가져오기
li_rtn = dw_emp.GetChild("dept_id", ldwc_dept)

IF li_rtn = 1 THEN
    // 2. 자식 DataWindow에 트랜잭션 연결 후 데이터 조회
    ldwc_dept.SetTransObject(SQLCA)
    ldwc_dept.Retrieve()
ELSE
    MessageBox("오류", "DDDW 참조를 가져올 수 없습니다.")
END IF

// 3. 메인 DataWindow 조회 (자식 조회 후에 실행!)
dw_emp.SetTransObject(SQLCA)
dw_emp.Retrieve()
```

### 4.2 ⭐ 조회 순서의 중요성

**반드시 자식 DataWindow를 먼저 Retrieve하고, 그 다음에 메인 DataWindow를 Retrieve해야 합니다.** 순서가 바뀌면 메인 DataWindow가 데이터를 가져올 때 자식 DataWindow의 명칭을 매핑할 수 없어 코드가 그대로 표시되는 버그가 발생합니다.

### 4.3 `AutoRetrieve` 속성

자식 DataWindow의 속성에는 **AutoRetrieve**라는 옵션이 있습니다. 이 옵션을 TRUE로 설정하면 메인 DataWindow가 `Retrieve()`될 때 자동으로 자식 DataWindow도 함께 조회됩니다. 하지만 실무에서는 다음과 같은 이유로 FALSE로 설정하고 명시적으로 처리하는 경우가 많습니다.

- 조회 순서를 명확히 제어해야 할 때
- 자식 DataWindow에 파라미터가 필요할 때
- 성능상 이유로 자식 DataWindow의 조회 시점을 분리해야 할 때

---

## 5. 실무 꿀팁: 대분류 ➡️ 소분류 연관 콤보박스 (Cascading DDDW)

실무에서 가장 많이 요구되는 기능은 **"앞의 콤보박스(대분류)를 선택하면, 뒤의 콤보박스(소분류)의 목록이 그에 맞게 동적으로 바뀌는 기능"**입니다. 예를 들어, '카테고리'를 선택하면 해당 카테고리에 속한 '상품'만 드롭다운에 표시하는 경우입니다.

### 5.1 구현 원리

1. 소분류 자식 DataWindow는 대분류 코드를 파라미터로 받도록 설계합니다.
2. 메인 DataWindow의 `ItemChanged` 이벤트에서 대분류 값이 변경되면, 소분류 DDDW의 참조를 얻어와 파라미터를 전달하며 다시 `Retrieve()`합니다.
3. 기존에 선택된 소분류 값은 초기화합니다.

### 5.2 소분류 자식 DataWindow 생성

소분류용 DataWindow(`d_dddw_product`)를 만들 때, 데이터 소스에 파라미터를 추가합니다.

```sql
SELECT product_id, product_name 
FROM product 
WHERE category_id = :category_id   -- 파라미터
ORDER BY product_name
```

### 5.3 메인 DataWindow 디자인

메인 DataWindow(`d_order`)에 두 개의 컬럼을 만듭니다.
- `category_id` (대분류, DDDW 연결)
- `product_id` (소분류, DDDW 연결)

### 5.4 `ItemChanged` 이벤트 구현

```powerbuilder
// 메인 DataWindow(dw_order)의 ItemChanged 이벤트
// 파라미터: dwo(변경된 컬럼 정보), data(새로 입력된 값)

DataWindowChild ldwc_product
String ls_category

// 변경된 컬럼이 대분류(category_id)인지 확인
IF dwo.Name = "category_id" THEN
    ls_category = data   // 사용자가 선택한 대분류 코드
    
    // 소분류 DDDW의 참조 가져오기
    IF This.GetChild("product_id", ldwc_product) = 1 THEN
        ldwc_product.SetTransObject(SQLCA)
        // 대분류 코드를 파라미터로 전달하여 소분류 목록 다시 조회
        ldwc_product.Retrieve(ls_category)
    END IF
    
    // 기존에 선택된 소분류 값이 있으면 초기화
    This.SetItem(row, "product_id", "")
END IF
```

### 5.5 윈도우 Open 이벤트

```powerbuilder
// 윈도우 Open 이벤트
DataWindowChild ldwc_category, ldwc_product

// 대분류 DDDW 초기화
dw_order.GetChild("category_id", ldwc_category)
ldwc_category.SetTransObject(SQLCA)
ldwc_category.Retrieve()

// 소분류 DDDW 초기화 (처음에는 전체 목록 또는 빈 목록)
dw_order.GetChild("product_id", ldwc_product)
ldwc_product.SetTransObject(SQLCA)
ldwc_product.Retrieve()   // 파라미터 없이 조회 (기본값 또는 전체)

// 메인 DataWindow 조회
dw_order.SetTransObject(SQLCA)
dw_order.Retrieve()
```

> 💡 **Tip**: 소분류 초기 조회 시 파라미터 없이 조회하면 모든 상품이 나올 수 있습니다. 이를 방지하려면 기본값을 위한 파라미터를 전달하거나, 아예 빈 결과를 반환하도록 설계할 수 있습니다.

---

## 6. DDDW 관련 고급 팁

### 6.1 코드와 명칭 매핑 확인

메인 DataWindow에서 코드는 저장되지만 화면에 명칭이 표시되는 원리는 다음과 같습니다.
- 메인 DataWindow의 컬럼은 실제로 코드 값을 가지고 있습니다.
- 하지만 DDDW 설정 덕분에 화면 표시 시 Display Column의 값이 보여집니다.
- `GetItemString()`으로 값을 읽으면 실제 코드 값이 반환됩니다.

### 6.2 DDDW 정렬 및 필터링

자식 DataWindow 자체에 정렬과 필터를 적용할 수 있습니다. 사용자가 드롭다운을 열었을 때 정렬된 목록을 보여주려면 자식 DataWindow의 `SetSort()`와 `Sort()`를 호출하면 됩니다.

```powerbuilder
ldwc_dept.SetSort("dept_name A")
ldwc_dept.Sort()
```

### 6.3 DDDW에서 선택한 값의 추가 정보 가져오기 (정확한 행 찾기)

사용자가 드롭다운에서 선택한 행의 다른 컬럼 정보를 가져와야 할 때가 많습니다. 예를 들어, 상품 선택 시 가격을 자동으로 화면에 채우는 경우입니다. 이때 주의할 점은, 메인 DataWindow의 `ItemChanged` 이벤트가 발생하는 시점에 자식 DataWindow(DDDW)의 내부 행 포인터(`GetRow()`)가 사용자가 방금 선택한 행을 정확히 가리킨다는 보장이 없다는 것입니다. 종종 첫 번째 행을 가리키는 오류가 발생할 수 있습니다.

따라서 **사용자가 선택한 값(`data` 인자)을 기준으로 자식 DataWindow에서 `Find()` 함수를 사용하여 정확한 행 번호를 찾은 후**, 필요한 정보를 가져와야 합니다. 이 패턴이 실무에서 검증된 안전한 방법입니다.

```powerbuilder
// 메인 DataWindow(dw_order)의 ItemChanged 이벤트
// dwo : 변경된 컬럼 정보
// data: 새로 입력된 값 (사용자가 선택한 코드)

IF dwo.Name = "product_id" THEN
    DataWindowChild ldwc_product
    Long ll_find_row
    
    // 1. 소분류 DDDW의 참조 가져오기
    IF This.GetChild("product_id", ldwc_product) = 1 THEN
        ldwc_product.SetTransObject(SQLCA)  // 이미 설정되어 있다면 생략 가능
        
        // 2. 사용자가 선택한 값(data)으로 자식 DW에서 정확한 행 찾기
        // (data가 문자열인 경우 홑따옴표로 감싸야 함)
        ll_find_row = ldwc_product.Find("product_id = '" + data + "'", 1, ldwc_product.RowCount())
        
        IF ll_find_row > 0 THEN
            // 3. 찾은 행에서 추가 정보(가격) 가져와서 메인 DW의 다른 컬럼에 설정
            Decimal ld_price = ldwc_product.GetItemDecimal(ll_find_row, "price")
            This.SetItem(row, "price", ld_price)
        END IF
    END IF
END IF
```

> **⚠️ 중요**: `Find()` 함수의 검색 조건 문자열에서 값 부분은 반드시 컬럼 타입에 맞게 처리해야 합니다. 문자열인 경우 위 예시처럼 홑따옴표로 감싸고, 숫자형이면 따옴표 없이 사용합니다.

이 방법을 사용하면 DDDW의 선택값과 관련된 추가 정보를 안정적으로 가져올 수 있습니다.

---

## 7. 결론

DropDownDataWindow(DDDW)는 파워빌더 개발자에게 필수적인 도구입니다. 단순한 콤보박스를 넘어 데이터베이스와 완벽하게 통합된 동적 목록을 제공하며, 사용자 경험을 크게 향상시킵니다.

- **DDDW**는 코드-명칭 분리, 실시간 반영, 다중 컬럼 표시 등에서 DDLB를 압도합니다.
- **GetChild()** 함수는 자식 DataWindow에 접근하는 핵심 열쇠입니다.
- **ItemChanged 이벤트**와 결합하면 연관 콤보박스(Cascading DDDW)도 쉽게 구현할 수 있습니다.
- 선택된 값의 추가 정보를 가져올 때는 **`Find()`를 사용하여 정확한 행을 찾는 패턴**을 반드시 따라야 합니다.

이제 DDDW의 개념과 활용법을 마스터했다면, 어떤 복잡한 UI 요구사항도 자신 있게 해결할 수 있을 것입니다. 실무에서 DDDW를 자유자재로 다루는 개발자로 한 걸음 더 나아가시길 바랍니다.