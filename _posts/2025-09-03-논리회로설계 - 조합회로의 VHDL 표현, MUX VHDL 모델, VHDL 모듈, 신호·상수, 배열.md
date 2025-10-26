---
layout: post
title: 논리회로설계 - 조합회로의 VHDL 표현, MUX VHDL 모델, VHDL 모듈, 신호·상수, 배열
date: 2025-09-03 21:25:23 +0900
category: 논리회로설계
---
# VHDL 소개 — **조합회로의 VHDL 표현**, **멀티플렉서(MUX) VHDL 모델**, **VHDL 모듈(Entity/Architecture)**, **신호·상수**, **배열(Array)**

> 표기: \(+\)=OR, \(\cdot\) 또는 생략=AND, \(\overline{X}\)=NOT \(X\).  
> 권장 라이브러리:  
> ```vhdl
> library ieee;
> use ieee.std_logic_1164.all;
> use ieee.numeric_std.all; -- signed/unsigned, to_integer 등
> ```
> *주의:* `std_logic_arith`, `std_logic_unsigned`는 벤더 확장입니다. **`numeric_std`**를 표준으로 사용하세요.

---

## 1) 조합회로의 VHDL 표현

### 1.1 동시문(Concurrent Statement) 3형식
1) **조건부 신호 할당(when-else)** — 간단한 2:1 선택/다중 조건
```vhdl
y <= a when sel = '0' else b;
z <= (a and b) when en = '1' else (others => '0');
```
2) **선택적 신호 할당(with-select)** — case 스타일
```vhdl
with sel select
  y <= d0 when "00",
       d1 when "01",
       d2 when "10",
       d3 when others;
```
3) **동시문 = 프로세스 변환 가능** — 동시문은 내부적으로 **프로세스**로 변환되어 해석됩니다.

### 1.2 조합 프로세스(Combinational Process) 정석
- **민감도 목록**은 **VHDL-2008**의 `process(all)` 사용을 권장(빠짐 방지).  
- 모든 출력에 대해 **완전할당**(else/others 포함) 또는 **초기 기본값**을 주어 **래치 유도 방지**.

```vhdl
architecture rtl of comb_ex is
  signal y : std_logic;
begin
  p_comb : process(all) is  -- VHDL-2008
  begin
    -- 기본값(권장)
    y <= '0';
    if a = '1' and b = '1' then
      y <= '1';
    end if;
  end process;
end architecture;
```

### 1.3 지연 모델(시뮬레이션 전용)
- **관성 지연**(기본): 짧은 펄스는 필터링
- **수송 지연**: 펄스 그대로 전달
```vhdl
y <= reject 0 ns inertial a after 2 ns; -- 관성(기본)
y <= transport a after 2 ns;            -- 수송
```
> 합성(하드웨어 구현) 대상 코드에는 **지연 지정**을 쓰지 않습니다. 테스트벤치용입니다.

### 1.4 조합회로 예시 (SOP/POS)
```vhdl
-- F = A·B + A'·C  (정적-1 해저드 방지용 합의항 +B·C 도 가능)
f <= (a and b) or ((not a) and c);
```

---

## 2) 멀티플렉서(MUX) VHDL 모델

### 2.1 2:1 MUX (세 가지 스타일)

**(1) when-else**
```vhdl
y <= d0 when sel = '0' else d1;
```

**(2) with-select**
```vhdl
with sel select
  y <= d0 when '0',
       d1 when others;
```

**(3) 조합 프로세스 + case**
```vhdl
p_mux2 : process(all) is
begin
  case sel is
    when '0'    => y <= d0;
    when others => y <= d1;
  end case;
end process;
```

### 2.2 4:1 MUX (가독성 좋게)
```vhdl
entity mux4 is
  generic (WIDTH : positive := 8);
  port (
    d0,d1,d2,d3 : in  std_logic_vector(WIDTH-1 downto 0);
    sel         : in  std_logic_vector(1 downto 0);
    y           : out std_logic_vector(WIDTH-1 downto 0)
  );
end entity;

architecture rtl of mux4 is
begin
  with sel select
    y <= d0 when "00",
         d1 when "01",
         d2 when "10",
         d3 when others;
end architecture;
```

### 2.3 *N:1* 파라메트릭 MUX (VHDL-2008, **unconstrained 배열** 사용)

#### 2.3.1 공용 타입 정의(패키지)
```vhdl
package util_pkg is
  -- 가변 길이 벡터의 배열(요소 폭도 불특정, VHDL-2008)
  type slv_array_t is array (natural range <>) of std_logic_vector;
  -- 정수의 clog2 (간단 구현)
  function clog2(n : positive) return natural;
end package;

package body util_pkg is
  function clog2(n : positive) return natural is
    variable v : natural := 0; variable x : natural := n-1;
  begin
    while x > 0 loop v := v + 1; x := x / 2; end loop; return v;
  end;
end package body;
```

#### 2.3.2 N:1 MUX 본체
```vhdl
library ieee; use ieee.std_logic_1164.all; use ieee.numeric_std.all;
use work.util_pkg.all;

entity muxN is
  generic (
    N     : positive := 8;  -- 입력 개수
    WIDTH : positive := 16  -- 각 입력 폭
  );
  port (
    d   : in  slv_array_t(0 to N-1)(WIDTH-1 downto 0); -- N×WIDTH
    sel : in  std_logic_vector(clog2(N)-1 downto 0);
    y   : out std_logic_vector(WIDTH-1 downto 0)
  );
end entity;

architecture rtl of muxN is
begin
  process(all) is
    variable idx : natural;
  begin
    idx := to_integer(unsigned(sel));
    if idx < N then
      y <= d(idx);
    else
      y <= (others => '0'); -- 범위 밖 보호(합성 시 sel 폭이 맞으면 제거)
    end if;
  end process;
end architecture;
```
> *도구가 VHDL-2008을 제한한다면*, 배열 포트를 벡터로 평탄화하거나(예: `d_flat : in std_logic_vector(N*WIDTH-1 downto 0)`) 인덱싱으로 슬라이스하세요.

---

## 3) VHDL 모듈: **Entity / Architecture / (Configuration / Package)**

### 3.1 모듈의 구성
- **Entity**: **포트**와 **Generic**(파라미터) 선언 — *인터페이스*
- **Architecture**: 동작/구조 — *구현*
- (선택) **Configuration**: 특정 Entity에 대해 어떤 Architecture를 묶을지 지정(대부분 툴이 자동 매핑)
- (권장) **Package**: 공통 **타입/상수/함수/컴포넌트** 정의

```vhdl
entity and2 is
  port ( a,b : in  std_logic;
         y   : out std_logic );
end entity;

architecture rtl of and2 is
begin
  y <= a and b;
end architecture;
```

### 3.2 컴포넌트/직접 인스턴스
- **직접 인스턴스(권장)** — 이름 충돌·설정 간소화
```vhdl
u0 : entity work.mux4
  generic map ( WIDTH => 8 )
  port map    ( d0 => x0, d1 => x1, d2 => x2, d3 => x3, sel => s, y => y );
```
- **컴포넌트 선언**은 구형 스타일(여전히 사용 가능).

### 3.3 구조적 설계(계층화) 예
```vhdl
entity top is
  port ( din0,din1,din2,din3 : in  std_logic_vector(7 downto 0);
         s                    : in  std_logic_vector(1 downto 0);
         y                    : out std_logic_vector(7 downto 0));
end entity;

architecture structural of top is
begin
  u_mux : entity work.mux4
    generic map ( WIDTH => 8 )
    port map (d0=>din0, d1=>din1, d2=>din2, d3=>din3, sel=>s, y=>y);
end architecture;
```

### 3.4 테스트벤치(조합용 기본형)
```vhdl
library ieee; use ieee.std_logic_1164.all; use ieee.numeric_std.all;

entity tb_mux4 is end;
architecture sim of tb_mux4 is
  signal d0,d1,d2,d3,y : std_logic_vector(3 downto 0);
  signal sel           : std_logic_vector(1 downto 0);
begin
  dut : entity work.mux4
    generic map (WIDTH => 4)
    port map (d0=>d0, d1=>d1, d2=>d2, d3=>d3, sel=>sel, y=>y);

  -- 자극
  stim : process
  begin
    d0 <= "0001"; d1 <= "0010"; d2 <= "0100"; d3 <= "1000";
    for i in 0 to 3 loop
      sel <= std_logic_vector(to_unsigned(i,2));
      wait for 10 ns;
    end loop;
    wait;
  end process;
end architecture;
```

---

## 4) **신호(signal)와 상수(constant)** (+ 변수 variable 요약)

### 4.1 신호 vs 변수
| 항목 | **signal** | **variable** |
|---|---|---|
| 대입 | `<=` | `:=` |
| 업데이트 | **덋타 사이클(미세 시간)** 후 갱신 | 즉시 갱신(프로세스 지역) |
| 범위 | 아키텍처/블록 전역 | 프로세스/블록 지역 |
| 용도 | 하드웨어 간 **연결선** | 계산 임시값, 루프 카운터 |

```vhdl
process(all) is
  variable tmp : unsigned(7 downto 0);
begin
  tmp := unsigned(a) + unsigned(b);
  y   <= std_logic_vector(tmp);
end process;
```

> **델타 사이클**: 동일 시각 내에서 신호 할당이 **이벤트 순서**로 전파됩니다. 조합 프로세스에서 **변수**로 중간값을 만들면 불필요한 델타를 줄일 수 있습니다.

### 4.2 상수/제네릭
```vhdl
constant ZERO4 : std_logic_vector(3 downto 0) := (others => '0');

entity foo is
  generic (WIDTH : positive := 16);
  port (...);
end entity;
```

### 4.3 해상형/미해상형 타입
- **`std_logic`**: **해상(resolved)** 타입 — 다중 드라이버의 충돌을 `'X'`로 해석 가능 (버스·3state 모델링)
- **`std_ulogic`**: **미해상(unresolved)** — 다중 드라이버 금지(안전), but 합성 시 제약↑  
> 대부분의 RTL은 `std_logic(_vector)` 사용. 3상태/풀업/`'Z'` 모델링 가능.

### 4.4 수치 연산 (numeric_std)
- `unsigned`, `signed`로 캐스팅 후 연산
```vhdl
signal a,b : std_logic_vector(7 downto 0);
signal s   : std_logic_vector(8 downto 0);
s <= std_logic_vector( ('0' & unsigned(a)) + unsigned(b) );
```

---

## 5) 배열(Array) — **std_logic_vector**, 사용자 정의 배열, ROM/LUT

### 5.1 기본: `std_logic_vector`
- 선언: `signal v : std_logic_vector(7 downto 0);`
- 슬라이스: `v(3 downto 0)`, 방향: `downto`(MSB→LSB), `to`(LSB→MSB)
- 결합/복제:  
```vhdl
w <= x & y;                     -- Concatenate
k <= (others => '0');           -- aggregate
y <= (3 downto 0 => '1', others => '0'); -- 부분 집합만 지정
```

### 5.2 사용자 정의 1차/2차 배열
```vhdl
-- 1차: N개의 벡터
type slv_array_t is array (natural range <>) of std_logic_vector;
signal bus : slv_array_t(0 to 3)(7 downto 0);

-- 2차(행렬)도 가능(도구 지원 확인)
type matrix_t is array (natural range <>) of slv_array_t(0 to 3)(7 downto 0);
```
> 일부 구형 합성기는 **중첩 비제약 배열**을 제한합니다. 문제가 있으면 1차원으로 평탄화하세요.

### 5.3 ROM/LUT를 **배열 상수**로
```vhdl
type rom_t is array (natural range <>) of std_logic_vector(6 downto 0);
constant SEVSEG : rom_t(0 to 15) := (
  0  => "1111110", 1  => "0110000", 2  => "1101101", 3  => "1111001",
  4  => "0110011", 5  => "1011011", 6  => "1011111", 7  => "1110000",
  8  => "1111111", 9  => "1111011", 10 => "1110111", 11 => "0011111",
  12 => "1001110", 13 => "0111101", 14 => "1001111", 15 => "1000111"
);

signal seg : std_logic_vector(6 downto 0);
seg <= SEVSEG(to_integer(unsigned(bin)));
```

### 5.4 배열과 **Generate** (반복 인스턴스)
```vhdl
gen_xor : for i in 0 to WIDTH-1 generate
  y(i) <= a(i) xor b(i);
end generate;
```

---

## 6) 실전 레시피 — **합성 친화 조합 템플릿** & **자주 하는 실수**

### 6.1 조합 템플릿 (VHDL-2008)
```vhdl
architecture rtl of foo is
begin
  p_comb : process(all) is
  begin
    -- 1) 기본값(모든 출력)
    y <= (others => '0');
    -- 2) 조건/케이스 완전 할당
    case sel is
      when "00"   => y <= a;
      when "01"   => y <= b;
      when "10"   => y <= a or b;
      when others => y <= (others => '0');
    end case;
  end process;
end architecture;
```

### 6.2 흔한 실수와 예방책
- **민감도 누락**(VHDL-93): `process(a,b)`에 `c`를 빼먹어 **래치** 유도 → **`process(all)`** 사용.
- **불완전 할당**: some 분기에서 출력 미할당 → 래치 생성 → **초기 기본값** 또는 **others** 필수.
- **형 변환 누락**: `std_logic_vector`끼리 **덧셈 불가** → `unsigned()`/`signed()` 변환.
- **`after`/`wait for`를 RTL에 사용**: 합성 불가 → 테스트벤치에서만 사용.

---

## 7) MUX 설계—스타일 비교 요약

| 스타일 | 코드량 | 가독성 | 확장성 | 합성 품질 |
|---|---|---|---|---|
| `when-else` | 짧음 | 단순 2:1에 최적 | 낮음 | 우수 |
| `with-select` | 깔끔 | 4:1, 8:1 등 | 보통 | 우수 |
| `process+case` | 유연 | 디폴트/예외 처리 용이 | 높음 | 우수 |
| 파라메트릭 배열 | 모듈화 | 가장 깔끔 | 매우 높음 | 도구/VHDL-2008 의존 |

---

## 8) 보너스: **패키지/유틸** 모듈화 예

```vhdl
package math_pkg is
  function max_nat(a,b : natural) return natural;
  function min_nat(a,b : natural) return natural;
end package;

package body math_pkg is
  function max_nat(a,b : natural) return natural is
  begin return (a>b) ? a : b; end;
  function min_nat(a,b : natural) return natural is
  begin return (a<b) ? a : b; end;
end package body;
```

패키지를 사용하면 **타입/상수/함수**를 여러 설계에서 재사용하고, 엔티티는 **깨끗한 인터페이스**만 유지할 수 있습니다.

---

## 9) 종합 예제 — **조합 논리 & MUX & 배열** 한 번에

```vhdl
-- 4채널 입력에서 조건에 따라 선택/연산을 수행하는 조합 블록
library ieee; use ieee.std_logic_1164.all; use ieee.numeric_std.all;

entity comb_block is
  generic (WIDTH : positive := 8);
  port (
    d   : in  std_logic_vector(4*WIDTH-1 downto 0); -- 평탄화 입력
    sel : in  std_logic_vector(1 downto 0);
    op  : in  std_logic_vector(1 downto 0); -- 00:pass, 01:inv, 10:or with ch0, 11:xor with ch1
    y   : out std_logic_vector(WIDTH-1 downto 0)
  );
end entity;

architecture rtl of comb_block is
  function slice(slv : std_logic_vector; i, w : natural) return std_logic_vector is
    variable lo : natural := i*w;
    variable hi : natural := (i+1)*w - 1;
  begin
    return slv(hi downto lo);
  end;
  signal ch0,ch1,ch2,ch3,tmp : std_logic_vector(WIDTH-1 downto 0);
begin
  ch0 <= slice(d,0,WIDTH);
  ch1 <= slice(d,1,WIDTH);
  ch2 <= slice(d,2,WIDTH);
  ch3 <= slice(d,3,WIDTH);

  -- 4:1 MUX
  with sel select
    tmp <= ch0 when "00",
           ch1 when "01",
           ch2 when "10",
           ch3 when others;

  -- 후처리 연산
  process(all) is
  begin
    case op is
      when "00"   => y <= tmp;
      when "01"   => y <= not tmp;
      when "10"   => y <= tmp or ch0;
      when others => y <= tmp xor ch1;
    end case;
  end process;
end architecture;
```

---

### 포켓 요약
- **조합 표현**: `when-else`, `with-select`, `process(all)` + 완전할당 = 래치 방지.  
- **MUX**: 2:1~N:1까지 스타일별 구현, VHDL-2008의 **비제약 배열**로 깔끔한 파라메트릭화.  
- **모듈**: Entity(인터페이스) + Architecture(구현) + Package(공용 타입/함수).  
- **신호/상수**: `signal`(<=, 델타), `variable`(:=, 즉시), `constant`, `generic`. `numeric_std`로 수치화.  
- **배열**: `std_logic_vector` 슬라이스/결합, 사용자 배열로 ROM·버스·N:1 MUX 표현.