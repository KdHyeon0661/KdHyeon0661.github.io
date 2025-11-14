---
layout: post
title: 논리회로설계 - 조합회로의 VHDL 표현, MUX VHDL 모델, VHDL 모듈, 신호·상수, 배열
date: 2025-09-03 21:25:23 +0900
category: 논리회로설계
---
# VHDL 소개 — **조합회로의 VHDL 표현**, **멀티플렉서(MUX) VHDL 모델**, **VHDL 모듈(Entity/Architecture)**, **신호·상수**, **배열(Array)**

> 권장 라이브러리(표준, 합성/시뮬 공통)
> ```vhdl
> library ieee;
> use ieee.std_logic_1164.all;
> use ieee.numeric_std.all;  -- signed/unsigned, to_integer, resize, shift_left/right 등
> ```
> 주의: `std_logic_arith`, `std_logic_unsigned/signed`는 비표준 벤더 확장입니다. 혼용하지 말고 `numeric_std`만 사용하세요.

---

## 조합회로의 VHDL 표현

### 동시문(Concurrent Statements) 3형식 요약

동시문은 **회로의 지속적 구동**을 의미합니다(=배선/게이트에 항상 걸려 있는 드라이버).

1) **조건부 신호 할당(when–else)** — 2:1 선택, 간단한 보호 로직
```vhdl
y <= a when sel = '0' else b;                         -- 2:1
z <= (a and b) when en = '1' else (others => '0');    -- EN 보호
```

2) **선택적 신호 할당(with–select)** — case 스타일, one-hot/코드 선택
```vhdl
with sel select
  y <= d0 when "00",
       d1 when "01",
       d2 when "10",
       d3 when others;
```

3) **동시문 ↔ 프로세스(조합) 등가** — 도구는 동시문을 내부적으로 **조합 프로세스**로 변환하여 해석합니다.

> 팁
> - 간단한 2:1/4:1 MUX→ `when-else`/`with-select`.
> - 복수 출력/디폴트/예외처리 많음→ `process(all) + case`가 명확.

---

### 조합 프로세스(Combinational Process) 정석 패턴

- **민감도 목록**: VHDL-2008의 `process(all)` 권장(누락 방지).
- **완전할당**: 모든 출력에 기본값 또는 모든 분기에서 값 할당 → **래치 방지**.

```vhdl
architecture rtl of comb_ex is
  signal y : std_logic;
begin
  p_comb : process(all) is
  begin
    y <= '0';                              -- ① 기본값
    if a = '1' and b = '1' then           -- ② 모든 경로에서 y에 값
      y <= '1';
    end if;                                -- ⇒ 래치 유도 없음
  end process;
end architecture;
```

> 래치가 필요 없다면 **항상** 기본값/others를 두고 “완전할당”을 지키세요.

---

### 시뮬레이션 지연 모델(테스트벤치 전용)

- **관성(inertial)**: 짧은 펄스를 소거(게이트 모델, 기본)
- **수송(transport)**: 펄스를 그대로 전달(배선 모델)
```vhdl
y <= reject 0 ns inertial a after 2 ns;  -- 관성(기본)
y <= transport a after 2 ns;             -- 수송
```
> 합성 대상 RTL에는 `after`, `wait for`를 쓰지 않습니다. (테스트벤치에서만)

---

### SOP/POS, 간단 예

```vhdl
-- F = A·B + A'·C  (정적-1 해저드 방지: +B·C를 합의항으로 추가 가능)
f <= (a and b) or ((not a) and c);
```

---

## 멀티플렉서(MUX) VHDL 모델

### 2:1 MUX — 3가지 스타일

**(1) when–else**
```vhdl
y <= d0 when sel = '0' else d1;
```
**(2) with–select**
```vhdl
with sel select
  y <= d0 when '0',
       d1 when others;
```
**(3) process + case**
```vhdl
p_mux2 : process(all) is
begin
  case sel is
    when '0'    => y <= d0;
    when others => y <= d1;
  end case;
end process;
```
> 세 방식은 **합성 결과 동일**(도구 최적화 전제). 팀 컨벤션에 맞게 선택하세요.

---

### 4:1 MUX — 가독성 우선 템플릿

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
         d3 when others;  -- 디폴트/예외 처리
end architecture;
```

---

### *N:1* 파라메트릭 MUX — VHDL-2008 **비제약 배열** 사용
#### 유틸 패키지(배열 타입 + clog2)

```vhdl
package util_pkg is
  type slv_array_t is array (natural range <>) of std_logic_vector;
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

#### N:1 MUX 본체

```vhdl
library ieee; use ieee.std_logic_1164.all; use ieee.numeric_std.all;
use work.util_pkg.all;

entity muxN is
  generic (N : positive := 8; WIDTH : positive := 16);
  port (
    d   : in  slv_array_t(0 to N-1)(WIDTH-1 downto 0);
    sel : in  std_logic_vector(clog2(N)-1 downto 0);
    y   : out std_logic_vector(WIDTH-1 downto 0)
  );
end;

architecture rtl of muxN is
begin
  process(all) is
    variable idx : natural;
  begin
    idx := to_integer(unsigned(sel));
    if idx < N then y <= d(idx);
    else            y <= (others => '0');  -- 보호
    end if;
  end process;
end;
```
> 구형 도구라면 포트를 평탄화(`d_flat : std_logic_vector(N*WIDTH-1 downto 0)`)하고 슬라이스 함수로 대체하세요.

---

### 우선순위 MUX (우선 선택/디폴트 포함)

```vhdl
-- sel3 > sel2 > sel1 > sel0 우선순위
process(all) is
begin
  y <= d0;                               -- 디폴트
  if    sel0 = '1' then y <= d0;
  elsif sel1 = '1' then y <= d1;
  elsif sel2 = '1' then y <= d2;
  elsif sel3 = '1' then y <= d3;
  end if;
end process;
```
> 복수 `sel`이 1일 수 있는 **우선순위 인코딩** 상황에서 사용합니다(합성 결과: 프라이어리티 트리).

---

### 3-State와 외부 버스 (Top-level에만)

대부분의 FPGA 내부는 **진짜 3상태**를 지원하지 않으므로 내부는 **MUX로 합성**됩니다. 외부 핀만 tri-state를 사용하세요.
```vhdl
-- top 레벨 예: I/O 핀에만 'Z' 허용
y <= data_out when oe = '1' else (others => 'Z');
```

---

## VHDL 모듈: **Entity / Architecture** (+ Configuration/Package)

### 역할

- **Entity**: **포트/제네릭**을 정의하는 **인터페이스**
- **Architecture**: 동작/구조 기술(**구현 본문**)
- (옵션) **Configuration**: 특정 Entity에 사용할 Architecture/바인딩 지정
- (권장) **Package/Context**: 타입·상수·함수·컴포넌트 등 공용 정의

```vhdl
entity and2 is
  port (a,b : in  std_logic;
        y   : out std_logic);
end;

architecture rtl of and2 is
begin
  y <= a and b;
end;
```

### 직접 인스턴스(현대 권장) vs 컴포넌트 선언

```vhdl
-- 직접 인스턴스(권장, 간결/안전)
u0 : entity work.mux4(rtl)
  generic map ( WIDTH => 8 )
  port map    ( d0=>x0, d1=>x1, d2=>x2, d3=>x3, sel=>s, y=>y );
```
> 컴포넌트 선언은 레거시/혼합언어 환경에서 유지 보수 목적으로만 사용하세요.

### 구조적 계층화 예

```vhdl
entity top is
  port ( din0,din1,din2,din3 : in  std_logic_vector(7 downto 0);
         s                    : in  std_logic_vector(1 downto 0);
         y                    : out std_logic_vector(7 downto 0));
end;

architecture structural of top is
begin
  u_mux : entity work.mux4
    generic map ( WIDTH => 8 )
    port map (d0=>din0, d1=>din1, d2=>din2, d3=>din3, sel=>s, y=>y);
end;
```

### 테스트벤치(조합 장치용 기본형)

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

  stim : process
  begin
    d0 <= "0001"; d1 <= "0010"; d2 <= "0100"; d3 <= "1000";
    for i in 0 to 3 loop
      sel <= std_logic_vector(to_unsigned(i,2));
      wait for 10 ns;
    end loop;
    assert false report "done" severity failure;
  end process;
end;
```

---

## **신호(signal)·상수(constant)** (+ 변수 variable 요약)

### 신호 vs 변수 — 차이 표

| 항목 | **signal** | **variable** |
|---|---|---|
| 대입 기호 | `<=` | `:=` |
| 갱신 시점 | **델타 사이클 종료 후** | **즉시**(프로세스 내부) |
| 스코프 | 아키텍처/블록 전역 | 프로세스/블록 지역 |
| 사용 용도 | 모듈 간 연결선, 레지스터/와이어 | 중간 계산, 누산기, 루프 카운터 |

```vhdl
process(all) is
  variable tmp : unsigned(7 downto 0);
begin
  tmp := unsigned(a) + unsigned(b);          -- 즉시 계산
  y   <= std_logic_vector(tmp);              -- 신호로 출력 대입(델타 후 적용)
end process;
```
> **델타 사이클**을 이해하면 “왜 같은 시각에 값이 한 박자 늦게 보이나?” 문제를 쉽게 해결할 수 있습니다.

### 상수/제네릭

```vhdl
constant ZERO4 : std_logic_vector(3 downto 0) := (others => '0');

entity foo is
  generic (WIDTH : positive := 16);
  port (...);
end entity;
```

### 해상형/미해상형 타입 & 다중 드라이버

- **`std_logic(_vector)`**: **해상(resolved)** 타입 — 다중 드라이버 충돌시 `'X'` 등으로 해석(버스/3state 모델)
- **`std_ulogic(_vector)`**: **미해상(unresolved)** — 다중 드라이버 금지(안전하지만 제약↑)
> RTL 실무는 보통 `std_logic(_vector)`를 사용합니다.

### 수치 연산 — `numeric_std`만

```vhdl
signal a,b : std_logic_vector(7 downto 0);
signal s   : std_logic_vector(8 downto 0);

s <= std_logic_vector(('0' & unsigned(a)) + unsigned(b));
lt <= '1' when signed(a) < signed(b) else '0';
```

---

## 배열(Array) — **std_logic_vector**, 사용자 정의 배열, ROM/LUT

### 기본 `std_logic_vector`

```vhdl
signal v : std_logic_vector(7 downto 0);
w <= x & y;                                -- 연결
k <= (others => '0');                      -- 전체 초기화
y <= (3 downto 0 => '1', others => '0');   -- 부분 지정
```
- 방향: `downto`(MSB→LSB), `to`(LSB→MSB). 팀 컨벤션을 유지하세요.

### 사용자 정의 배열(비제약 1차/2차)

```vhdl
-- 1차: N개의 벡터
type slv_array_t is array (natural range <>) of std_logic_vector;
signal bus : slv_array_t(0 to 3)(7 downto 0);

-- 2차(행렬) 예 — 도구 지원 확인
type mat_t is array (natural range <>) of slv_array_t(0 to 3)(7 downto 0);
```
> 일부 합성기는 “중첩 비제약 배열”에 제한이 있습니다. 문제가 생기면 1차 벡터로 평탄화하고, 슬라이스 함수로 다루세요.

### ROM/LUT를 **배열 상수**로

```vhdl
type rom_t is array (natural range <>) of std_logic_vector(6 downto 0);
constant SEVSEG : rom_t(0 to 15) := (
  0=>"1111110", 1=>"0110000", 2=>"1101101", 3=>"1111001",
  4=>"0110011", 5=>"1011011", 6=>"1011111", 7=>"1110000",
  8=>"1111111", 9=>"1111011", 10=>"1110111", 11=>"0011111",
  12=>"1001110", 13=>"0111101", 14=>"1001111", 15=>"1000111"
);

seg <= SEVSEG(to_integer(unsigned(bin)));
```

### Generate — 패턴 반복/벡터화

```vhdl
gen_bit : for i in 0 to WIDTH-1 generate
  y(i) <= a(i) xor b(i);
end generate;
```

---

## 실전 레시피 — **합성 친화 조합 템플릿** & 함정 회피

### 조합 템플릿(완전할당, VHDL-2008)

```vhdl
architecture rtl of foo is
begin
  p_comb : process(all) is
  begin
    y <= (others => '0');                -- 기본값
    case sel is
      when "00"   => y <= a;
      when "01"   => y <= b;
      when "10"   => y <= a or b;
      when others => y <= (others => '0');
    end case;
  end process;
end architecture;
```

### 흔한 실수 → 예방책

- **민감도 누락**(VHDL-93): `process(a,b)`에서 `c` 빠짐 → 래치 유도 → **`process(all)` 사용**
- **불완전 할당**: 어떤 분기에서 출력 미할당 → 래치 생성 → **기본값/others** 추가
- **형 변환 누락**: `std_logic_vector`끼리 덧셈 → **반드시 `unsigned()`/`signed()` 캐스팅**
- **지연/대기**를 RTL에 사용: 합성 불가 → **테스트벤치에서만** 사용

---

## 스타일 비교: MUX 구현 선택 가이드

| 스타일          | 코드량 | 가독성      | 확장성     | 합성 품질 |
|-----------------|--------|-------------|------------|-----------|
| `when-else`     | 짧음   | 2:1 최적    | 낮음       | 우수      |
| `with-select`   | 깔끔   | 4:1, 8:1    | 보통       | 우수      |
| `process+case`  | 유연   | 디폴트/예외 | 높음       | 우수      |
| 파라메트릭 배열 | 모듈화 | 가장 깔끔   | 매우 높음  | 도구 의존 |

---

## 보너스: 유틸 패키지 모듈화(자주 쓰는 함수)

```vhdl
package math_pkg is
  function max_nat(a,b : natural) return natural;
  function min_nat(a,b : natural) return natural;
  function slice(slv : std_logic_vector; i, w : natural)
    return std_logic_vector;  -- 평탄화 벡터 슬라이스
end package;

package body math_pkg is
  function max_nat(a,b : natural) return natural is
  begin return (a>b) ? a : b; end;

  function min_nat(a,b : natural) return natural is
  begin return (a<b) ? a : b; end;

  function slice(slv : std_logic_vector; i, w : natural)
    return std_logic_vector is
    variable lo : natural := i*w;
    variable hi : natural := (i+1)*w - 1;
  begin
    return slv(hi downto lo);
  end;
end package body;
```

---

## 종합 예제 — **조합 논리 + MUX + 배열(평탄화)**

요구: 4채널 입력 중 선택한 채널에 후처리 연산을 적용해 출력.
- `sel`: 4:1 MUX 선택
- `op`: 후처리(00 pass / 01 invert / 10 or ch0 / 11 xor ch1)

```vhdl
library ieee; use ieee.std_logic_1164.all; use ieee.numeric_std.all;

entity comb_block is
  generic (WIDTH : positive := 8);
  port (
    d   : in  std_logic_vector(4*WIDTH-1 downto 0); -- 평탄화 입력
    sel : in  std_logic_vector(1 downto 0);
    op  : in  std_logic_vector(1 downto 0);         -- 00 pass, 01 inv, 10 or ch0, 11 xor ch1
    y   : out std_logic_vector(WIDTH-1 downto 0)
  );
end;

architecture rtl of comb_block is
  function slice(slv : std_logic_vector; i, w : natural)
    return std_logic_vector is
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
end;
```

### 간단 테스트벤치(랜덤 자극)

```vhdl
library ieee; use ieee.std_logic_1164.all; use ieee.numeric_std.all;
library std;  use std.textio.all;
use ieee.math_real.all;  -- uniform()

entity tb is end;
architecture sim of tb is
  constant W : positive := 8;
  signal d   : std_logic_vector(4*W-1 downto 0);
  signal sel : std_logic_vector(1 downto 0);
  signal op  : std_logic_vector(1 downto 0);
  signal y   : std_logic_vector(W-1 downto 0);
begin
  dut: entity work.comb_block
    generic map (WIDTH => W)
    port map(d=>d, sel=>sel, op=>op, y=>y);

  stim: process
    variable r1,r2 : real := 0.5;
    variable u     : real;
    variable i     : integer;
  begin
    for t in 0 to 50 loop
      uniform(r1,r2,u);
      sel <= std_logic_vector(to_unsigned(integer(u*4.0) mod 4, 2));
      uniform(r1,r2,u);
      op  <= std_logic_vector(to_unsigned(integer(u*4.0) mod 4, 2));
      for i in 0 to 3 loop
        uniform(r1,r2,u);
        d((i+1)*W-1 downto i*W) <= std_logic_vector(to_unsigned(integer(u*256.0) mod 256, W));
      end loop;
      wait for 10 ns;
    end loop;
    assert false report "done" severity failure;
  end process;
end;
```

---

## 포켓 요약

- **조합 표현**: `when-else`, `with-select`, `process(all)+case` — 공통 핵심은 **완전할당**으로 래치 방지.
- **MUX**: 2:1~N:1까지 스타일별 구현. VHDL-2008 **비제약 배열**로 파라메트릭화 깔끔.
- **모듈 구조**: Entity(인터페이스) + Architecture(구현) + Package(Context로 묶기).
- **신호·상수**: signal(델타 후 갱신) vs variable(즉시); constant/generic로 재사용/파라메터화.
- **배열**: `std_logic_vector` 조합, 사용자 정의 배열, ROM/LUT, generate로 반복 패턴 처리.
- **합성/시뮬 규율**: RTL에 시간 제어 금지, 내부 3-state 지양, `numeric_std`만 사용.

---

### 부록 A: 수식 메모(우선순위/델타 개념)

- 연산자 **우선순위** \(높음→낮음\):
  $$ \*,/,\mathrm{mod},\mathrm{rem}\ >\ +,-,\&\ >\ \text{shift/rotate}\ >\ \text{relational}\ >\ \text{and}\ >\ \text{or,xor}\ >\ \text{nand,nor,xnor} $$
- **델타 사이클**: 물리 시간 \(t\)에서 이벤트 정착을 위해 \((t,\delta)\) 단계를 반복하여 안정점에 도달 후 \(t+\Delta t\)로 진행.
