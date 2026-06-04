# Verilog HDL을 활용한 Stopwatch & Watch 설계 및 구현

**작성자:** 온디바이스 AI 시스템 반도체설계 1기 전정묵

---

## Ⅰ. 프로젝트 개요

### 1. 설계 목적
| 구분 | 내용 |
|---|---|
| **핵심 목표** | FPGA 환경에서 100MHz 시스템 클럭을 분주하여 실시간으로 동작하는 Stopwatch와 Watch 기능 구현 |
| **학습 목적** | Verilog HDL을 사용하여 사용자에게 시각적으로 전달하는 제어 로직을 설계함으로써 디지털 논리 회로의 설계 및 검증 프로세스 이해 |

### 2. 개발 환경
| 목록 | 비고 |
|---|---|
| **사용 언어** | Verilog HDL, RTL 환경 검증 |
| **대상 보드** | FPGA 개발 보드 (Basys3) |
| **출력 표시** | 4-Digit 7-Segment Display |

### 3. 주요 기능 및 사양
| 사양 항목 | 상세 설명 |
|---|---|
| **시간 측정** | 100MHz 클럭 분주를 통해 0.01초(10ms) 단위의 정밀한 시간 측정 |
| **시간 표시 범위** | 밀리초(msec), 초(sec), 분(min), 시(hour) 단위 계산 |
| **디스플레이 모드** | 선택에 따라 `[Hour:Min]` 또는 `[Sec:Msec]` 단위를 7-Segment에 출력 |
| **카운트 방향** | 스위치 설정에 따라 순방향(Up-count) 및 역방향(Down-count) 측정 지원 |

### 4. 사용자 인터페이스 (UI)
| INPUT | 기능 설명 |
|---|---|
| **Run/Stop (Btn)** | 버튼 입력을 통해 타이머의 시작과 일시 정지를 제어 |
| **Clear (Btn)** | 카운트 동작 중 또는 정지 시 시간을 Reset (초기화) |
| **Mode Select (SW)** | 스위치를 이용한 카운트 방향 및 디스플레이 모드 전환 |

---

## Ⅱ. 전체 시스템 구조

### 1. 시스템 계층 구조 (Hierarchy)
본 시스템은 모듈 설계 원칙에 따라 최상위 모듈인 `TOP_counter_10000`을 중심으로 **제어부**, **데이터 처리부**, **디스플레이부** 세 가지 계층으로 구성됩니다. 각 모듈은 독립적인 기능을 수행하며 신호 전달을 통해 전체 스톱워치/시계 기능을 완성합니다.

### 2. 전체 시스템 구성 및 블록다이어그램
![전체 시스템 구성](images/스크린샷%202026-06-04%20180551.png)

![전체 블록다이어그램](images/스크린샷%202026-06-04%20180543.jpg)

| 구분 | Instance 명 | 주요 역할 |
|---|---|---|
| **제어 로직** | `btn_debounce` | 스위치 및 버튼 노이즈(채터링) 제거 |
| | `control_unit` | 시스템 상태 관리 및 시계/스톱워치 모드 제어 |
| **데이터 처리** | `stopwatch_datapath` | Stopwatch 카운팅 및 Run/Stop 연산 처리 |
| | `watch_datapath` | Watch 카운팅 및 Up/Down 시간 수정 기능 |
| **디스플레이** | `fnd_controller` | 처리된 시간 데이터를 7-Segment 규격에 맞게 변환하여 표시 |

### 3. 세부 블록다이어그램

#### 가. Watch 블록다이어그램
![Watch 블록다이어그램](images/스크린샷%202026-06-04%20180559.png)

#### 나. Stopwatch 블록다이어그램
![Stopwatch 블록다이어그램](images/스크린샷%202026-06-04%20180605.png)

#### 다. FND Controller 블록다이어그램
![FND Controller 블록다이어그램](images/스크린샷%202026-06-04%20180610.jpg)

#### 라. FND Digit & Data 블록다이어그램
![FND Digit & Data 블록다이어그램](images/스크린샷%202026-06-04%20180631.png)

### 4. TOP MODULE Input / Output 포트 정의
![TOP MODULE 포트 정의](images/스크린샷%202026-06-04%20180638.png)

| 구분 | 포트명 | 기능 설명 |
|---|---|---|
| **Input** | `clk` | 100MHz 시스템 메인 클럭 |
| | `reset` | 시스템 전체 비동기 리셋 |
| | `sw[0] / i_mode` | 카운트 모드 선택 (0: Up, 1: Down) |
| | `sw[1] / i_sel` | 기능 선택 (0: 스톱워치, 1: 시계) |
| | `sw[2]` | Display Select (0: `sec:msec`, 1: `hour:min`) |
| | `btn_r` | Run/Stop 제어 버튼 입력 (Watch: min/msec 증가) |
| | `btn_l` | Clear 제어 버튼 입력 (Watch: hour/sec 증가) |
| | `btn_down_u` | 시계(수정모드): 왼쪽 자릿수 감소 (Down) |
| | `btn_down_d` | 시계(수정모드): 오른쪽 자릿수 감소 (Down) |
| **Output**| `fnd_digit` | 4자리 7-Segment 중 활성화할 자릿수 선택 신호 (4-bit) |
| | `fnd_data` | 7-Segment에 표시할 데이터 출력 (A~G, DP) (8-bit) |

---

## Ⅲ. 제어 및 입력부

### 1. 버튼 채터링 제거 (btn_debounce)
기계식 스위치 및 버튼 입력 시 발생하는 접점 떨림(채터링)으로 인한 오작동을 방지하기 위해 입력을 샘플링합니다.

![btn_debounce 설계](images/스크린샷%202026-06-04%20180647.jpg)

| 설계 상세 항목 | 내용 |
|---|---|
| **샘플링 클럭 생성** | 100MHz 클럭을 분주하여 1ms(1KHz) 주기의 저속 샘플링 클럭을 생성 |
| **Shift Register 활용** | 1ms마다 버튼 상태를 체크하여 `q_reg`에 8개의 샘플 값을 순차적으로 저장 |
| **Edge 검출** | 이전 상태값과 현재 상태값을 비교(AND, NOT 연산)하여 버튼이 1번 눌렸을 때 정확히 1회의 신호(`o_btn`)만 출력 |

**btn_debounce Simulation 분석:**
![btn_debounce Simulation](images/스크린샷%202026-06-04%20180656.png)
입력(`i_btn`)이 지속되더라도 Shift Register가 모두 1로 채워지는 순간 Edge 검출 로직을 통해 단 한 번의 출력 펄스만 생성되는 것을 검증하였습니다.

### 2. 제어 유닛 (Control Unit)
사용자의 입력을 받아 시스템의 현재 상태를 결정하고 제어 신호를 생성하는 FSM(Finite State Machine) 모듈입니다.

| 상태 / 제어 | 동작 설명 |
|---|---|
| **FSM State** | `STOP`(정지), `RUN`(동작), `CLEAR`(초기화) 상태를 가짐 |
| **제어 신호 출력** | `i_run_stop`, `i_clear` 버튼 입력에 따라 다음 상태(`next_st`)로 전이하고, `o_run_stop`, `o_clear` 신호를 Datapath로 전달 |
| **수정 모드 제어** | Watch 모드 동작 시 `o_watch_change`, `o_watch_up`, `o_watch_down` 등의 시간 보정 제어 신호 활성화 |

---

## Ⅳ. 데이터 처리부 (Datapath)
시스템의 시간 정보를 생성하는 하위 모듈들을 통합 관리하며, 100MHz 클럭을 기반으로 10ms 단위 틱을 생성 후 계층적 카운팅(msec -> sec -> min -> hour)을 수행합니다.

### 1. Tick Generator (tick_gen_100hz)
![tick gen 100hz](images/스크린샷%202026-06-04%20180713.jpg)

| 설계 상세 항목 | 내용 |
|---|---|
| **주파수 분주** | 100MHz 메인 클럭을 10ms(100Hz) 주기로 변환하여 0.01초 단위의 펄스(1 tick) 발생 |
| **동작 조건** | `i_run_stop`이 1일 때만 내부 카운터가 증가하며, `999,999` 도달 시 1 tick 출력 |

### 2. Tick Counter 모듈
틱 신호를 받아 설정된 최댓값(TIMES)에 도달하면 다음 상위 단위(초->분->시)로 캐리 틱(`o_tick`)을 전달하는 계층형 카운터입니다.

**Tick Counter Simulation 분석:**
![tick counter simulation](images/스크린샷%202026-06-04%20180706.png)

| 모드 상태 | 입력/조건 | 결과 및 출력 |
|---|---|---|
| **Up-count (mode=0)** | `counter_reg` != `TIMES - 1` | `counter_next` = `counter_reg` + 1, `o_tick` = 0 |
| | `counter_reg` == `TIMES - 1` (최댓값 도달) | `counter_next` = 0, `o_tick` = 1 (상위 캐리 발생) |
| **Down-count (mode=1)** | `counter_reg` != 0 | `counter_next` = `counter_reg` - 1, `o_tick` = 0 |
| | `counter_reg` == 0 (최솟값 도달) | `counter_next` = `TIMES - 1`, `o_tick` = 1 (상위 빌림 발생) |

---

## Ⅴ. 디스플레이 제어부 (FND Display)
데이터 처리부에서 연산된 시간 데이터를 사람이 식별할 수 있도록 4자리 7-Segment(FND) 디스플레이 규격으로 디코딩 및 시분할 출력합니다.

### 1. 세부 모듈 로직 설계
![Display Control Modules](images/스크린샷%202026-06-04%20180722.jpg)

| 모듈명 | 주요 기능 및 설명 |
|---|---|
| `clk_div` | 디스플레이 시분할을 위해 100MHz 클럭을 사람이 육안으로 인식 가능한 1KHz 속도로 분주 |
| `dot_onoff` | 시각적 효과를 위해 msec 카운트에 따라 Dot LED를 점멸 (0~49구간 점등, 50~99구간 소등) |
| `mux_2x1` | 스위치(`sel_display`) 설정에 따라 FND에 표시할 데이터를 `[Hour:Min]` 또는 `[Sec:Msec]` 중 선택 |
| `mux_8x1` | 4자리의 각 숫자와 Dot 정보를 순차적으로 선택(스위칭)하여 출력으로 내보냄 |
| `counter_8` | 1KHz 틱에 맞춰 4자리 FND 자리를 순차적으로 결정하는 3-bit 선택 신호(`digit_sel`) 생성 |
| `decoder_2x4` | `digit_sel` 값을 감시하여 어떤 위치의 FND 자릿수를 활성화할지 제어(`fnd_digit` 출력) |

### 2. BCD 및 Digit Splitter
![BCD Decoder](images/스크린샷%202026-06-04%20180730.jpg)

| 모듈명 | 상세 기능 |
|---|---|
| `digit_splitter` | FND는 한 번에 1자리 숫자(0~9)만 표시할 수 있으므로, 연산된 2자리 시간 데이터(예: 31초)를 10의 자리(3)와 1의 자리(1)로 분리 (나눗셈 및 나머지 연산 활용) |
| `bcd` | 분리된 4-bit 숫자 데이터를 입력받아 Case문을 통해 7-Segment LED를 점등하기 위한 8-bit 패턴(`fnd_data`)으로 디코딩 변환 |

---

## Ⅵ. 결론 및 고찰

### 1. 주요 디버깅 (Debugging) 사례
![Debugging](images/스크린샷%202026-06-04%20180754.png)

| 이슈 사항 | 원인 및 해결 방법 |
|---|---|
| **현상** | 합성 후 특정 FND 데이터가 정상적으로 출력되지 않고 회로가 끊어지는 현상 발생 |
| **원인 파악** | RTL Schematic 툴을 통해 시각적 검증을 수행한 결과, `mux_8x1` 모듈의 선택 신호(`sel`)의 비트 폭(Bit-width)이 2-bit로 잘못 설계되어 8개의 입력을 모두 커버하지 못함 확인 |
| **해결** | 선택 신호의 비트 폭을 3-bit(`input [2:0] sel`)로 확장 선언하여 Bit Width 매칭 문제 해결 및 정상 동작 확인 |

### 2. 고찰
계층적 모듈화(Hierarchy)를 통한 Top-Down 방식의 설계는 방대한 디지털 회로를 관리 가능하게 만들어 주었으며, 특히 RTL Schematic 시각화 툴과 Simulation Waveform 검증이 디버깅에 필수적임을 체감할 수 있었습니다. 시스템 클럭 분주(Clock Division), 채터링 제거(Debounce), FSM 기반의 상태 제어 등 디지털 논리 회로 설계의 정석적인 패턴들을 성공적으로 하드웨어 상(Basys3)에 구현한 의미 있는 프로젝트였습니다.
