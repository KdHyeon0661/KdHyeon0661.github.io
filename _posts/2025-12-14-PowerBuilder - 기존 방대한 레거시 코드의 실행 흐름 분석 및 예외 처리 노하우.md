---
layout: post
title: PowerBuilder - 기존 방대한 레거시 코드의 실행 흐름 분석 및 예외 처리 노하우
date: 2025-12-14 17:30:23 +0900
category: PowerBuilder
---
# 기존 방대한 레거시 코드의 실행 흐름 분석 및 예외 처리 노하우

PowerBuilder로 개발된 시스템을 유지보수하다 보면 수백, 수천 개의 객체와 방대한 양의 레거시 코드를 마주하게 됩니다. 특히 오래된 프로젝트는 문서화가 부족하고, 복잡한 이벤트 흐름과 예외 처리 방식이 혼재되어 있어 문제 발생 시 원인 파악에 많은 시간이 소요됩니다. 이 글에서는 기존 레거시 코드의 실행 흐름을 효과적으로 추적하고 분석하는 노하우와, Null 처리 및 예외 상황에 대응하는 방법을 상세히 소개합니다. 이를 통해 에러 발생 시 빠르고 정확하게 문제를 해결하는 '해결사'로 거듭날 수 있을 것입니다.

---

## 1. 방대한 레거시 코드의 실행 흐름 분석 노하우

### 1.1 이벤트 순서 이해의 중요성

PowerBuilder 애플리케이션은 이벤트 기반(Event-Driven)으로 동작합니다. 윈도우 오픈, 버튼 클릭, 데이터윈도우 아이템 변경 등 수많은 이벤트가 연쇄적으로 발생하며, 각 이벤트에 연결된 스크립트가 실행됩니다. 문제가 발생했을 때 어떤 이벤트가 어떤 순서로 실행되는지 파악하는 것이 핵심입니다.

### 1.2 디버거를 이용한 실시간 추적

가장 강력한 도구는 PowerBuilder IDE의 디버거입니다.

- **중단점(Breakpoint) 전략적 배치**: 의심되는 이벤트의 첫 줄과 주요 함수 호출 지점에 중단점을 설정합니다.
- **Call Stack 분석**: 중단점에서 멈춘 후 Call Stack 창을 열면 현재까지의 이벤트/함수 호출 순서를 역추적할 수 있습니다.
- **Watch 창 활용**: 주요 변수의 값 변화를 실시간으로 관찰합니다.
- **Step Into/Over**: 이벤트 간 이동하며 실행 흐름을 한 단계씩 따라갑니다.

### 1.3 메시지 박스로 흐름 추적 (개발 중)

디버거를 사용할 수 없는 환경(예: 운영 이슈 분석)에서는 임시로 메시지 박스를 삽입하여 이벤트 실행 순서를 확인할 수 있습니다.

```powerbuilder
// 이벤트 시작 부분에 삽입
MessageBox("Trace", "w_customer.Open 이벤트 시작")
```

단, 운영 코드에 이런 메시지 박스를 남기면 사용자 경험을 해치므로, 조건부 컴파일이나 별도 로깅 함수를 사용하는 것이 좋습니다.

### 1.4 로그 파일에 흐름 기록

앞서 배운 로깅 기법을 활용하여 주요 이벤트의 실행을 파일에 기록하면, 운영 중에도 실행 순서를 추적할 수 있습니다.

```powerbuilder
// 로그 함수 호출
f_log_write("w_customer.Open 이벤트 시작 - 사용자: " + gs_user_id)
```

### 1.5 PFC 애플리케이션에서의 추적

PFC 프레임워크를 사용하는 프로젝트라면 **메시지 라우터(Message Router)**의 흐름을 이해해야 합니다. PFC에서는 메뉴 클릭 등이 직접 윈도우 이벤트를 호출하지 않고 메시지 라우터를 통해 전달됩니다. `pfc_Open`, `pfc_Retrieve` 등 PFC 표준 이벤트에 중단점을 설정하고, `inv_router` 관련 코드를 추적하면 전체 흐름을 파악할 수 있습니다.

### 1.6 코드 검색 도구 활용

- **전체 프로젝트 검색**: 특정 함수나 변수가 어디서 호출되는지 찾으려면 PowerBuilder IDE의 **Browse** 기능이나 **Search in Files**를 사용합니다.
- **참조(References) 확인**: 객체를 우클릭하고 **Find References**를 선택하면 해당 객체를 사용하는 모든 위치를 찾을 수 있습니다.

### 1.7 실행 흐름 다이어그램 그리기

복잡한 프로세스는 직접 손으로 다이어그램을 그려보는 것도 큰 도움이 됩니다. 윈도우 오픈 → 데이터 조회 → 사용자 입력 → 저장 순서를 따라가며 각 단계에서 발생하는 이벤트와 함수 호출을 정리합니다.

### 1.8 레거시 코드 분석 팁

- **변수 명명 규칙 파악**: 접두사(예: `g_` 전역, `i_` 인스턴스, `l_` 지역)를 통해 변수의 범위를 추정합니다.
- **상속 관계 확인**: 객체 브라우저(Object Browser)에서 윈도우나 사용자 객체의 상속 계층을 확인합니다. 부모 객체에 정의된 이벤트나 함수가 자식에서 오버라이드되었는지 파악해야 합니다.
- **주석 활용**: 오래된 코드에는 개발자의 주석이 남아 있을 수 있습니다. 주석을 꼼꼼히 읽으면 의도를 이해하는 데 도움이 됩니다.

---

## 2. Null 처리 및 예외 상황 대응

### 2.1 PowerBuilder에서 Null의 의미

PowerBuilder에서 Null은 '값이 없음' 또는 '알 수 없음'을 나타내는 특수한 상태입니다. 데이터베이스의 NULL과 동일한 개념입니다. Null은 어떤 변수에든 할당될 수 있으며, Null이 포함된 연산 결과는 대부분 Null이 됩니다.

```powerbuilder
Integer li_a, li_b
SetNull(li_a)   // li_a를 Null로 설정
li_b = li_a + 5 // li_b도 Null이 됨
```

### 2.2 IsNull() 함수로 Null 체크

Null 상태를 확인하려면 반드시 `IsNull()` 함수를 사용해야 합니다. `IF li_a = Null`과 같은 비교는 올바르지 않습니다.

```powerbuilder
IF IsNull(li_a) THEN
   MessageBox("경고", "값이 없습니다.")
END IF
```

### 2.3 Null 관련 자주 발생하는 오류

#### 2.3.1 문자열 연결 시 Null
```powerbuilder
String ls_name, ls_greeting
ls_name = GetItemString(...) // 만약 Null이면?
ls_greeting = "안녕하세요, " + ls_name + "님" // ls_name이 Null이면 전체 결과 Null
```

**해결**: 문자열 연결 전 IsNull 체크 또는 `Null` 대신 빈 문자열로 변환.

```powerbuilder
IF IsNull(ls_name) THEN ls_name = ""
```

#### 2.3.2 DataWindow 컬럼 값이 Null인 경우
DataWindow에서 `GetItem` 계열 함수를 호출할 때 해당 컬럼 값이 Null이면 런타임 오류가 발생하지 않고 변수에 Null이 할당됩니다.

```powerbuilder
String ls_phone = dw_1.GetItemString(1, "phone") // phone이 Null이면 ls_phone도 Null
```

이후 ls_phone을 사용할 때 Null 체크를 하지 않으면 예상치 못한 결과가 발생할 수 있습니다.

**해결**: DataWindow 값 사용 전 반드시 `IsNull()` 체크.

```powerbuilder
IF IsNull(dw_1.GetItemString(1, "phone")) THEN
   // Null 처리
END IF
```

### 2.4 Null을 고려한 함수 작성

함수에서 반환값이 Null일 수 있다면, 호출자에게 Null 가능성을 알리는 것이 좋습니다.

```powerbuilder
FUNCTION String f_get_customer_phone (String as_id)
   String ls_phone
   SELECT phone INTO :ls_phone FROM customer WHERE id = :as_id;
   IF SQLCA.SQLCode = 100 THEN
      SetNull(ls_phone)   // 데이터 없음 → Null 반환
   END IF
   RETURN ls_phone
END FUNCTION
```

호출부에서는 다음과 같이 사용합니다.

```powerbuilder
String ls_phone = f_get_customer_phone("C001")
IF IsNull(ls_phone) THEN
   // 데이터 없음 처리
END IF
```

### 2.5 PowerBuilder의 예외 처리 메커니즘

PowerBuilder는 전통적으로 임베디드 SQL의 상태 코드(SQLCode)나 DataWindow의 DBError 이벤트를 통해 오류를 처리해 왔습니다. 또한 **PowerBuilder 8.0부터 도입된 Try-Catch-Finally** 구조를 활용하면 더욱 구조적인 예외 처리가 가능합니다.

#### 2.5.1 임베디드 SQL 오류 처리

```powerbuilder
DELETE FROM employee WHERE emp_id = :ll_id USING SQLCA;
IF SQLCA.SQLCode = -1 THEN
   ROLLBACK USING SQLCA;
   MessageBox("오류", SQLCA.SQLErrText)
ELSE
   COMMIT USING SQLCA;
END IF
```

#### 2.5.2 DataWindow DBError 이벤트 활용

DataWindow에서 데이터베이스 관련 오류가 발생하면 DBError 이벤트가 발생합니다. 여기서 사용자 정의 메시지를 표시하거나 오류를 로깅할 수 있습니다.

```powerbuilder
// dw_1의 DBError 이벤트
IF sqldbcode = 2627 THEN
   MessageBox("중복 오류", "이미 존재하는 키입니다.")
   RETURN 1   // 기본 오류 메시지 박스 방지
END IF
RETURN 0
```

#### 2.5.3 SystemError 이벤트

처리되지 않은 예외가 발생하면 애플리케이션 객체의 **SystemError** 이벤트가 실행됩니다. 여기서 로그를 남기고 우아하게 종료할 수 있습니다.

```powerbuilder
// Application 객체의 SystemError 이벤트
f_log_write("치명적 오류: " + Error.Text)
HALT CLOSE
```

#### 2.5.4 Try-Catch 예외 처리 (PowerBuilder 8.0 이상)

PowerBuilder 8.0부터 도입된 Try-Catch 구문을 사용하면 구조적인 예외 처리가 가능합니다. 시스템 에러(0으로 나누기, 널 참조 등)는 `RuntimeError`로 잡고, 개발자가 `Throw`로 던진 예외는 `Exception`으로 잡을 수 있습니다.

```powerbuilder
TRY
   // 예외가 발생할 수 있는 코드
   Integer li_result = 10 / 0   // 0으로 나누기 런타임 에러 발생
CATCH (RuntimeError re)
   // 파워빌더의 치명적인 시스템 에러는 RuntimeError로 잡습니다.
   MessageBox("시스템 오류", "런타임 에러 발생: " + re.Text)
CATCH (Exception ex)
   // 개발자가 명시적으로 던진(Throw) 예외를 잡습니다.
   MessageBox("사용자 예외", "일반 예외: " + ex.GetMessage())
FINALLY
   // 에러 발생 여부와 상관없이 항상 실행되는 정리 코드
END TRY
```

**참고**: PowerBuilder 8.0 이후의 모든 버전에서 이 구문을 사용할 수 있습니다. 만약 PowerBuilder 5/6/7과 같은 초창기 버전을 유지보수하는 경우에는 전통적인 오류 처리 방식을 사용해야 합니다.

### 2.6 예외 상황 대응 실전 팁

#### 2.6.1 NullReference 오류 방지
객체가 생성되었는지 확인하지 않고 메서드를 호출하면 NullReference 오류가 발생할 수 있습니다.

```powerbuilder
IF IsValid(lnv_object) THEN
   lnv_object.of_something()
END IF
```

#### 2.6.2 데이터베이스 연결 상태 확인
`CONNECT` 후 `SQLCA.SQLCode`를 반드시 확인하고, 실패 시 적절히 처리합니다.

#### 2.6.3 외부 리소스 사용 후 정리
파일, OLE 객체, 트랜잭션 객체 등은 사용 후 반드시 닫거나 소멸시킵니다.

```powerbuilder
// 파일 처리 후
FileClose(li_handle)

// OLE 객체
ole_object.DisconnectObject()
DESTROY ole_object

// 사용자 정의 객체
DESTROY lnv_custom
```

#### 2.6.4 예외 로깅
예외 발생 시 사용자에게 보여주는 메시지와 함께 로그 파일에도 상세 정보를 기록하여 추후 분석에 활용합니다.

```powerbuilder
TRY
   // 위험한 코드
CATCH (RuntimeError re)
   String ls_log
   ls_log = "런타임 오류 발생: " + re.Text + " at " + String(Now())
   f_log_write(ls_log)
   MessageBox("오류", "시스템 오류가 발생했습니다.")
CATCH (Exception ex)
   String ls_log
   ls_log = "예외 발생: " + ex.GetMessage() + " at " + String(Now())
   f_log_write(ls_log)
   MessageBox("오류", ex.GetMessage())
END TRY
```

---

## 3. 종합 예제: 레거시 코드 분석과 예외 처리 강화

다음은 실제 유지보수 상황에서 발생할 수 있는 시나리오입니다.

### 상황
고객 등록 화면에서 가끔 "DataWindow 업데이트 실패" 오류가 발생합니다. 오류 메시지 외에 아무 정보도 남지 않아 원인 파악이 어렵습니다.

### 분석 과정

1. **이벤트 흐름 추적**:
   - 저장 버튼의 Clicked 이벤트에 중단점을 설정하고 디버그 모드 실행.
   - Clicked 이벤트에서 호출되는 함수(`uf_save`)를 Step Into로 따라감.
   - `dw_customer.Update()` 호출 전후로 주요 변수(DataWindow 행 상태 등)를 Watch에 등록.

2. **Null 값 확인**:
   - DataWindow의 각 컬럼 값을 GetItemString으로 가져올 때 Null이 있는지 확인.
   - 필수 입력 컬럼에 Null이 있으면 Update 실패 원인이 될 수 있음.

3. **DBError 이벤트 확인**:
   - DataWindow의 DBError 이벤트에 중단점을 설정하여 어떤 오류 코드가 발생하는지 확인.
   - 예: 8152(문자열 잘림), 547(참조 무결성) 등.

4. **로깅 추가**:
   - 저장 시도 전후로 로그 파일에 입력 값과 결과를 기록하도록 임시 코드 추가.

### 해결

DBError 이벤트에서 확인한 결과, 특정 컬럼의 길이가 데이터베이스 컬럼 길이를 초과하여 문자열 잘림 오류(8152)가 발생함을 발견. 해당 컬럼의 DataWindow 편집 스타일을 수정하여 입력 길이를 제한하고, 코드에서도 길이 체크 로직을 추가하여 문제 해결.

### 예외 처리 강화

향후 유사 문제를 방지하기 위해 저장 함수에 Try-Catch를 적용하고, DBError 이벤트에 상세 로깅을 추가합니다.

```powerbuilder
// dw_customer의 DBError 이벤트
String ls_log = "DBError 발생: 코드=" + String(sqldbcode) + ", 메시지=" + sqlerrtext
f_log_write(ls_log)
// 사용자에게는 간결한 메시지
CHOOSE CASE sqldbcode
   CASE 8152
      MessageBox("입력 오류", "입력 값이 너무 깁니다.")
   CASE ELSE
      MessageBox("데이터베이스 오류", sqlerrtext)
END CHOOSE
RETURN 1
```

---

## 4. 결론: '해결사'가 되기 위한 핵심 마인드

레거시 코드를 분석하고 예외를 처리하는 능력은 단순히 문법을 아는 것을 넘어, **시스템의 전체적인 흐름을 이해하고 문제의 근본 원인을 찾는 통찰력**이 필요합니다.

- **체계적인 접근**: 디버거와 로깅을 적극 활용하고, 이벤트 순서를 기록하며, 변수의 상태를 추적하세요.
- **Null과 예외에 민감해지기**: Null이 전파되는 경로를 항상 의식하고, 모든 예외 상황을 단순한 오류 처리로 끝내지 말고 로깅과 함께 분석하세요.
- **코드 리딩 능력 향상**: 상속 관계, PFC 구조, 메시지 라우터 등 PowerBuilder 특유의 패턴을 익히면 레거시 코드를 읽는 속도가 빨라집니다.
- **문서화 습관**: 문제를 해결한 후에는 간단한 분석 노트나 주석을 남겨 두어, 같은 문제가 재발했을 때 빠르게 대응할 수 있도록 합니다.
