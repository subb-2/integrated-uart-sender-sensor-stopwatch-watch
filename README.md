# 통합 디지털 시스템 설계 — 2조

> Basys3(Xilinx FPGA) 기반 Stopwatch / Clock 시스템에  
> SR04 초음파 센서, DHT11 온습도 센서, UART PC 인터페이스를 통합한 디지털 시스템

---

## 📌 프로젝트 개요

기존 Stopwatch / Clock 시스템을 확장하여 SR04, DHT11 센서를 통합하고,  
UART 통신을 통해 외부 PC와 데이터를 송수신할 수 있도록 설계한 통합 디지털 시스템입니다.

- SR04 초음파 센서를 이용한 거리 측정 및 7-Segment 표시
- DHT11 센서를 이용한 온도 / 습도 측정 및 7-Segment 표시
- UART를 이용하여 현재 시간 및 온도 / 습도를 PC로 전송
- PC에서 UART 명령어를 통해 모드 전환 및 버튼 제어 지원

---

## 🎯 동작 모드

| SW[1] & SW[4] | Mode | 7-Segment 표시 | LED[1:0] | PC 명령 |
|:---:|------|--------------|:---:|------|
| [0] | Stopwatch | HH:MM / SS:CC  (최대 99:59 / 59:99) | [0] | R / N / C |
| [1] | Clock | HH:MM / SS:CC  (24시간제) | [1] | R / N / C |
| [1] + SW[3] | Time-set | HH:MM / SS:CC  (선택 자리 LED 점등) | 선택 자리 | R / N / C |
| [2] | SR04 | `Sr04` / `xxx.x` (0.1 cm 단위) | [2] | T |
| [3] | DHT11 | `hxx.x` / `txx.x` (습도 / 온도) | [3] | H |
| — | PC Control | — | LED[5] = 1 | M / 0~4 |

---

## 🕹️ 입력 매핑

### 버튼 (Button)

| Button | Stopwatch / Clock Mode | SR04 / DHT11 Mode |
|--------|------------------------|-------------------|
| BTN_U | Run / Stop &nbsp;&nbsp;/&nbsp;&nbsp; Up | — |
| BTN_C | NA &nbsp;&nbsp;/&nbsp;&nbsp; Time-set select | — |
| BTN_D | Clear &nbsp;&nbsp;/&nbsp;&nbsp; Down | — |
| BTN_L | — | 거리 측정 시작 (SR04) |
| BTN_R | — | 온습도 측정 시작 (DHT11) |

### 스위치 (Switch)

| Switch | 기능 |
|--------|------|
| SW[1] & SW[4] | 동작 모드 선택 ([0] Stopwatch / [1] Clock / [2] SR04 / [3] DHT11) |
| SW[0] | Stopwatch 카운트 방향 (Up / Down) |
| SW[2] | 디스플레이 포맷 전환 |
| SW[3] | Time-set 모드 진입 (Clock 모드에서 LED 점등) |
| SW[15] | Reset |

### PC 명령어 (UART ASCII, 9600 bps)

| 명령 | 동작 |
|------|------|
| `M` | PC 제어 모드 토글 (활성화 시 물리 스위치 무효화) |
| `R` | BTN_U (Run / Stop / Up) |
| `N` | BTN_C (Time-set select) |
| `C` | BTN_D (Clear / Down) |
| `T` | BTN_L — SR04 거리 측정 시작 |
| `H` | BTN_R — DHT11 온습도 측정 시작 |
| `Q` | 현재 시간 + DHT11 데이터 출력 요청 |
| `0` ~ `4` | SW[0] ~ SW[4] 토글 |

**Q 명령 출력 포맷**
```
HH:MM:SS:CC
T xx.xC
H xx.x%
```

---

## 🏗️ 시스템 구조

```
                    ┌────────────────────────────────────────────────────┐
                    │               top_stopwatch_watch                  │
  clk ─────────────►│                                                    │
  reset ───────────►│  ┌────────────┐  ┌───────────────┐                │
  mode_sw ─────────►│  │  Control   │  │  Clock/SW     │                │
  btn_8/5/2 ───────►│  │   Unit     │  │  Datapath     │                │
  echo ─────────────►│  └─────┬──────┘  └───────┬───────┘                │
  uart_rx ──────────►│        │                  │                        │
                    │  ┌──────▼──────┐  ┌───────▼───────┐               │
                    │  │  SR04 CTRL  │  │  DHT11 CTRL   │               │
                    │  └──────┬──────┘  └───────┬───────┘               │
                    │         │                  │                        │
                    │  ┌──────▼──────────────────▼──────┐               │
                    │  │          FND Controller          │               │
                    │  └──────────────────┬──────────────┘               │
                    │                     │                               │
                    │  ┌──────────────────▼──────────────┐               │
                    │  │            UART Top              │               │
                    │  │   (RX_FIFO / TX_FIFO / ASCII     │               │
                    │  │         SENDER)                  │               │
                    │  └──────────────────────────────────┘               │
                    └────────────────────────────────────────────────────┘
```

---

## 📁 파일 구성

```
sources_1/
├── top_stopwatch_clock.v   # 최상위 모듈 — 전체 서브모듈 연결
├── control_unit.v          # 모드/버튼/스위치 제어 및 PC 명령 처리
├── fnd_controller.v        # 7-Segment 표시 제어 (BCD 변환, 소수점 포함)
├── uart_top.v              # UART TX/RX + FIFO + ASCII SENDER 통합
├── fifo.v                  # RX / TX 버퍼용 FIFO (DEPTH=4, BIT_WIDTH=8)
├── sr04_crtrl_top.v        # SR04 초음파 거리 측정 FSM
├── dht11_ctrl.v            # DHT11 온습도 센서 FSM (★ 본인 담당 모듈)
└── btn_debounce.v          # 버튼 디바운스 처리

constrs_1/
└── (Basys3 pin constraint .xdc file)
```

---

## 🔧 모듈 설계 세부 사항

### Control Unit

모드 스위치, 물리 버튼, UART PC 명령을 통합 처리하여 각 서브모듈에 제어 신호를 분배합니다.

| 신호 | 설명 |
|------|------|
| `mode_sw_com` | PC 제어 모드 시 `pc_mode_sw`를 우선 선택, 아니면 물리 `mode_sw` 사용 |
| `w_sr04_btn` | SR04 모드 활성화 시에만 버튼 신호 전달 |
| `w_dht11_btn` | DHT11 모드 활성화 시에만 버튼 신호 전달 |
| `w_mode_swclk` | Stopwatch / Clock 모드 판별 (`2'b00` 또는 `2'b01`) |

### FND Controller

입력 비트 폭을 26-bit로 통일하고, SR04 / DHT11 전용 MUX를 추가하였습니다.

| 항목 | 설명 |
|------|------|
| BCD 변환 | 센서 데이터를 각 자릿수로 분리 (`% 10`, `/ 10`, ...) |
| 소수점 표시 | `dot` 신호 개별 제어로 소수점 위치 지정 |
| SR04 포맷 | `Sr04` 문자 + `xxx.x cm` |
| DHT11 포맷 | `t xx.x` (온도) / `h xx.x` (습도) |

### UART / FIFO / ASCII SENDER

| 모듈 | 설명 |
|------|------|
| UART TX | `TX_start` 시 Start bit → 8bit 데이터 → Stop bit 순서로 LSB 방식 전송 |
| UART RX | 하강 에지 감지 후 비트 중앙(CNT=7) 샘플링, 9600 bps (16배속 tick) |
| FIFO | write pointer / read pointer 방식, `full=1`이면 `we=0`으로 오버플로 방지 |
| ASCII SENDER | FND Controller에서 시간 / 온습도 데이터를 받아 `HH:MM:SS`, `T xx.xC`, `H xx.x%` 포맷으로 전송 |

**PC 제어 명령어 목록**

| 명령 | 동작 |
|------|------|
| `M` | PC 제어 모드 전환 (물리 스위치 무효화) |
| `0`~`4` | 모드 스위치 SW 제어 |
| `R` / `N` / `C` | run/stop / next / clear 버튼 동작 |
| `T` | SR04 측정 시작 |
| `H` | DHT11 측정 시작 |
| `Q` | 현재 FPGA 데이터 PC로 출력 |

---

### DHT11 Controller (`dht11_ctrl.v`) ★

Single-Wire Half-Duplex 방식으로 DHT11 센서와 통신하여 40-bit 온습도 데이터를 수신합니다.

**전송 데이터 구조 (40-bit, MSB)**

```
[39:32] 습도 정수 (8-bit) | [31:24] 습도 소수 (8-bit)
[23:16] 온도 정수 (8-bit) | [15: 8] 온도 소수 (8-bit)
[ 7: 0] Checksum  (8-bit)
```

**FSM 상태 구성**

| 상태 | 동작 |
|------|------|
| `IDLE` | 대기. `start` 신호 또는 60초 자동 카운트로 `START` 전환 |
| `START` | `dhtio = 0` 출력, 19ms (tick 1900회) 유지 후 `WAIT` 전환 |
| `WAIT` | `dhtio`를 Hi-Z로 해제 (`io_sel = 0`), 센서 응답 대기 |
| `SYNC_L` | 센서의 80us Low 응답 확인 (`dhtio_rise` 감지) |
| `SYNC_H` | 센서의 80us High 응답 확인 (`dhtio_fall` 감지) |
| `DATA_SYNC` | 각 비트 시작 Low 구간 대기 (`dhtio_rise` 감지 시 카운터 초기화) |
| `DATA_C` | High 구간 10us tick 카운팅: `tick >= 5` → `'1'`, 미만 → `'0'` (40회 반복) |
| `STOP` | 전송 완료, checksum 검증 후 `IDLE` 복귀. 성공 시 `dht11_valid = 1` |

**Tri-state 버스 제어**

```verilog
// io_sel_reg = 1 : FPGA가 dhtio를 구동 (START 구간)
// io_sel_reg = 0 : Hi-Z 해제, 센서가 dhtio를 구동 (수신 구간)
assign dhtio = (io_sel_reg) ? dhtio_reg : 1'bz;
```

**Synchronizer & Edge Detector**

외부 신호(dhtio)의 메타스타빌리티 방지를 위해 2-FF 동기화를 적용하고, 상승/하강 에지를 검출합니다.

```verilog
// 2-FF Synchronizer
dhtio_meta_reg   <= dhtio;
dhtio_sync_reg   <= dhtio_meta_reg;
dhtio_sync_2_reg <= dhtio_sync_reg;

// Edge detection
wire dhtio_rise = dhtio_sync_reg & ~dhtio_sync_2_reg;
wire dhtio_fall = ~dhtio_sync_reg & dhtio_sync_2_reg;
```

> **Trouble Shooting**: Reset 직후 의도하지 않은 `rise edge` 발생 문제  
> 원인: Synchronizer 초기값을 `0`으로 설정 → `dhtio` 기본 상태(High)와 불일치  
> 해결: `dhtio_meta_reg`, `dhtio_sync_reg`, `dhtio_sync_2_reg` 초기값을 모두 `1`로 수정

**Watchdog 타이머**

통신 오류 또는 연결 단선으로 FSM이 중간 상태에서 멈추는 경우를 대비한 자동 복구 로직입니다.

```verilog
// IDLE이 아닌 상태에서 ~1초(99,999 tick × 10us) 이상 유지되면 강제 IDLE 복귀
if (c_state != IDLE && timeout_rst_reg >= 17'd99_999) begin
    n_state = IDLE;
    ...
end
```

**자동 측정 (Auto Start)**

Reset 후 0.1초 뒤 첫 측정을 시작하고, 이후 매 60초마다 자동으로 반복 측정합니다.

```verilog
// 60s 주기 자동 시작
if (auto_cnt_reg == 23'd6_000_000 - 1) begin
    auto_cnt_next = 0;
    n_state = START;
end
```

---

### SR04 Controller

| 상태 | 동작 |
|------|------|
| `IDLE` | `start` 신호 대기 |
| `TRIG` | 10us TTL 트리거 펄스 출력 (`trig_cnt` 10회) |
| `WAIT` | echo 신호 대기 (TIMEOUT_WAIT = 25ms) |
| `CAL_1` | echo High 구간 1us tick 카운팅, 하강 에지 시 `CAL_2`로 전환 |
| `CAL_2` | `distance_x10`을 58씩 감산하여 cm 단위 변환 (나눗셈 대체, timing slack 방지) |

> Negative Slack 발생 원인: 나눗셈 연산자 `/` 사용  
> 해결: 반복 감산 방식(`distance_x10 -= 58; distance_div++`)으로 대체 → WNS 4.029ns 확보

---

## 🖥️ 개발 환경

| 항목 | 내용 |
|------|------|
| HDL | Verilog |
| EDA Tool | Xilinx Vivado |
| 타겟 보드 | Digilent Basys3 (Artix-7) |
| 시뮬레이터 | Vivado Simulator (XSim) |
| UART 설정 | 9600 bps, 8-bit data, Stop 1-bit, No parity |

---

## ✅ 시뮬레이션 검증 항목

- **FIFO**: push 1회 → `empty=0`, push 4회 → `full=1`, pop 1회 → `full=0`, pop 4회 → `empty=1`
- **UART loopback**: `8'h30` 송신 후 `8'b00110000` 수신 확인
- **ASCII SENDER**: `w_sender_tx_data`에서 `12:00:00` 시간 포맷 생성 확인
- **SR04**: `echo` 상승/하강 에지 검출, `dist=100` (echo 580us → 10cm) 출력 확인, TIMEOUT 25ms 동작 확인
- **DHT11**: 0/1 비트 판별 (`50us` 기준), checksum 성공 시 `valid=1`, 실패 시 `valid=0`, Watchdog 1초 복구 동작 확인
- **통합 시뮬레이션**: PC 명령 `M` 으로 모드 전환, DHT11 자동 측정(T=25, H=50), SR04 echo 타임아웃/비정상 응답 처리, UART 31-byte 전송 확인

---

## ⚙️ 타이밍 결과 (Vivado Implementation)

| 항목 | 값 |
|------|----|
| Worst Negative Slack (WNS) | 4.029 ns |
| Total Negative Slack (TNS) | 0 ns |
| Failing Endpoints | 0 |

> SR04 `/58` 나눗셈 연산자를 반복 감산 방식으로 교체 후 모든 타이밍 제약 조건 만족

---

## 🐛 Trouble Shooting

### 1. DHT11 Reset 직후 오동작 (의도하지 않은 상승 에지)

**문제**: Reset 직후 `dhtio_edge_rise`가 1회 발생하여 FSM이 잘못된 상태로 진입.

**원인**: Synchronizer 내 `dhtio_q1`, `dhtio_q2` 초기값이 `0`으로 설정되어 있어, DHT11의 기본 버스 상태(High=1)와 불일치 → 첫 클록에서 `(~0) & 1 = 1`로 rise edge 오감지.

**해결**: 초기값을 `1'b1`로 수정하여 기본 버스 상태와 동기화.

### 2. DHT11 통신 오류 시 FSM 정지

**문제**: 센서 연결 불량 또는 통신 오류 발생 시 FSM이 중간 상태에서 멈춰 이후 측정 불가.

**원인**: 상태 전환 조건이 외부 신호(`echo`, `dhtio_rise/fall`)에만 의존하므로 신호가 오지 않으면 무한 대기.

**해결**: Watchdog 타이머 추가. IDLE이 아닌 상태에서 99,999 tick(≈1초) 초과 시 강제로 IDLE 복귀. 오류 측정 시 이전 유효 값 유지.

### 3. SR04 통합 후 Negative Slack 발생

**문제**: SR04 거리 계산에서 `distance <= (echo_cnt * 10) / 58` 연산자 사용 시 WNS −2.194ns 발생, Failing Endpoints 13개.

**원인**: `/` 연산자가 조합 논리로 합성될 때 깊은 경로를 생성하여 타이밍 위반.

**해결**: `distance_x10 -= 58; distance_div++` 반복 감산 방식으로 교체 → WNS 4.029ns, Failing Endpoints 0.

### 4. FND Controller 7-Segment 표시 오류

**문제**: 기존 Stopwatch의 `time_sel` 비트 폭(6-bit)으로는 센서 데이터 표시 범위 부족.

**원인**: 기존 MUX 구조가 6-bit 입력 기준으로 설계되어 상위 자릿수 표시 불가.

**해결**: 각 자릿수 전용 MUX를 추가하고, 입력 비트 폭을 26-bit로 통일. 여분 자리에 `S`, `r`, `t`, `h` 문자 매핑 추가.
