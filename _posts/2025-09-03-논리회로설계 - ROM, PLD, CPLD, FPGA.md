---
layout: post
title: 논리회로설계 - ROM, PLD, CPLD, FPGA
date: 2025-09-03 20:25:23 +0900
category: 논리회로설계
---
# 읽기 전용 메모리(ROM) · 프로그래머블 논리소자(PLD) · 복합 프로그래머블 논리소자(CPLD) · 필드 프로그래머블 게이트어레이(FPGA) — 완전 정리

> 표기: \(+\)=OR, \(\cdot\) 또는 생략=AND, \(\overline{X}\)=NOT \(X\).
> 목표: **개념·구조·동작**, **프로그래밍(구성) 기술**, **지연·전력·해저드·보안**, **설계 흐름**과 **실전 예**까지 생략 없이 정리한다.
> 최신 기준 메모: 현대 범용 FPGA의 기본 LUT는 **6입력(일부 7/8분해 구조 포함)** 이 일반적이며, **SRAM 구성 + 비트스트림 암호화/인증**이 표준 관행이다. CPLD는 대체로 **비휘발성(Flash/E²)** 이고 결정적 타이밍을 제공한다.

---

## ROM(Read-Only Memory)

### 개념 — “진리표를 저장해 읽는다”

- 입력 \(n\)비트(주소) → 출력 \(m\)비트(데이터).
- 사실상 임의의 조합논리 \(F:\{0,1\}^{n}\to\{0,1\}^{m}\)를 **테이블**로 구현.
- 필요 비트 수: \(\boxed{2^{n}\times m}\)

### 내부 구조와 지연

```
입력(n) ──▶ 주소 디코더(n→2^n) ─▶ 워드라인 선택 ─▶ 메모리 셀(2^n×m) ─▶ 센스앰프 ─▶ 출력(m)
```
- 지연 모델
  \[
  t_{\text{ROM}} \approx t_{\text{decode}} + t_{\text{cell}} + t_{\text{sense}}
  \]
- **장점**: 다중 출력에 유리, 논리 복잡도와 무관하게 **내용만 바꾸면 기능 변경**.
- **단점**: 희소한 1/0 패턴에는 **비트 낭비**, 주소 전환 시 **글리치** 가능(디코더 경쟁).

### 종류와 프로그래밍

- **Mask ROM**: 제조 시 고정(대량/저가, 변경 불가).
- **PROM(퓨즈)**: 1회 프로그래밍(OTP).
- **EPROM(자외선 소거)**: 창 패키지, 전체 소거.
- **EEPROM/Flash**: 전기적 소거·쓰기(부분/블록).
- **FPGA의 ROM**: 분산RAM/LUTRAM/BRAM에 초기값 로딩으로 구현.

### 설계 팁(타이밍/전력/해저드)

- **출력 레지스터**로 동기화하여 글리치 흡수(주소 전환 시 안정화).
- 순차 접근/버스트는 **프리패치/파이프라인**으로 속도·전력 이득.
- FPGA 내부 ROM은 **동기 읽기** 추론이 일반적.

### 예: 4→7 세그먼트 디코더(동기 ROM)

```verilog
module seg7_rom(
  input  logic        clk,
  input  logic  [3:0] bin,   // 0..F
  output logic  [6:0] seg    // abcdefg (1=on)
);
  // 분산/블록 RAM로 추론될 수 있는 ROM
  logic [6:0] mem [0:15];
  initial begin
    mem[4'h0]=7'b1111110; mem[4'h1]=7'b0110000;
    mem[4'h2]=7'b1101101; mem[4'h3]=7'b1111001;
    mem[4'h4]=7'b0110011; mem[4'h5]=7'b1011011;
    mem[4'h6]=7'b1011111; mem[4'h7]=7'b1110000;
    mem[4'h8]=7'b1111111; mem[4'h9]=7'b1111011;
    mem[4'hA]=7'b1110111; mem[4'hB]=7'b0011111;
    mem[4'hC]=7'b1001110; mem[4'hD]=7'b0111101;
    mem[4'hE]=7'b1001111; mem[4'hF]=7'b1000111;
  end
  always_ff @(posedge clk) seg <= mem[bin];
endmodule
```

---

## PLD(Programmable Logic Device) — PLA/PAL/GAL

> 핵심: **AND-OR 배열**로 SOP(곱의 합) 논리를 재구성.
> 입력과 그 보수(\(X, \overline{X}\))가 AND 어레이로 들어가 **제품항**을 만들고, OR 어레이가 이를 합산.

### PLA vs PAL vs GAL

| 항목 | PLA | PAL | GAL |
|---|---|---|---|
| 배열 | AND **가변** + OR **가변** | AND **가변** + OR **고정(한정된 제품항)** | PAL 호환 + **재프로그래밍**(EEPROM/Flash) |
| 자유도/성능 | 자유도↑/지연↑ | 제한↑/지연↓/예측↑ | 현장 업데이트 용이 |
| 용도 | 복잡한 다중출력 | 빠른 고정 구조 | PAL 대체 |

### 구조 및 매크로셀

```
입력/보수 ─▶ AND 배열(제품항) ─▶ OR 배열 ─▶ (옵션) FF ─▶ 출력/피드백
```
- 매크로셀: 출력 폴라리티, OE, 레지스터 포함.
- 장점: **희소 진리표**에 효율적(필요한 제품항만).
- 해저드 대응: **합의항(consensus)** 을 추가해 정적-1 해저드 억제.

### 제품항 기반 예(공유 최적화)

\[
F_1=AB+\overline{A}C,\quad F_2=AC+BC
\]
- 공통 제품항 \(AC, BC\) 등을 **한번만 생성**해 여러 출력에 OR 배선.

```verilog
module pld_style(
  input  logic A,B,C,
  output logic F1,F2
);
  logic p_ab   = A & B;
  logic p_notA_C = (~A) & C;
  logic p_ac   = A & C;
  logic p_bc   = B & C;

  assign F1 = p_ab | p_notA_C; // AB + A' C
  assign F2 = p_ac | p_bc;     // AC + BC
endmodule
```

---

## CPLD(Complex PLD) — “큰 PAL 여러 개 + 전역 매트릭스”

### 구조 요약

- **Function Block**: 매크로셀 뭉치(제품항 OR + 레지스터).
- **Product-Term Allocator**: 제품항 공유/할당.
- **Global Interconnect**: 블록 간 연결(지연 일정).
- **I/O 블록**: 드라이브/슬루/표준, OE, IOB 레지스터.

### 특징과 용도

- **비휘발성**(대개 Flash/E²) → 전원 인가 즉시 동작.
- **결정적 타이밍**(경로 편차 작음) → 주소 디코딩, 프로토콜 glue, 보드 핀 리매핑에 최적.
- 넓은 OR/디코딩/상태 펄스 생성에 강함.

### 타이밍/전력/해저드

- 두 단계(SOP) + 라우팅 → **예측 가능**.
- 합의항 추가로 해저드 억제 가능.
- 전력은 FPGA 대비 낮은 편(규모가 작고 라우팅 단순).

---

## FPGA(Field-Programmable Gate Array)

> **LUT + 레지스터 + 거대한 라우팅 + 하드 블록(DSP/BRAM/PHY)**.
> **SRAM 구성**이 일반적(비트스트림 로딩), 일부는 **비휘발성(Flash/안티퓨즈)**.

### 기본 블록

- **LUT(K입력)**: K=6 일반적(일부 7/8분해). 임의의 \(K\)-변수 조합논리/소규모 ROM/LUTRAM/SRL.
- **FF/레지스터**: LUT 옆(파이프라인 용이).
- **BRAM/URAM**: 대용량 동기식 메모리.
- **DSP 슬라이스**: 멀티플라이어/누산기/프리어디더.
- **클록 네트워크**: 글로벌/리전, PLL/MMCM.
- **PHY/I/O**: DDR/SerDes/LVDS.
- **Carry Chain**: 덧셈기/카운터/XOR 고속화.

### 구성(Configuration)

- **SRAM형**: 비트스트림 로딩(외부/내부 Flash).
  - **암호화(AES)** + **인증(해시/서명)** 권장.
  - **부분 재구성(PR)** 로 동작 중 일부 재구성.
- **Flash/E²형**: 내장 비휘발성, 즉시 부트.
- **안티퓨즈**: OTP, 지연↓/보안↑, 재구성 불가.

### 타이밍/전력/해저드

- 타이밍은 **라우팅 품질** 좌우 → P&R와 **STA**가 핵심.
- 글리치 가능성↑ → **동기 설계**(레지스터 경계), **CDC** 엄격.
- 전력: **클록/토글** 지배적 → 클록 게이팅, CE, 파이프라인, 전압/온도 관리.

### HDL 설계 흐름

```
RTL(Verilog/VHDL) → 합성 → 배치·배선(P&R) → STA(타이밍 닫기) → 비트스트림 → 다운로드
             ↘ 시뮬레이션(기능/타이밍) ↙  ↘ ILA/SignalTap(온칩 디버그)
```

---

## ROM vs PLD vs CPLD vs FPGA — **선택 가이드**

| 항목 | ROM/EPROM/Flash | PLA/PAL/GAL (PLD) | **CPLD** | **FPGA** |
|---|---|---|---|---|
| 핵심 구조 | 디코더+메모리 | AND-OR 배열 | 매크로셀+전역 매트릭스 | LUT+라우팅+하드 블록 |
| 비휘발성 | Mask/PROM/EEPROM/Flash: 예 | 보통 예(Flash/E²) | 예(대부분) | SRAM형: 아니오 / Flash·안티퓨즈: 예 |
| 지연 특성 | 일정(디코더/센스) | 2단(SOP) | **결정적/예측 용이** | 라우팅 의존/가변(고성능) |
| 규모/유연성 | \(2^n m\) 테이블 | 소·중 규모 SOP | 중규모 glue/디코딩 | 대규모 병렬·DSP/메모리 |
| 재구성 | (E)EPROM/Flash는 가능 | 가능(GAL) | 가능 | 매우 자유(PR 포함) |
| 보안 | 내용 유출 시 복제 위험 | 낮음 | 중간 | **비트스트림 암/인증** 필수 |
| 대표 용도 | 상수맵/마이크로코드 | 주소/상태/펄스 | 버스/핀 리매핑 | 가속기/영상/통신/SoC |

---

## 비용 모델 · 지연 · 해저드 · 보안

### 면적/비트 비용

- **ROM**: \(\,2^{n}m\,\)비트(밀집 진리표에 유리).
- **PLD/CPLD**: **제품항 수**와 **리터럴**에 비례(희소 진리표에 유리).
- **FPGA**: LUT 개수(입력 수 K와 분해도), 라우팅 사용량, 하드 블록 활용도.

### 지연

- ROM: \(t_{\text{decode}}+t_{\text{sense}}\) (거의 일정).
- PLD/CPLD: AND→OR 두 레벨 + 라우팅(결정적).
- FPGA: LUT 레벨 수 + 라우팅 지연(파이프라인으로 완화).

### 해저드

- ROM: 주소 글리치 시 출력 스파이크 → **출력 레지스터**.
- PLD/CPLD: 정적-1 해저드 → **합의항 추가**.
- FPGA: 재수렴 경로 글리치 → **동기 경계/균형 파이프라인**.

### 보안

- FPGA(SRAM) 비트스트림 **암호화/인증** 기본.
- 안티퓨즈/Flash형은 물리 복제 난이도↑.
- ROM/PLD/CPLD도 펌웨어 접근 제어·JTAG 보안 필수.

---

## 구현 예 — **같은 기능을 서로 다르게**

### 문제 정의

- 입력 \(A,B,C,D\) → 출력 2개 \(Y_1,Y_2\)
\[
Y_1 = AB + \overline{A}C,\qquad
Y_2 = (A \oplus B)C + D
\]

#### (a) ROM 방식 (동기)

```verilog
module rom_dual(
  input  logic        clk,
  input  logic  [3:0] in,
  output logic  [1:0] y
);
  logic [1:0] mem [0:15];
  // 테이블 채움(모든 조합 16×2비트)
  initial begin : init
    integer i;
    for (i=0;i<16;i++) begin
      bit A=i[3], B=i[2], C=i[1], D=i[0];
      bit y1 = (A&B) | ((~A)&C);
      bit y2 = ((A^B)&C) | D;
      mem[i] = {y1,y2};
    end
  end
  always_ff @(posedge clk) y <= mem[in];
endmodule
```

#### (b) PLD/CPLD 스타일(제품항 공유)

```verilog
module pterms_dual(input logic A,B,C,D, output logic Y1,Y2);
  logic p_ab   = A & B;
  logic p_notA_C = (~A) & C;
  logic p_axb_c  = (A ^ B) & C; // XOR는 CPLD 내부에서 2-3제품항으로 합성될 수 있음
  assign Y1 = p_ab | p_notA_C;
  assign Y2 = p_axb_c | D;
endmodule
```

#### (c) FPGA LUT 친화(파이프라인)

```verilog
module fpga_lutpipe(
  input  logic clk,
  input  logic A,B,C,D,
  output logic Y1,Y2
);
  // 1단: 부분식 (LUT 매핑)
  logic ab, notA_c, axb_c;
  always_ff @(posedge clk) begin
    ab     <= A & B;
    notA_c <= (~A) & C;
    axb_c  <= (A ^ B) & C; // 6-LUT 하나로 가능
  end
  // 2단: 출력 결합
  always_ff @(posedge clk) begin
    Y1 <= ab | notA_c;
    Y2 <= axb_c | D;
  end
endmodule
```
- FPGA는 **파이프라인 단계**로 Fmax를 크게 끌어올릴 수 있다.

---

## 설계 흐름 · 체크리스트

### 선택 로드맵

1) 진리표가 **밀집**이고 \(n\)이 작다 → **ROM**.
2) **희소** + 제품항/다중출력 공유 필요 → **PLD/CPLD**.
3) 대규모/고성능/가속/인터페이스 통합 → **FPGA**.

### 합성 단서

- ROM: `case`/배열 초기화/`$readmemh`로 **동기 ROM** 추론.
- PLD/CPLD: **SOP 최소화**(K-map, Quine–McCluskey, Espresso). **제품항 한도** 확인.
- FPGA: LUT 입력 수 고려해 **분해/균형 트리/카리 체인/DSP/BRAM** 사용. **레지스터 풍부**하게.

### 타이밍/전력/검증

- **STA**로 경로 확인, **타이밍 예산**을 앞당겨 잡는다.
- 클록·리셋·CE는 **전용 네트워크**에 매핑.
- 온칩 디버그(ILA/SignalTap)로 bring-up/필드 이슈 대응.

---

## 고급 주제

### 부분 재구성(PR, FPGA)

- 고정(Static) 영역과 PR 파티션 분리, **경계 신호** 레지스터링, **OOC 합성**과 타이밍 가이드 필요.
- 사용 예: 가속 커널 스왑, 멀티테넌트.

### 메모리 초기화/보호

- FPGA 구성 시 BRAM/URAM/분산RAM에 **초기화 값** 자동 로딩.
- 보안: 비트스트림 **암호화/인증** 활성화, 키 관리·디버그 포트 잠금.

### 해저드 엔지니어링

- PLD/CPLD: SOP 경로 겹침 부족 시 정적-1 해저드 → **합의항 추가**.
- FPGA: 재수렴 경로 글리치 → **레지스터 삽입**으로 동기화.

---

## 계산 예 — 어느 방식이 유리한가?

### ROM 비트 수

- 예) 5입력(주소) → 3출력: \(\;2^{5}\times 3 = 96\)비트.
  - **밀집 진리표**면 ROM이 간단·작다.

### PLD/CPLD 제품항

- 같은 기능이 \(k\)개의 제품항으로 표현되면 면적은 대략 **제품항 수 × 평균 리터럴**에 비례.
- **희소**하면 제품항이 크게 줄어 ROM보다 유리.

### FPGA LUT 추정

- 6-LUT 기준, 입력 10개 OR → 균형 트리로 \(\lceil\log_{6}(10)\rceil\) 단계가 아니라
  **팬인 분해**(예: 6→6→…)와 **카리 체인/XOR** 활용으로 단계/지연을 최적화.
- 파이프라인을 넣으면 단계 수는 늘지만 **Fmax↑**.

---

## 추가 예제 — ROM vs PLA 최소화 비교

### 사양

\[
F(A,B,C,D)=\sum m(0,2,5,6,7,8,10,13,15)
\]
- **ROM**: \(2^{4}\times 1=16\)비트 저장.
- **PLA**: Quine–McCluskey/Espresso로 최소화 → (예) 3제품항 6리터럴이면 면적 이득.
- **CPLD**: 해당 제품항을 **매크로셀 공유**로 배치.
- **FPGA**: 6-LUT 1~2개로 매핑 가능(식에 따라 다름).

---

## 연습문제

1) 5입력 3출력 함수의 **ROM 비트 수**와, Espresso로 최소화 시 예상 **제품항 수**에 따른 **PLD 면적 추정**을 비교하라(밀집/희소 두 케이스).
2) \(F_1=AB+AC+AD,\;F_2=\overline{A}C+BD\). CPLD(매크로셀당 제품항 5개)에서 **제품항 공유 전략**으로 매핑하라.
3) FPGA(6-LUT)에서 12입력 OR을 **균형 트리**와 **카리 체인 기반**으로 각각 설계하고, 예상 지연을 정성적으로 비교하라.
4) ROM 기반 7세그 디코더와 PLA 기반 디코더의 **해저드 특성**과 **완화 기법**을 서술하라(레지스터/합의항).
5) SRAM FPGA에서 **부분 재구성(PR)** 을 적용할 때, PR 파티션 경계 신호에 필요한 **레지스터링**과 **타이밍 제약**을 정리하라.

---

## 마무리 요약

- **ROM**: 진리표 그 자체. 다중출력/밀집 논리에 단순·강력.
- **PLD**: 제품항 기반 SOP. 희소 논리에 효율, 합의항으로 해저드 제어.
- **CPLD**: 비휘발성·결정적 타이밍·보드 glue에 최적.
- **FPGA**: 6-LUT + 하드블록 + P&R로 대규모/고성능 구현(보안·PR 고려).
- 설계 선택은 **밀집/희소**, **성능/전력/보안**, **현장 재구성 필요성**을 종합해 결정한다.
