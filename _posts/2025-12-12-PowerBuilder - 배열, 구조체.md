---
layout: post
title: PowerBuilder - 배열, 구조체
date: 2025-12-12 19:30:23 +0900
category: PowerBuilder
---
# 배열, 구조체(Structure)

PowerBuilder에서 데이터를 효율적으로 관리하기 위해 배열과 구조체를 사용합니다. 배열은 동일한 데이터 타입의 여러 값을 하나의 변수로 묶어 처리할 수 있게 해주고, 구조체는 서로 다른 데이터 타입의 항목들을 하나의 단위로 그룹화하여 복잡한 데이터를 체계적으로 표현합니다. 이 글에서는 배열과 구조체의 개념부터 선언, 사용 방법, 고급 기능까지 자세히 알아보겠습니다.

---

## 배열 (Array)

배열은 같은 데이터 타입의 요소들이 인덱스로 구분되는 자료 구조입니다. PowerBuilder의 배열은 다른 언어와 달리 기본 인덱스가 1부터 시작하며, 정적 배열과 동적 배열을 모두 지원합니다.

### 1차원 배열

#### 정적 배열 선언

크기가 고정된 배열을 선언할 때는 배열 크기를 명시합니다.

```
데이터타입 배열이름[크기]
```

예제:
```
Integer li_scores[10]        // 10개의 정수를 저장하는 배열
String ls_names[5]           // 5개의 문자열을 저장하는 배열
```

배열 요소에 접근할 때는 인덱스를 사용합니다.

```
li_scores[1] = 95            // 첫 번째 요소에 값 할당
li_scores[2] = 87
ls_names[1] = "홍길동"

MessageBox("첫 번째 학생", String(li_scores[1]) + " " + ls_names[1])
```

#### 동적 배열 선언

크기를 미리 지정하지 않고, 실행 중에 필요에 따라 크기가 변하는 배열입니다. 대괄호 안을 비워둡니다.

```
데이터타입 배열이름[]
```

예제:
```
Integer li_dynamic[]         // 빈 동적 배열
String ls_dynamic[]
```

동적 배열에 값을 할당하면 인덱스에 따라 자동으로 크기가 조정됩니다.

```
li_dynamic[1] = 100          // 첫 번째 요소 할당 → 배열 크기 1
li_dynamic[3] = 300          // 세 번째 요소 할당 → 크기 3 (2번 요소는 기본값 0)
```

중간에 건너뛴 인덱스는 해당 데이터 타입의 기본값으로 채워집니다.

#### 배열 초기화

선언과 동시에 초기값을 지정할 수 있습니다.

```
Integer li_numbers[5] = {10, 20, 30, 40, 50}
String ls_colors[3] = {"Red", "Green", "Blue"}
```

동적 배열도 초기화 가능합니다.

```
Integer li_values[] = {1, 2, 3, 4, 5}   // 크기 5의 동적 배열
```

#### 배열의 크기 확인

`UpperBound()` 함수와 `LowerBound()` 함수를 사용하여 배열의 상한과 하한을 알 수 있습니다. 1차원 배열의 경우 `UpperBound()`가 요소 개수를 의미합니다.

```
Integer li_scores[10]
Integer li_count

li_count = UpperBound(li_scores)   // 결과: 10

// 동적 배열의 경우
Integer li_dynamic[] = {1, 2, 3}
li_count = UpperBound(li_dynamic)  // 결과: 3
```

### 2차원 배열

행과 열로 구성된 배열입니다.

#### 정적 2차원 배열 선언

```
Integer li_matrix[5, 10]        // 5행 10열의 2차원 배열
```

#### 동적 2차원 배열 선언

```
Integer li_matrix[,]            // 2차원 동적 배열
```

#### 값 할당 및 참조

```
li_matrix[2, 3] = 100           // 2행 3열에 값 할당
Integer li_val = li_matrix[2, 3]
```

#### 2차원 배열의 크기 확인

`UpperBound()`에 차원을 지정합니다.

```
Integer li_rows, li_cols
li_rows = UpperBound(li_matrix, 1)   // 행의 개수
li_cols = UpperBound(li_matrix, 2)   // 열의 개수
```

### 다차원 배열

3차원 이상의 배열도 지원합니다.

```
Integer li_cube[3, 3, 3]        // 3x3x3 3차원 배열
Integer li_dynamic[,,]          // 3차원 동적 배열
```

### 배열 관련 내장 함수

| 함수 | 설명 | 예시 |
|------|------|------|
| `LowerBound(array[, n])` | 배열의 지정된 차원 하한 반환 | `LowerBound(li_scores)` → 1 |
| `UpperBound(array[, n])` | 배열의 지정된 차원 상한 반환 | `UpperBound(li_scores, 2)` |
| `ArrayLen(array)` | 배열의 전체 요소 개수 반환 (1차원만) | `ArrayLen(li_dynamic)` |

### 배열 복사

배열 변수를 다른 변수에 대입하면 참조가 아닌 실제 값이 복사됩니다.

```
Integer li_source[] = {1, 2, 3}
Integer li_target[]

li_target = li_source           // li_target은 {1, 2, 3}을 가진 별도의 배열
li_target[1] = 100              // li_source[1]은 여전히 1
```

### 가변 길이 배열의 동적 확장 (성능 고려)

동적 배열은 인덱스에 값을 할당할 때마다 자동으로 크기가 조정됩니다. 하지만 대량의 데이터를 루프를 돌며 1건씩 추가하면 매번 메모리를 재할당하므로 성능이 저하될 수 있습니다.

이럴 때는 배열의 마지막 예상 인덱스에 더미(Dummy) 값을 먼저 하나 넣어주면, 한 번에 해당 크기만큼 메모리가 확보되어 처리 속도가 훨씬 빨라집니다.

```
Integer li_array[]
Integer li_i

// 1000개의 공간을 한 번에 확보 (마지막 인덱스에 더미 값 할당)
li_array[1000] = 0

// 이제 1부터 999까지 값을 채워도 메모리 재할당이 발생하지 않음
FOR li_i = 1 TO 999
    li_array[li_i] = li_i * 10
NEXT
```

> **참고**: PowerBuilder에는 배열 크기를 명시적으로 설정하는 내장 함수(예: SetLength)가 없습니다. 위와 같은 방식으로 필요한 크기를 미리 확보할 수 있습니다.

### 실전 예제: 학생 점수 처리

```
// 학생 5명의 점수를 저장하는 배열
Integer li_scores[5] = {85, 92, 78, 94, 88}
Integer li_i, li_sum = 0, li_max = 0, li_min = 999

// 총점, 최고점, 최저점 계산
FOR li_i = 1 TO 5
    li_sum += li_scores[li_i]
    IF li_scores[li_i] > li_max THEN li_max = li_scores[li_i]
    IF li_scores[li_i] < li_min THEN li_min = li_scores[li_i]
NEXT

MessageBox("결과", "총점: " + String(li_sum) + "~r~n평균: " + String(li_sum / 5) + "~r~n최고: " + String(li_max) + "~r~n최저: " + String(li_min))
```

---

## 구조체 (Structure)

구조체는 서로 다른 데이터 타입의 항목들을 하나의 단위로 묶는 사용자 정의 데이터 타입입니다. 레코드(Record)라고도 하며, 관련된 여러 정보를 하나의 객체처럼 다룰 수 있습니다.

### 구조체의 정의 위치

PowerBuilder에서는 구조체를 정의할 수 있는 위치에 따라 두 가지 유형이 있습니다.

- **전역 구조체 (Global Structure)**: 애플리케이션 전체에서 사용할 수 있는 구조체. 별도의 Structure Painter에서 정의합니다.
- **객체 수준 구조체 (Object-Level Structure)**: 특정 윈도우, 사용자 객체 등에 종속된 구조체. 해당 객체 내에서만 사용 가능합니다.

### 전역 구조체 생성 방법

1. **Structure Painter 열기**
   - 메뉴: **File → New**
   - **PB Object** 탭에서 **Structure** 아이콘 선택

2. **구조체 정의**
   Structure Painter가 열리면 다음과 같이 필드를 정의합니다.

   ```
   [구조체 정의 화면]
   ┌─────────────────────────────────┐
   │ Structure Name: s_customer       │
   ├─────────────────────────────────┤
   │ Type         Variable Name       │
   │ ─────────────────────────────── │
   │ String       cust_id             │
   │ String       cust_name           │
   │ String       phone               │
   │ Date         birth_date          │
   │ Decimal{2}   total_purchase      │
   │ Boolean      is_active           │
   └─────────────────────────────────┘
   ```

3. **저장**
   정의한 이름(예: `s_customer`)으로 저장합니다.

### 객체 수준 구조체 생성 방법

1. 해당 객체(윈도우, 사용자 객체)를 Painter로 엽니다.
2. **Declare → Window Structures** (윈도우의 경우) 선택
3. 구조체 이름과 필드를 정의합니다.

### 구조체 변수 선언 및 사용

#### 구조체 변수 선언

```
s_customer lstr_cust1, lstr_cust2   // s_customer 타입의 변수 선언
```

#### 필드 접근

점(`.`) 연산자를 사용하여 각 필드에 접근합니다.

```
lstr_cust1.cust_id = "C001"
lstr_cust1.cust_name = "홍길동"
lstr_cust1.phone = "02-1234-5678"
lstr_cust1.birth_date = 1990-01-15
lstr_cust1.total_purchase = 1500000
lstr_cust1.is_active = TRUE

MessageBox("고객명", lstr_cust1.cust_name)
```

#### 구조체 전체 복사

구조체 변수끼리 대입하면 모든 필드가 복사됩니다.

```
lstr_cust2 = lstr_cust1   // lstr_cust2는 lstr_cust1의 모든 값을 가짐
```

### 구조체 배열

구조체의 배열을 선언하여 여러 개의 레코드를 관리할 수 있습니다.

```
s_customer lstr_cust_list[100]        // 정적 배열
s_customer lstr_cust_dyn[]             // 동적 배열
```

사용 예:

```
// 동적 배열에 데이터 추가
Integer li_count = 1
lstr_cust_dyn[li_count].cust_id = "C001"
lstr_cust_dyn[li_count].cust_name = "홍길동"
li_count++

lstr_cust_dyn[li_count].cust_id = "C002"
lstr_cust_dyn[li_count].cust_name = "김철수"

// 배열 순회
FOR li_i = 1 TO UpperBound(lstr_cust_dyn)
    MessageBox("고객", lstr_cust_dyn[li_i].cust_name)
NEXT
```

### 구조체의 중첩

구조체의 필드로 다른 구조체를 사용할 수 있습니다.

```
// 주소 구조체 정의
Global Type s_address
    String street
    String city
    String zipcode
End Type

// 고객 구조체에 주소 포함
Global Type s_customer
    String cust_id
    String cust_name
    s_address address       // 중첩 구조체
End Type
```

사용:

```
s_customer lstr_cust
lstr_cust.cust_id = "C001"
lstr_cust.address.city = "서울"
lstr_cust.address.zipcode = "12345"
```

### 구조체와 함수

함수의 인자나 반환 타입으로 구조체를 사용할 수 있습니다.

```
// 전역 함수: 고객 정보 출력
FUNCTION String f_cust_string (s_customer cust)
    RETURN cust.cust_id + ": " + cust.cust_name + " (" + cust.phone + ")"
END FUNCTION

// 호출
String ls_info = f_cust_string(lstr_cust1)
```

### 구조체와 DataWindow

DataWindow의 데이터를 구조체 배열로 가져오거나, 구조체 배열을 DataWindow에 설정할 수 있습니다.

#### DataWindow 데이터를 구조체 배열로 가져오기

```
s_customer lstr_cust[]
Integer li_rows, li_i

li_rows = dw_customer.RowCount()
FOR li_i = 1 TO li_rows
    lstr_cust[li_i].cust_id = dw_customer.GetItemString(li_i, "cust_id")
    lstr_cust[li_i].cust_name = dw_customer.GetItemString(li_i, "cust_name")
    lstr_cust[li_i].phone = dw_customer.GetItemString(li_i, "phone")
NEXT
```

#### 구조체 배열을 DataWindow에 설정하기

PowerBuilder에서 직접 구조체 배열을 DataWindow에 설정하는 함수는 없지만, 반복문을 사용하여 추가할 수 있습니다.

```
FOR li_i = 1 TO UpperBound(lstr_cust)
    dw_customer.InsertRow(0)
    dw_customer.SetItem(li_i, "cust_id", lstr_cust[li_i].cust_id)
    dw_customer.SetItem(li_i, "cust_name", lstr_cust[li_i].cust_name)
NEXT
```

### 구조체의 활용 예: 고객 관리

```
// 고객 정보를 구조체로 정의
Global Type s_customer
    String id
    String name
    String phone
    String email
    Date   reg_date
End Type

// 윈도우 함수: 고객 목록을 구조체 배열로 반환
FUNCTION s_customer[] uf_get_customer_list ()
    s_customer lstr_cust[]
    Integer li_i = 1
    
    // 데이터베이스 조회 가정 (실제로는 DataWindow 사용)
    lstr_cust[li_i].id = "C001"; lstr_cust[li_i].name = "홍길동"; lstr_cust[li_i].phone = "010-1111-2222"; li_i++
    lstr_cust[li_i].id = "C002"; lstr_cust[li_i].name = "김철수"; lstr_cust[li_i].phone = "010-3333-4444"; li_i++
    lstr_cust[li_i].id = "C003"; lstr_cust[li_i].name = "이영희"; lstr_cust[li_i].phone = "010-5555-6666"; 
    
    RETURN lstr_cust
END FUNCTION

// 윈도우 오픈 이벤트에서 호출하여 리스트박스에 표시
s_customer lstr_list[]
Integer li_i

lstr_list = uf_get_customer_list()

FOR li_i = 1 TO UpperBound(lstr_list)
    lb_customer.AddItem(lstr_list[li_i].id + " - " + lstr_list[li_i].name)
NEXT
```

---

## 배열과 구조체의 고급 활용

### 가변 길이 배열의 동적 확장 (복습)

앞서 설명한 대로, 동적 배열에 대량의 데이터를 추가할 때는 마지막 예상 인덱스에 더미 값을 먼저 할당하여 성능을 최적화할 수 있습니다.

### 구조체의 기본값 설정

구조체 변수를 선언하면 각 필드는 데이터 타입의 기본값으로 초기화됩니다. 사용자 정의 초기값을 원하면 별도의 함수를 만들어 사용할 수 있습니다.

```
// 새 구조체를 기본값으로 초기화하는 함수
FUNCTION s_customer f_new_customer ()
    s_customer lstr_cust
    lstr_cust.id = ""
    lstr_cust.name = "이름 없음"
    lstr_cust.phone = ""
    lstr_cust.reg_date = Today()
    RETURN lstr_cust
END FUNCTION
```

### 구조체와 사용자 객체 (NVO) 비교

구조체는 데이터를 묶는 용도로 매우 유용하지만, 다음과 같은 한계가 있습니다.

- **함수나 이벤트를 가질 수 없습니다.** 즉, 데이터와 관련된 동작(메서드)을 포함할 수 없습니다.
- **상속(Inheritance)이 불가능합니다.** 구조체 간에 계층 관계를 만들 수 없습니다.

따라서 프로젝트 규모가 커져서 데이터를 다루는 공통 로직을 상속받아 사용해야 한다면, 구조체 대신 인스턴스 변수만 가진 **비시각적 객체(Non-Visual Object, NVO)**를 데이터 컨테이너로 사용하는 것이 훨씬 유리합니다. NVO는 메서드와 속성을 캡슐화할 수 있고, 상속을 통해 기능을 확장할 수 있습니다.

```
// NVO로 데이터 컨테이너 정의 (n_customer)
// 인스턴스 변수: id, name, phone
// 메서드: of_save(), of_validate() 등
```

---

## 디버깅 팁

### 배열 인덱스 오류 주의

PowerBuilder에서 배열 인덱스는 1부터 시작합니다. 0부터 시작하는 언어에 익숙한 경우 실수하기 쉽습니다.

### 배열 범위 초과 접근

정적 배열의 범위를 벗어난 인덱스에 접근하면 런타임 오류가 발생합니다. 항상 `UpperBound()`로 크기를 확인하는 습관을 들이세요.

### 구조체 필드 오타

구조체 필드 이름을 잘못 입력하면 컴파일 오류가 발생합니다. 대소문자까지 정확히 일치해야 합니다.

---

## 마치며

배열과 구조체는 PowerBuilder에서 데이터를 체계적으로 관리하는 기본 도구입니다. 배열은 동일한 성격의 데이터 집합을 처리할 때, 구조체는 서로 다른 속성을 가진 데이터를 하나의 논리적 단위로 묶을 때 유용합니다. 이 두 가지를 적절히 조합하면 복잡한 데이터도 효율적으로 다룰 수 있습니다.

특히 구조체는 함수 간에 여러 값을 주고받아야 하거나, DataWindow의 데이터를 임시로 저장해야 할 때 매우 유용합니다. 다만 구조체는 데이터만 담을 수 있고 상속이 불가능하므로, 객체 지향적 설계가 필요하다면 비시각적 객체(NVO) 사용을 고려해야 합니다.

배열과 구조체를 능숙하게 활용하면 코드의 가독성과 유지보수성을 크게 향상시킬 수 있습니다.