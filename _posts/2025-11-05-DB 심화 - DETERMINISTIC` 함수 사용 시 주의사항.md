---
layout: post
title: DB 심화 - DETERMINISTIC` 함수 사용 시 주의사항
date: 2025-11-05 22:25:23 +0900
category: DB 심화
---
# Oracle DETERMINISTIC 함수의 이해와 올바른 사용법

> **핵심 요약**  
> `DETERMINISTIC`은 함수가 "동일한 입력에 대해 항상 동일한 출력을 반환한다"는 것을 개발자가 보증하는 선언입니다. 이는 자동 캐싱을 제공하지 않지만, 함수 기반 인덱스(FBI)와 가상 컬럼의 필수 조건입니다. 잘못 사용하면 심각한 데이터 무결성 문제를 초래할 수 있으므로 주의가 필요합니다.

---

## DETERMINISTIC의 본질적 의미

```sql
CREATE OR REPLACE FUNCTION fx_normalize_text(p_txt IN VARCHAR2)
RETURN VARCHAR2
DETERMINISTIC
IS
BEGIN
  -- 단순한 문자열 변환: 항상 동일한 입력에 대해 동일한 출력
  RETURN TRIM(LOWER(p_txt));
END;
/
```

**중요한 이해 사항:**
- `DETERMINISTIC`은 **"동일 입력 → 항상 동일 출력"** 을 개발자가 보증하는 선언입니다.
- **자동 캐싱 기능을 제공하지 않습니다.** 캐싱이 필요하다면 `RESULT_CACHE`를 사용해야 합니다.
- 함수 기반 인덱스(FBI), 가상 컬럼, 쿼리 재작성의 필수 전제 조건입니다.
- 거짓으로 선언하면 인덱스와 실제 데이터 간 불일치가 발생할 수 있습니다.

---

## DETERMINISTIC을 사용해서는 안 되는 상황

### 1. 시간에 의존하는 함수
```sql
-- ❌ 절대 사용 금지: 출력이 시간에 따라 변함
CREATE OR REPLACE FUNCTION get_today()
RETURN DATE
DETERMINISTIC
IS
BEGIN
  RETURN TRUNC(SYSDATE);  -- 매일 다른 값 반환
END;
/
```

### 2. 난수 또는 시퀀스 생성
```sql
-- ❌ 절대 사용 금지: 매번 다른 값 반환
CREATE OR REPLACE FUNCTION generate_random_number()
RETURN NUMBER
DETERMINISTIC
IS
BEGIN
  RETURN DBMS_RANDOM.VALUE();  -- 항상 다른 값
END;
/
```

### 3. 세션 또는 NLS 설정에 의존하는 함수
```sql
-- ❌ 주의 필요: NLS 설정에 따라 결과가 달라질 수 있음
CREATE OR REPLACE FUNCTION format_text(p_txt VARCHAR2)
RETURN VARCHAR2
DETERMINISTIC
IS
BEGIN
  -- INITCAP의 결과는 NLS 설정에 의존적임
  RETURN INITCAP(p_txt);
END;
/
```

### 4. 외부 테이블 데이터에 의존하는 함수
```sql
-- ❌ 절대 사용 금지: 외부 데이터 변경에 따라 출력이 변함
CREATE OR REPLACE FUNCTION get_exchange_rate(p_currency VARCHAR2)
RETURN NUMBER
DETERMINISTIC
IS
  v_rate NUMBER;
BEGIN
  SELECT rate INTO v_rate FROM exchange_rates WHERE currency = p_currency;
  RETURN v_rate;  -- 환율 테이블 데이터가 변경되면 출력도 변경됨
END;
/
```

### 5. 패키지 상태에 의존하는 함수
```sql
-- ❌ 절대 사용 금지: 패키지 변수 상태에 따라 결과가 달라짐
CREATE OR REPLACE PACKAGE pkg_stateful AS
  g_counter NUMBER := 0;
  
  FUNCTION get_next_value RETURN NUMBER DETERMINISTIC;
END pkg_stateful;
/

CREATE OR REPLACE PACKAGE BODY pkg_stateful AS
  FUNCTION get_next_value RETURN NUMBER DETERMINISTIC IS
  BEGIN
    g_counter := g_counter + 1;
    RETURN g_counter;  -- 호출할 때마다 다른 값 반환
  END;
END pkg_stateful;
/
```

---

## DETERMINISTIC vs RESULT_CACHE 비교

| 특성 | DETERMINISTIC | RESULT_CACHE |
|------|---------------|--------------|
| **의미** | "동일 입력 → 동일 출력" 보증 | SGA에 결과 캐싱 |
| **캐싱** | 제공하지 않음 | 자동 캐싱 제공 |
| **데이터 일관성** | 개발자 책임 | 자동 무효화 지원 |
| **함수 기반 인덱스** | 필수 조건 | 관련 없음 |
| **위험성** | 잘못 선언 시 데이터 무결성 문제 | 캐시 일관성 관리 필요 |
| **적합한 상황** | 순수 수학 함수 | 비용이 큰 계산, 입력 다양성 낮음 |

---

## 함수 기반 인덱스(FBI)와 가상 컬럼에서의 활용

### 안전한 사용 예시
```sql
-- 정규화 함수: 순수한 문자열 변환
CREATE OR REPLACE FUNCTION normalize_phone(p_phone VARCHAR2)
RETURN VARCHAR2
DETERMINISTIC
IS
BEGIN
  -- 숫자만 추출: 항상 동일한 입력에 대해 동일한 출력
  RETURN REGEXP_REPLACE(p_phone, '[^0-9]', '');
END;
/

-- 가상 컬럼과 함수 기반 인덱스 생성
CREATE TABLE customers (
  customer_id NUMBER PRIMARY KEY,
  phone_raw   VARCHAR2(20),
  phone_norm  VARCHAR2(20) GENERATED ALWAYS AS (normalize_phone(phone_raw)) VIRTUAL
);

CREATE INDEX idx_customers_phone_norm ON customers(phone_norm);

-- 효율적인 검색 가능
SELECT * FROM customers WHERE phone_norm = normalize_phone('(123) 456-7890');
```

### 위험한 사용 예시
```sql
-- ❌ 위험: 외부 테이블에 의존하는 함수
CREATE OR REPLACE FUNCTION calculate_discount(p_customer_id NUMBER)
RETURN NUMBER
DETERMINISTIC
IS
  v_discount_rate NUMBER;
BEGIN
  -- 고객 등급 테이블에서 할인율 조회
  SELECT discount_rate INTO v_discount_rate 
  FROM customer_grades 
  WHERE customer_id = p_customer_id;
  
  RETURN v_discount_rate;  -- 테이블 데이터 변경 시 출력도 변경됨
END;
/

-- 이 함수로 생성된 인덱스는 데이터 변경 시 무효화될 수 있음
```

---

## 결정성 위반 요소 체크리스트

다음 요소 중 하나라도 함수에 포함되어 있다면 `DETERMINISTIC`을 사용해서는 안 됩니다:

### 시간 관련
- `SYSDATE`, `SYSTIMESTAMP`, `CURRENT_DATE`
- `CURRENT_TIMESTAMP`

### 난수 및 시퀀스
- `DBMS_RANDOM` 패키지 함수
- `SYS_GUID()`
- 시퀀스의 `NEXTVAL`

### 세션 상태
- NLS 관련 설정 (`NLS_SORT`, `NLS_COMP` 등)
- 시간대 설정 (`TIME_ZONE`)
- `SYS_CONTEXT` 값
- 역할(Role) 및 권한
- 클라이언트 식별자

### 외부 의존성
- 데이터베이스 테이블/뷰 조회
- 패키지 전역 변수
- 데이터베이스 링크
- 외부 파일 또는 네트워크 자원

---

## 성능 최적화 대안

### 1. 조인을 통한 집합적 처리
```sql
-- ❌ 비효율적: 행마다 함수 호출
SELECT * FROM orders WHERE calculate_tax(amount, region) > 100;

-- ✅ 효율적: 조인 활용
SELECT o.*
FROM orders o
JOIN tax_rates t ON t.region = o.region
WHERE o.amount * t.rate > 100;
```

### 2. 스칼라 서브쿼리 캐싱
```sql
-- Oracle이 동일 입력 값에 대해 결과를 자동 캐싱
SELECT o.order_id,
       (SELECT status_name FROM order_statuses WHERE status_id = o.status_id) AS status
FROM orders o
WHERE o.customer_id = :customer_id;
```

### 3. RESULT_CACHE 활용
```sql
CREATE OR REPLACE FUNCTION get_product_category(p_product_id NUMBER)
RETURN VARCHAR2
RESULT_CACHE
IS
  v_category VARCHAR2(100);
BEGIN
  SELECT category_name INTO v_category
  FROM product_categories
  WHERE category_id = (
    SELECT category_id FROM products WHERE product_id = p_product_id
  );
  
  RETURN v_category;
END;
/
```

### 4. 머티리얼라이즈드 뷰 활용
```sql
-- 복잡한 계산 결과를 사전에 구해 저장
CREATE MATERIALIZED VIEW mv_order_summary
REFRESH ON COMMIT
AS
SELECT o.order_id,
       o.amount,
       t.rate AS tax_rate,
       o.amount * t.rate AS tax_amount
FROM orders o
JOIN tax_rates t ON t.region = o.region;
```

---

## 결정성 검증 방법

### 반복 호출 일관성 테스트
```sql
-- 동일 입력에 대한 반복 호출 결과 검증
DECLARE
  v_input VARCHAR2(100) := 'Test Input';
  v_first_result VARCHAR2(100);
  v_current_result VARCHAR2(100);
  v_is_consistent BOOLEAN := TRUE;
BEGIN
  -- 첫 번째 호출 결과 저장
  v_first_result := your_function(v_input);
  
  -- 1000번 반복 호출하여 일관성 검증
  FOR i IN 1..1000 LOOP
    v_current_result := your_function(v_input);
    
    IF v_current_result != v_first_result THEN
      v_is_consistent := FALSE;
      EXIT;
    END IF;
  END LOOP;
  
  IF v_is_consistent THEN
    DBMS_OUTPUT.PUT_LINE('함수는 결정적으로 보입니다.');
  ELSE
    DBMS_OUTPUT.PUT_LINE('함수는 비결정적입니다!');
  END IF;
END;
/
```

### 세션 설정 변경 테스트
```sql
-- NLS 설정 변경 시 결과 일관성 검증
ALTER SESSION SET NLS_SORT = 'BINARY';
SELECT your_function('test') FROM dual;

ALTER SESSION SET NLS_SORT = 'GERMAN';
SELECT your_function('test') FROM dual;

ALTER SESSION SET NLS_COMP = 'LINGUISTIC';
SELECT your_function('test') FROM dual;
```

### 시간 경과 테스트
```sql
-- 시간이 지남에 따라 결과 변경 여부 검증
SELECT your_function('input') FROM dual;

-- 10초 대기
BEGIN
  DBMS_LOCK.SLEEP(10);
END;
/

SELECT your_function('input') FROM dual;
```

---

## 안전한 사용 패턴 예시

### 1. 데이터 정규화 함수
```sql
CREATE OR REPLACE FUNCTION normalize_email(p_email VARCHAR2)
RETURN VARCHAR2
DETERMINISTIC
IS
BEGIN
  -- 이메일 주소 표준화: 항상 동일한 변환
  RETURN LOWER(TRIM(REGEXP_REPLACE(p_email, '\s+', '')));
END;
/

-- 가상 컬럼 및 인덱스 생성
ALTER TABLE users ADD email_normalized GENERATED ALWAYS AS (normalize_email(email)) VIRTUAL;
CREATE INDEX idx_users_email_norm ON users(email_normalized);
```

### 2. 해시 생성 함수
```sql
CREATE OR REPLACE FUNCTION generate_data_hash(p_data VARCHAR2)
RETURN VARCHAR2
DETERMINISTIC
IS
BEGIN
  -- 결정적 해시 생성
  RETURN DBMS_CRYPTO.HASH(
    UTL_I18N.STRING_TO_RAW(p_data, 'AL32UTF8'),
    DBMS_CRYPTO.HASH_SH256
  );
END;
/
```

### 3. 비즈니스 규칙 적용 (순수 계산)
```sql
CREATE OR REPLACE FUNCTION calculate_age_category(p_birth_date DATE)
RETURN VARCHAR2
DETERMINISTIC
IS
  v_age_years NUMBER;
BEGIN
  -- 순수 계산: 나이 기반 카테고리 분류
  v_age_years := FLOOR(MONTHS_BETWEEN(SYSDATE, p_birth_date) / 12);
  
  RETURN CASE
    WHEN v_age_years < 18 THEN 'MINOR'
    WHEN v_age_years BETWEEN 18 AND 64 THEN 'ADULT'
    ELSE 'SENIOR'
  END;
END;
/
```

---

## 실무 적용 지침

### 배포 전 확인사항
1. **순수성 검증**: 함수가 오직 입력 매개변수만 사용하는지 확인하세요.
2. **외부 의존성 제거**: 데이터베이스 테이블, 시간, 세션 상태에 의존하지 않는지 확인하세요.
3. **NLS 독립성**: 문자열 함수가 NLS 설정에 독립적인지 확인하세요.
4. **테스트 수행**: 다양한 환경에서 일관된 결과를 반환하는지 테스트하세요.
5. **문서화**: 함수의 결정적 특성과 사용 제한사항을 명확히 문서화하세요.

### 성능 고려사항
- `DETERMINISTIC` 함수는 행 단위로 호출될 때 CPU 부하가 클 수 있습니다.
- 대량 데이터 처리 시 조인이나 집합적 접근 방식을 고려하세요.
- 함수 기반 인덱스는 생성 및 유지 관리 비용이 있습니다.

### 유지보수 지침
- `DETERMINISTIC` 함수의 구현 변경 시 관련된 모든 인덱스를 재구성해야 합니다.
- 함수의 결정적 특성이 변경되면 `DETERMINISTIC` 선언을 제거하고 관련 객체를 재평가해야 합니다.
- 정기적으로 함수의 결정성을 검증하는 테스트를 수행하세요.

---

## 결론

`DETERMINISTIC` 함수 선언은 Oracle 데이터베이스에서 중요한 최적화 도구이지만, 잘못 사용하면 심각한 데이터 무결성 문제를 초래할 수 있습니다. 올바른 사용을 위한 핵심 원칙은 다음과 같습니다:

### 핵심 원칙
1. **진정한 순수 함수에만 적용**: 함수가 오직 입력 매개변수에만 의존하고, 외부 상태(시간, 세션, 데이터베이스 테이블 등)에 전혀 의존하지 않을 때만 `DETERMINISTIC`을 선언하세요.

2. **함수 기반 인덱스의 필수 조건**: FBI나 가상 컬럼을 생성하려면 반드시 `DETERMINISTIC` 함수가 필요하지만, 이는 함수가 진정으로 결정적일 때만 안전합니다.

3. **캐싱과 구분**: `DETERMINISTIC`은 자동 캐싱을 제공하지 않습니다. 캐싱이 필요하다면 `RESULT_CACHE`를 사용하세요.

4. **테스트 기반 검증**: 함수의 결정성은 다양한 환경(다른 세션 설정, 다른 시간대, 다른 데이터베이스 상태)에서 테스트하여 검증해야 합니다.

### 실전 조언
- 의심스러운 경우 `DETERMINISTIC`을 사용하지 마세요. 오류 가능성이 있는 최적화보다 안전한 비최적화가 더 좋습니다.
- 복잡한 비즈니스 로직은 가능한 한 조인, 뷰, 머티리얼라이즈드 뷰 등 집합적 접근 방식으로 구현하세요.
- 성능 문제가 발생할 때는 함수 호출 최소화, 인덱스 설계, 쿼리 재작성 등 다양한 접근 방식을 고려하세요.

### 최종 점검
함수에 `DETERMINISTIC`을 추가하기 전에 스스로에게 물어보세요: "이 함수가 100년 후, 다른 데이터베이스 서버에서, 다른 사용자 세션에서 실행되어도 동일한 입력에 대해 정확히 동일한 결과를 반환할 것인가?" 만약 이 질문에 확신 있게 "예"라고 답할 수 없다면, `DETERMINISTIC`을 사용하지 않는 것이 안전합니다.

기억하세요: 데이터베이스의 최우선 가치는 정확성입니다. 성능 최적화는 정확성을 훼손하지 않는 범위 내에서 이루어져야 합니다. `DETERMINISTIC`의 올바른 사용은 이 원칙을 준수하면서도 시스템 성능을 향상시키는 강력한 도구가 될 수 있습니다.