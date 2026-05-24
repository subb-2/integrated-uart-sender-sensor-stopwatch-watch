# 🔹 통합 디지털 시스템 설계 — 2조

> Basys3(Xilinx FPGA) 기반 Stopwatch / Clock 시스템에  
> SR04 초음파 센서, DHT11 온습도 센서, UART PC 인터페이스를 통합한 디지털 시스템

---

## 📌 프로젝트 개요

기존 Stopwatch / Clock 시스템을 확장하여 SR04, DHT11 센서를 통합하고, UART 통신을 통해 외부 PC와 데이터를 송수신할 수 있도록 설계한 통합 디지털 시스템이다.

- SR04 초음파 센서를 이용한 거리 측정 및 7-Segment 표시
- DHT11 센서를 이용한 온도 / 습도 측정 및 7-Segment 표시
- UART를 이용하여 현재 시간 및 온도 / 습도를 PC로 전송
- PC에서 UART 명령어를 통해 모드 전환 및 버튼 제어 지원

---

## 👥 역할 분담

| 이름 | 담당 |
|------|------|
| 김민수 | UART Controller, FIFO, ASCII SENDER |
| 김성훈 | SR04 Controller |
| 김수빈 | DHT11 Controller |
| 하지훈 | 시스템 통합 (Control Logic, FND, LED Indicator) |

---

## 🎯 동작 모드

| SW[1] & SW[4] | Mode | 7-Segment 표시 | LED[1:0] | PC 명령 |
|:---:|------|------|:---:|------|
| [0] | Stopwatch | HH:MM / SS:CC | [0] | R / N / C |
| [1] | Clock (+ Time-set) | HH:MM / SS:CC | [1] | R / N / C |
| [2] | SR04 | `Sr04` / `xxx.x` (0.1 cm 단위) | [2] | T |
| [3] | DHT11 | `hxx.x` / `txx.x` | [3] | H |
| — | PC Control | — | LED[5] = 1 | M / 0~4 |

---

## 🕹️ 입력 매핑

### 버튼 (Button)

| Button | Stopwatch Mode | Clock Mode | SR04 / DHT11 Mode |
|--------|---------------|------------|-------------------|
| BTN_U | Run / Stop | Up | — |
| BTN_C | — | Time-set Select | — |
| BTN_D | Clear | Down | — |
| BTN_L | — | — | 거리 측정 시작 (SR04) |
| BTN_R | — | — | 온습도 측정 시작 (DHT11) |

### 스위치 (Switch)

| Switch | 기능 |
|--------|------|
| SW[1] & SW[4] | 동작 모드 선택 |
| SW[0] | Stopwatch 카운트 방향 (Up / Down) |
| SW[2] | 디스플레이 포맷 전환 |
| SW[3] | Time-set 모드 진입 (Clock 모드 전용) |
| SW[15] | Reset |

> PC 제어 모드 활성화 시 UART 입력 우선, 물리 입력 무효화

### PC 명령어 (UART ASCII, 9600 bps)

| 명령 | 대응 입력 | 동작 |
|------|---------|------|
| `M` | — | PC 제어 모드 토글 |
| `R` | BTN_U | Run / Stop / Up |
| `N` | BTN_C | Time-set Select |
| `C` | BTN_D | Clear / Down |
| `T` | BTN_L | SR04 거리 측정 시작 |
| `H` | BTN_R | DHT11 온습도 측정 시작 |
| `Q` | — | 현재 시간 + DHT11 데이터 출력 |
| `0`~`4` | SW[0]~SW[4] | 해당 스위치 토글 |

`Q` 명령 출력 포맷: `HH:MM:SS:CC` / `T xx.xC` / `H xx.x%`

---

## 🏗️ 시스템 구조

```text
                    ┌────────────────────────────────────────────────────┐
                    │               top_stopwatch_watch                  │
  clk ─────────────►│                                                    │
  reset ───────────►│  ┌────────────┐  ┌───────────────┐                │
  mode_sw ─────────►│  │  Control   │  │  Clock / SW   │                │
  btn ─────────────►│  │   Unit     │  │  Datapath     │                │
  echo ─────────────►│  └─────┬──────┘  └───────┬───────┘               │
  uart_rx ──────────►│        │                  │                       │
                    │  ┌──────▼──────┐  ┌───────▼───────┐               │
                    │  │  SR04 CTRL  │  │  DHT11 CTRL   │               │
                    │  └──────┬──────┘  └───────┬───────┘               │
                    │  ┌──────▼──────────────────▼──────┐               │
                    │  │          FND Controller          │               │
                    │  └──────────────────┬──────────────┘               │
                    │  ┌──────────────────▼──────────────┐               │
                    │  │     UART Top (RX/TX FIFO +       │               │
                    │  │          ASCII SENDER)           │               │
                    │  └──────────────────────────────────┘               │
                    └────────────────────────────────────────────────────┘
```

---

## 📁 파일 구성

```text
sources_1/
├── top_stopwatch_clock.v   # 최상위 모듈 — 전체 서브모듈 연결
├── control_unit.v          # 모드/버튼/스위치 제어 및 PC 명령 처리
├── fnd_controller.v        # 7-Segment 표시 제어 (BCD 변환, 소수점 포함)
├── uart_top.v              # UART TX/RX + FIFO + ASCII SENDER 통합
├── fifo.v                  # RX / TX 버퍼용 FIFO
├── sr04_ctrl_top.v         # SR04 초음파 거리 측정 FSM
├── dht11_ctrl.v            # DHT11 온습도 센서 FSM
└── btn_debounce.v          # 버튼 디바운스 처리

constrs_1/
└── (Basys3 pin constraint .xdc)
```

---

## 🔧 모듈 설계 세부 사항

### Control Unit

모드 스위치, 물리 버튼, UART PC 명령을 통합 처리하여 각 서브모듈에 제어 신호를 분배한다.

- PC 제어 모드 활성화 시 `pc_mode_sw`를 우선 선택하여 물리 스위치를 무효화
- 각 모드(Stopwatch / Clock / SR04 / DHT11)에 해당하는 버튼 신호만 해당 모듈로 전달
- 모드 외 버튼 입력은 차단하여 오동작 방지

### FND Controller

입력 비트 폭을 26-bit로 통일하고, SR04 / DHT11 전용 MUX를 추가하였다.

- 센서 데이터를 자릿수별로 분리하여 각 세그먼트에 직접 매핑
- `dot` 신호 개별 제어로 소수점 위치 지정
- 여분 자릿수에 `S`, `r`, `t`, `h` 문자 매핑 추가

| 모드 | 표시 형식 |
|------|---------|
| Stopwatch / Clock | HH:MM / SS:CC |
| SR04 | `Sr04` / `xxx.x cm` |
| DHT11 | `t xx.x` / `h xx.x` |

### UART / FIFO / ASCII SENDER

- **UART TX**: Start bit → 8-bit 데이터 → Stop bit 순서로 LSB 방식 전송
- **UART RX**: 하강 에지 감지 후 비트 중앙(CNT=7) 샘플링 (9600 bps, 16배속 tick)
- **FIFO**: write pointer / read pointer 방식, Full 시 write 차단으로 오버플로 방지
- **ASCII SENDER**: FND Controller로부터 시간 / 온습도 데이터를 받아 `HH:MM:SS`, `T xx.xC`, `H xx.x%` 포맷으로 순차 전송

---

### DHT11 Controller ★

Single-Wire Half-Duplex 방식으로 DHT11 센서와 통신하여 40-bit 온습도 데이터를 수신한다.

**전송 데이터 구조 (40-bit, MSB)**

| 비트 범위 | 내용 |
|---------|------|
| [39:32] | 습도 정수 (8-bit) |
| [31:24] | 습도 소수 (8-bit) |
| [23:16] | 온도 정수 (8-bit) |
| [15:8] | 온도 소수 (8-bit) |
| [7:0] | Checksum (8-bit) |

**FSM 상태 구성**

| 상태 | 동작 |
|------|------|
| `IDLE` | 대기. `start` 신호 또는 60초 자동 카운트로 `START` 전환 |
| `START` | `dhtio = 0` 출력, 19ms 유지 후 `WAIT` 전환 |
| `WAIT` | `dhtio`를 Hi-Z로 해제, 센서 응답 대기 |
| `SYNC_L` | 센서의 80us Low 응답 확인 |
| `SYNC_H` | 센서의 80us High 응답 확인 |
| `DATA_SYNC` | 각 비트 시작 Low 구간 대기 |
| `DATA_C` | High 구간 tick 카운팅으로 `'0'` / `'1'` 판별 (40회 반복) |
| `STOP` | Checksum 검증 후 `IDLE` 복귀. 성공 시 `dht11_valid = 1` |

**Tri-state 버스 제어**

`io_sel` 신호로 FPGA 출력 구간과 센서 수신 구간을 구분하여 단일 핀을 공유한다.

- `io_sel = 1` : FPGA가 `dhtio`를 구동 (START 구간)
- `io_sel = 0` : Hi-Z 해제, 센서가 `dhtio`를 구동 (수신 구간)

**Synchronizer & Edge Detector**

외부 신호 `dhtio`의 메타스타빌리티 방지를 위해 2-FF 동기화를 적용하고, 동기화된 신호를 기반으로 상승 / 하강 에지를 검출한다.

**Watchdog 타이머**

통신 오류 또는 연결 단선으로 FSM이 중간 상태에서 멈추는 경우를 대비한 자동 복구 로직이다. IDLE이 아닌 상태에서 약 1초 이상 유지되면 강제로 IDLE로 복귀하며, 오류 측정 시에는 이전 유효 값을 유지한다.

**자동 측정 (Auto Start)**

Reset 후 약 0.1초 뒤 첫 측정을 자동 시작하고, 이후 매 60초마다 반복 측정한다.

---

### SR04 Controller

| 상태 | 동작 |
|------|------|
| `IDLE` | `start` 신호 대기 |
| `TRIG` | 10us TTL 트리거 펄스 출력 |
| `WAIT` | echo 신호 대기 (TIMEOUT = 25ms) |
| `CAL_1` | echo High 구간 1us tick 카운팅, 하강 에지 시 `CAL_2`로 전환 |
| `CAL_2` | `distance_x10`을 58씩 반복 감산하여 cm 단위 변환 |

나눗셈 연산자 사용 시 발생하는 Timing Violation을 반복 감산 방식으로 대체하여 WNS 4.029ns를 확보하였다.

---

## 🐛 Trouble Shooting

### 1. DHT11 Reset 직후 오동작 (의도하지 않은 상승 에지)

| 항목 | 내용 |
|------|------|
| 문제 | Reset 직후 `dhtio_edge_rise`가 1회 발생하여 FSM이 잘못된 상태로 진입 |
| 원인 | Synchronizer FF 초기값이 `0`으로 설정되어 DHT11 기본 버스 상태(High)와 불일치, 첫 클록에서 rise edge 오감지 |
| 해결 | Synchronizer FF 초기값을 모두 `1`로 수정하여 기본 버스 상태와 동기화 |
| 결과 | Reset 이후 초기 오동작 제거 및 안정적인 초기화 동작 확보 |

### 2. DHT11 통신 오류 시 FSM 정지

| 항목 | 내용 |
|------|------|
| 문제 | 센서 연결 불량 또는 통신 오류 시 FSM이 중간 상태에서 멈춰 이후 측정 불가 |
| 원인 | 상태 전환 조건이 외부 신호(edge 감지)에만 의존하여 신호 미수신 시 무한 대기 |
| 해결 | Watchdog 타이머 추가, 약 1초 초과 시 강제 IDLE 복귀 및 이전 유효 값 유지 |
| 결과 | 비정상 상황에서도 자동 복구 가능 |

### 3. SR04 통합 후 Negative Slack 발생

| 항목 | 내용 |
|------|------|
| 문제 | 거리 계산에 나눗셈 연산자 사용 시 WNS −2.194ns, Failing Endpoints 13개 발생 |
| 원인 | 나눗셈 연산자가 조합 논리로 합성될 때 깊은 경로를 생성하여 타이밍 위반 |
| 해결 | 반복 감산 방식으로 대체 |
| 결과 | WNS 4.029ns, Failing Endpoints 0 |

### 4. FND Controller 7-Segment 표시 오류

| 항목 | 내용 |
|------|------|
| 문제 | 기존 Stopwatch의 `time_sel` 비트 폭 부족으로 센서 데이터 표시 범위 제한 |
| 원인 | 기존 MUX 구조가 6-bit 입력 기준으로 설계되어 상위 자릿수 표시 불가 |
| 해결 | 각 자릿수 전용 MUX 추가, 입력 비트 폭 26-bit로 통일 |
| 결과 | SR04 / DHT11 데이터 정상 출력, 문자 매핑 확장 |

---

## ✅ 시뮬레이션 검증 항목

- **FIFO**: push / pop 동작에 따른 full / empty 플래그 정상 전환 확인
- **UART Loopback**: 송신 데이터와 수신 데이터 일치 확인
- **ASCII SENDER**: 시간 포맷 생성 및 tx_start 펄스 순서 확인
- **SR04**: echo 에지 검출, 거리 계산 정확도, TIMEOUT 25ms 동작 확인
- **DHT11**: 0/1 비트 판별, Checksum 성공/실패 분기, Watchdog 1초 복구 동작 확인
- **통합 시뮬레이션**: PC 명령 `M`으로 모드 전환, DHT11 자동 측정, SR04 타임아웃/비정상 응답 처리, UART 31-byte 전송 확인

---

## ⚙️ 타이밍 결과 (Vivado Implementation)

| 항목 | 값 |
|------|----|
| Worst Negative Slack (WNS) | 4.029 ns |
| Total Negative Slack (TNS) | 0 ns |
| Failing Endpoints | 0 |

> SR04 나눗셈 연산자를 반복 감산 방식으로 교체 후 모든 타이밍 제약 조건 만족

---

## 🖥️ 개발 환경

| 항목 | 내용 |
|------|------|
| HDL | Verilog |
| EDA Tool | Xilinx Vivado |
| 타겟 보드 | Digilent Basys3 (Artix-7) |
| 시뮬레이터 | Vivado Simulator (XSim) |
| UART 설정 | 9600 bps / 8-bit / Stop 1-bit / No parity |
