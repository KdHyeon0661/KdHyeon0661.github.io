---
layout: post
title: DB 심화 - Cursor Sharing
date: 2025-11-05 21:25:23 +0900
category: DB 심화
---
# Oracle Cursor Sharing 완전 가이드

> **핵심 이해**
> Oracle Cursor Sharing은 동일하거나 유사한 SQL 문장이 같은 실행 계획을 재사용하도록 하는 메커니즘입니다. 이는 파싱 오버헤드를 줄이고, 메모리 사용을 최적화하며, 시스템 성능을 향상시킵니다. 효과적인 커서 공유는 데이터베이스 성능 최적화의 핵심 요소입니다.

---

## 실습 환경 준비

```sql
-- 테스트 테이블 생성 및 샘플 데이터 적재
DROP TABLE cs_demo PURGE;

-- 편향된 데이터 분포를 가진 테이블 생성
-- status 컬럼: 'HOT' 1%, 'COLD' 99%
CREATE TABLE cs_demo (
  id      NUMBER PRIMARY KEY,
  status  VARCHAR2(10),
  payload VARCHAR2(200)
);

-- 샘플 데이터 삽입 (10만 건)
BEGIN
  FOR i IN 1..100000 LOOP
    INSERT INTO cs_demo
    VALUES (
      i,
      CASE WHEN DBMS_RANDOM.VALUE(0,100) < 1 THEN 'HOT' ELSE 'COLD' END,
      RPAD('x',100,'x')
    );
  END LOOP;
  COMMIT;
END;
/

-- 인덱스 생성
CREATE INDEX ix_cs_demo_status ON cs_demo(status);

-- 통계 정보 수집 (히스토그램 포함)
BEGIN
  DBMS_STATS.GATHER_TABLE_STATS(
    ownname    => USER,
    tabname    => 'CS_DEMO',
    method_opt => 'for columns status size 254',  -- 히스토그램 생성
    cascade    => TRUE
  );
END;
/
```

---

## 커서 공유의 기본 개념

### 커서의 구조와 유형
커서는 SQL 문장의 실행 가능한 표현입니다. Oracle에서는 두 가지 주요 유형의 커서가 있습니다:

1. **Parent Cursor**: SQL 텍스트 자체를 식별하는 상위 객체
2. **Child Cursor**: 특정 실행 계획과 환경 설정을 포함하는 실제 실행 객체

동일한 Parent Cursor 아래에 여러 Child Cursor가 존재할 수 있으며, 각 Child는 다른 실행 계획을 가질 수 있습니다.

### 커서 공유의 중요성
효과적인 커서 공유는 다음과 같은 이점을 제공합니다:
- **파싱 오버헤드 감소**: SQL 문장의 반복적 파싱 비용 절감
- **메모리 효율성**: Shared Pool의 메모리 사용 최적화
- **성능 향상**: 라이브러리 캐시 경합 감소
- **실행 계획 안정성**: 일관된 성능 예측 가능

---

## 바인드 변수와 리터럴 값의 비교

### 리터럴 값의 문제점
리터럴 값을 사용한 SQL 문장은 각기 다른 텍스트로 인식되어 커서 공유가 불가능합니다:

```sql
-- 각 문장은 다른 SQL로 처리됨
SELECT COUNT(*) FROM cs_demo WHERE status='HOT';
SELECT COUNT(*) FROM cs_demo WHERE status='COLD';
SELECT COUNT(*) FROM cs_demo WHERE status='WARM';
```

이러한 접근 방식은 다음과 같은 문제를 야기합니다:
- Shared Pool 메모리 낭비
- 파싱 오버헤드 증가
- 라이브러리 캐시 경합 발생 가능성 증가

### 바인드 변수의 적절한 사용
바인드 변수를 사용하면 동일한 SQL 템플릿을 재사용할 수 있습니다:

```sql
-- 바인드 변수 사용 예제
VAR status_param VARCHAR2(10)
EXEC :status_param := 'HOT';

SELECT COUNT(*)
FROM cs_demo
WHERE status = :status_param;
```

바인드 변수의 장점:
- SQL Injection 방지
- 커서 공유 가능
- 파싱 오버헤드 감소
- 메모리 사용 최적화

---

## CURSOR_SHARING 파라미터 이해

### 파라미터 값별 특성

```sql
-- 현재 CURSOR_SHARING 설정 확인
SHOW PARAMETER cursor_sharing;
```

**EXACT (기본값)**
- 정확히 동일한 SQL 텍스트만 공유
- 가장 안정적이고 예측 가능한 동작
- 애플리케이션이 바인드 변수를 적절히 사용할 때 권장

**FORCE**
- 리터럴 값을 시스템 생성 바인드 변수로 자동 변환
- 레거시 애플리케이션에서 유용할 수 있음
- 히스토그램 기반 최적화에 영향을 줄 수 있음

**SIMILAR (구식, 비권장)**
- 과거에는 선택도가 유사한 경우 공유하는 목적
- 현대 Oracle 버전에서 비효율적이고 비일관적
- 사용을 권장하지 않음

### 실용적 권장사항
1. 가능한 항상 **EXACT** 모드 사용
2. 바인드 변수를 애플리케이션 레벨에서 구현
3. FORCE 모드는 임시 조치로만 사용하고 근본적 해결 후 EXACT로 복귀

---

## Bind Peeking과 Adaptive Cursor Sharing (ACS)

### Bind Peeking의 동작 원리
Bind Peeking은 커서가 처음 파싱될 때 바인드 변수의 값을 확인하여 실행 계획을 결정하는 메커니즘입니다:

```sql
-- Bind Peeking 예제
VAR status_param VARCHAR2(10);

-- 첫 번째 실행: 'HOT' 값 (1% 데이터)
EXEC :status_param := 'HOT';
SELECT /* CS_DEMO */ COUNT(*) FROM cs_demo WHERE status = :status_param;

-- 두 번째 실행: 'COLD' 값 (99% 데이터)
EXEC :status_param := 'COLD';
SELECT /* CS_DEMO */ COUNT(*) FROM cs_demo WHERE status = :status_param;
```

**문제점**: 첫 번째 바인드 값('HOT')에 기반한 실행 계획이 두 번째 실행('COLD')에 부적절할 수 있습니다.

### Adaptive Cursor Sharing (ACS)
ACS는 이 문제를 해결하기 위해 도입된 메커니즘입니다:

1. **Bind-Sensitive Cursor**: 실행 중 통계 피드백을 수집
2. **Bind-Aware Cursor**: 수집된 피드백을 기반으로 최적의 실행 계획 선택
3. **다중 Child Cursor**: 다른 바인드 값 범위에 대해 다른 실행 계획 생성

```sql
-- ACS 상태 확인
SELECT sql_id, child_number, is_bind_sensitive, is_bind_aware, is_shareable
FROM v$sql
WHERE sql_text LIKE 'SELECT /* CS_DEMO */%';
```

---

## 세션 커서 캐싱 최적화

### 세션 커서 캐시의 역할
세션 커서 캐시는 개별 세션 수준에서 최근 사용한 커서 핸들을 저장하여 재사용성을 높입니다:

```sql
-- 세션 커서 캐시 관련 파라미터 확인
SHOW PARAMETER session_cached_cursors;

-- 권장 설정 (워크로드에 따라 조정)
ALTER SYSTEM SET session_cached_cursors = 100 SCOPE=BOTH;
```

### 성능 모니터링
```sql
-- 세션별 커서 캐시 효율성 분석
SELECT sid, 
       name, 
       value
FROM v$sesstat s
JOIN v$statname n ON s.statistic# = n.statistic#
WHERE n.name IN ('session cursor cache hits', 
                 'parse count (total)', 
                 'parse count (hard)')
  AND s.sid = SYS_CONTEXT('USERENV', 'SID');
```

**최적화 지표**:
- 높은 `session cursor cache hits` 값
- 낮은 `parse count (hard)` 비율
- 적절한 `session_cached_cursors` 설정

---

## 일반적인 문제와 해결책

### 문제 1: 과도한 Child Cursor 생성

**증상**:
- `v$sql`에서 높은 `version_count`
- 라이브러리 캐시 경합 증가
- 메모리 사용량 증가

**원인 분석**:
```sql
-- Child Cursor 분리 원인 조사
SELECT * 
FROM v$sql_shared_cursor 
WHERE sql_id = '&sql_id';
```

**해결책**:
1. 바인드 변수 데이터 타입 일관성 유지
2. NLS 파라미터 표준화
3. 애플리케이션 연결 풀 설정 통일

### 문제 2: 바인드 변수 사용 시 성능 저하

**증상**:
- 바인드 변수 도입 후 특정 쿼리 성능 저하
- 실행 계획이 최적이 아닌 경우

**해결책**:
1. 히스토그램 통계 정확성 확인
```sql
BEGIN
  DBMS_STATS.GATHER_TABLE_STATS(
    ownname => USER,
    tabname => 'YOUR_TABLE',
    method_opt => 'FOR COLUMNS YOUR_COLUMN SIZE 254'
  );
END;
/
```

2. Adaptive Cursor Sharing 활성화 확인
3. 필요시 SQL Profile 또는 SQL Plan Baseline 적용

### 문제 3: 파싱 오버헤드 과다

**증상**:
- 높은 `parse count (total)` 값
- CPU 사용량 증가
- 응답 시간 저하

**해결책**:
1. 바인드 변수 사용 강화
2. 세션 커서 캐시 크기 최적화
3. 애플리케이션 레벨 Statement 캐싱 구현

---

## 애플리케이션 구현 패턴

### Java (JDBC) 구현
```java
// 올바른 바인드 변수 사용
public int getOrderCount(String status) throws SQLException {
    String sql = "SELECT COUNT(*) FROM orders WHERE status = ?";
    
    try (Connection conn = dataSource.getConnection();
         PreparedStatement pstmt = conn.prepareStatement(sql)) {
        
        pstmt.setString(1, status);  // 바인드 변수 사용
        
        try (ResultSet rs = pstmt.executeQuery()) {
            if (rs.next()) {
                return rs.getInt(1);
            }
        }
    }
    return 0;
}
```

### Python 구현
```python
import oracledb

def get_order_count(status):
    sql = "SELECT COUNT(*) FROM orders WHERE status = :status"
    
    with oracledb.connect(user=USER, password=PASSWORD, dsn=DSN) as conn:
        with conn.cursor() as cursor:
            cursor.execute(sql, status=status)  # 명명된 바인드 변수
            return cursor.fetchone()[0]
```

### PL/SQL 구현
```plsql
CREATE OR REPLACE FUNCTION get_order_count(
    p_status IN VARCHAR2
) RETURN NUMBER IS
    v_count NUMBER;
BEGIN
    EXECUTE IMMEDIATE 
        'SELECT COUNT(*) FROM orders WHERE status = :1'
        INTO v_count
        USING p_status;  -- USING 절로 바인드 변수 전달
        
    RETURN v_count;
END;
/
```

---

## 성능 모니터링과 튜닝

### 시스템 레벨 모니터링
```sql
-- 시스템 전체 파싱 통계
SELECT name, 
       value,
       ROUND(value * 100 / SUM(value) OVER (), 2) as percentage
FROM v$sysstat
WHERE name LIKE 'parse%'
ORDER BY value DESC;

-- 라이브러리 캐시 효율성
SELECT namespace,
       gets,
       gethits,
       ROUND(gethitratio * 100, 2) as hit_ratio_percent,
       pins,
       pinhits,
       ROUND(pinhitratio * 100, 2) as pin_hit_ratio_percent
FROM v$librarycache
WHERE namespace IN ('SQL AREA', 'TABLE/PROCEDURE', 'BODY', 'TRIGGER');
```

### SQL 레벨 분석
```sql
-- 상위 파싱 SQL 식별
SELECT sql_id,
       SUBSTR(sql_text, 1, 50) as sql_snippet,
       parse_calls,
       executions,
       ROUND(parse_calls/NULLIF(executions, 0), 2) as parse_per_exec,
       version_count,
       loaded_versions
FROM v$sqlarea
WHERE parse_calls > 1000
ORDER BY parse_calls DESC
FETCH FIRST 20 ROWS ONLY;
```

### 경합 이벤트 모니터링
```sql
-- 라이브러리 캐시 관련 경합 이벤트
SELECT event,
       total_waits,
       time_waited_micro/1000000 as seconds_waited,
       ROUND(time_waited_micro/total_waits/1000, 2) as avg_ms_per_wait
FROM v$system_event
WHERE event LIKE 'library cache%' 
   OR event LIKE 'cursor:%'
   OR event LIKE '%mutex%'
ORDER BY time_waited_micro DESC;
```

---

## 실전 최적화 전략

### 전략 1: 점진적 바인드 변수 도입
레거시 애플리케이션을 위한 접근 방식:
1. `CURSOR_SHARING=FORCE`로 임시 적용
2. AWR/ASH 리포트로 핫 SQL 식별
3. 상위 SQL부터 순차적으로 바인드 변수 변환
4. 변환 완료 후 `CURSOR_SHARING=EXACT`로 복귀

### 전략 2: 하이브리드 접근법
데이터 분포가 편향된 컬럼 처리:
```sql
-- 자주 사용되는 값은 리터럴로 처리
IF p_status = 'HOT' THEN
    EXECUTE IMMEDIATE 
        'SELECT /*+ INDEX(orders status_idx) */ COUNT(*) 
         FROM orders WHERE status = ''HOT'''
        INTO v_count;
ELSE
    -- 그 외 값은 바인드 변수 사용
    EXECUTE IMMEDIATE 
        'SELECT COUNT(*) FROM orders WHERE status = :1'
        INTO v_count
        USING p_status;
END IF;
```

### 전략 3: 실행 계획 안정화
```sql
-- SQL Plan Baseline 적용
DECLARE
    v_plan pls_integer;
BEGIN
    v_plan := DBMS_SPM.LOAD_PLANS_FROM_CURSOR_CACHE(
        sql_id          => '&sql_id',
        plan_hash_value => &plan_hash_value
    );
    DBMS_OUTPUT.PUT_LINE('Loaded ' || v_plan || ' plans');
END;
/
```

---

## 결론: 효과적인 커서 공유를 위한 종합 전략

커서 공유 최적화는 Oracle 데이터베이스 성능 관리의 핵심 요소입니다. 효과적인 구현을 위해 다음 원칙을 준수하세요:

### 1. 기본 원칙 확립
- **바인드 변수 우선**: 모든 새로운 개발에서는 바인드 변수를 기본으로 사용
- **파라미터 설정**: `CURSOR_SHARING=EXACT` 유지, 필요한 경우에만 FORCE 임시 적용
- **타입 일치**: 바인드 변수 데이터 타입을 컬럼 타입과 정확히 일치시킴

### 2. 모니터링과 측정
- **정기적 모니터링**: 파싱 통계, 라이브러리 캐시 효율, 경합 이벤트 추적
- **성과 측정**: 변경 전후 성능 지표 비교 (응답 시간, 처리량, CPU 사용률)
- **문제 식별**: 높은 `version_count`, 파싱 오버헤드, 경합 이벤트 조기 발견

### 3. 상황별 최적화 전략
- **OLTP 시스템**: 바인드 변수 적극 활용, 세션 커서 캐시 최적화
- **데이터 웨어하우스**: Adaptive Cursor Sharing 활용, 히스토그램 통계 정확성 유지
- **하이브리드 환경**: 핫 값은 리터럴 처리, 나머지는 바인드 변수 사용

### 4. 지속적인 개선 사이클
1. **측정**: 현재 상태의 성능 기준선 설정
2. **분석**: 문제점과 개선 기회 식별
3. **실험**: 제한된 환경에서 변경 사항 테스트
4. **적용**: 검증된 개선사항 운영 환경 반영
5. **검증**: 변경 효과 지속적 모니터링

### 5. 조직적 접근
- **개발 표준**: 코딩 표준에 바인드 변수 사용 명시
- **교육**: 개발자 대상 최적화 기법 교육
- **자동화**: 정기적 성능 점검 및 튜닝 자동화
- **문서화**: 최적화 결정과 결과 체계적 문서화

커서 공유 최적화는 단순한 기술적 조치를 넘어 데이터베이스 성능 문화의 일부입니다. 지속적인 관심과 체계적인 접근을 통해 안정적이고 효율적인 데이터베이스 환경을 구축할 수 있습니다. 모든 최적화 결정은 실제 성능 측정과 비즈니스 가치 창출에 기반해야 합니다.