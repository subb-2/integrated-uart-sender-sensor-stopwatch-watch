# 통합 디지털 시스템 설계 — 2조

📅 프로젝트 정보

- **진행 기간**: 2026.02.12 ~ 2026.02.22
- **설계 대상**: SR04 / DHT11 센서 통합 + UART PC 인터페이스 + Stopwatch / Clock 시스템
- **기술 스택**: `Verilog`, `Vivado`, `Basys3 (Artix-7)`, `Vivado Simulator (XSim)`

---

## 📌 프로젝트 개요

기존 Stopwatch / Clock 시스템을 확장하여 SR04, DHT11 센서를 통합하고, UART 통신을 통해 외부 PC와 데이터를 송수신할 수 있도록 설계한 통합 디지털 시스템이다.

- SR04 초음파 센서를 이용한 거리 측정 및 7-Segment 표시
- DHT11 센서를 이용한 온도 / 습도 측정 및 7-Segment 표시
- UART를 이용하여 현재 시간 및 온도 / 습도를 PC로 전송
- PC에서 UART 명령어를 통해 모드 전환 및 버튼 제어 지원

---

## 👥 팀 구성

| 이름 | 담당 |
|------|------|
| 김민수 | UART Controller, FIFO, ASCII SENDER |
| 김성훈 | SR04 Controller |
| 김수빈 | DHT11 Controller |
| 하지훈 | 시스템 통합 (Control Logic, FND, LED Indicator) |

---

## 🏗️ 시스템 구조

<img width="1412" height="670" alt="Image" src="https://github.com/user-attachments/assets/97307d34-2d3f-4ada-be40-c7ca26046a20" />

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

## 🔑 주요 구현 내용

### 1. Control Unit

모드 스위치, 물리 버튼, UART PC 명령을 통합 처리하여 각 서브모듈에 제어 신호를 분배한다.

- PC 제어 모드 활성화 시 `pc_mode_sw`를 우선 선택하여 물리 스위치를 무효화
- 각 모드(Stopwatch / Clock / SR04 / DHT11)에 해당하는 버튼 신호만 해당 모듈로 전달
- 모드 외 버튼 입력은 차단하여 오동작 방지

### 2. UART / FIFO / ASCII SENDER

- **UART TX**: Start bit → 8-bit 데이터 → Stop bit 순서로 LSB 방식 전송
- **UART RX**: 하강 에지 감지 후 비트 중앙(`CNT=7`) 샘플링 (9600 bps, 16배속 tick)
- **FIFO**: write pointer / read pointer 방식, Full 시 write 차단으로 오버플로 방지
- **ASCII SENDER**: FND Controller로부터 시간 / 온습도 데이터를 받아 `HH:MM:SS`, `T xx.xC`, `H xx.x%` 포맷으로 순차 전송

### 3. SR04 Controller

| 상태 | 동작 |
|------|------|
| `IDLE` | `start` 신호 대기 |
| `TRIG` | 10us TTL 트리거 펄스 출력 |
| `WAIT` | echo 신호 대기 (TIMEOUT = 25ms) |
| `CAL_1` | echo High 구간 1us tick 카운팅, 하강 에지 시 `CAL_2`로 전환 |
| `CAL_2` | `distance_x10`을 58씩 반복 감산하여 cm 단위 변환 |

나눗셈 연산자 사용 시 발생하는 Timing Violation을 반복 감산 방식으로 대체하여 WNS 4.029ns를 확보하였다.

### 4. DHT11 Controller

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

- **Tri-state 버스 제어**: `io_sel` 신호로 FPGA 출력 구간과 센서 수신 구간을 구분
- **Synchronizer & Edge Detector**: 외부 신호의 메타스타빌리티 방지를 위해 2-FF 동기화 적용
- **Watchdog 타이머**: IDLE이 아닌 상태에서 약 1초 이상 유지되면 강제 IDLE 복귀, 오류 시 이전 유효 값 유지
- **자동 측정**: Reset 후 약 0.1초 뒤 첫 측정 자동 시작, 이후 매 60초마다 반복

### 5. FND Controller

입력 비트 폭을 26-bit로 통일하고, SR04 / DHT11 전용 MUX를 추가하였다.

| 모드 | 표시 형식 |
|------|---------|
| Stopwatch / Clock | HH:MM / SS:CC |
| SR04 | `Sr04` / `xxx.x cm` |
| DHT11 | `t xx.x` / `h xx.x` |

---

## ✅ 검증 결과

| 검증 항목 | 결과 |
|---------|------|
| FIFO push/pop 경계 조건 | ✅ Full / Empty 플래그 정상 전환 확인 |
| UART Loopback | ✅ 송수신 데이터 일치 확인 |
| ASCII SENDER | ✅ 시간 포맷 생성 및 `tx_start` 펄스 순서 확인 |
| SR04 거리 계산 | ✅ echo 에지 검출, 정확도 및 TIMEOUT 25ms 동작 확인 |
| DHT11 비트 판별 | ✅ 0/1 판별, Checksum 성공/실패 분기, Watchdog 1초 복구 확인 |
| 통합 시뮬레이션 | ✅ PC 명령 `M` 모드 전환, DHT11 자동 측정, UART 31-byte 전송 확인 |

---

## 🚀 문제 해결 (Troubleshooting)

### 1. DHT11 Reset 직후 오동작 (의도하지 않은 상승 에지)

- **문제**: Reset 직후 `dhtio_edge_rise`가 1회 발생하여 FSM이 잘못된 상태로 진입
- **원인**: Synchronizer FF 초기값이 `0`으로 설정되어 DHT11 기본 버스 상태(High)와 불일치, 첫 클록에서 rise edge 오감지
- **해결**: Synchronizer FF 초기값을 모두 `1`로 수정하여 기본 버스 상태와 동기화
- **결과**: Reset 이후 초기 오동작 제거 및 안정적인 초기화 동작 확보

### 2. DHT11 통신 오류 시 FSM 정지

- **문제**: 센서 연결 불량 또는 통신 오류 시 FSM이 중간 상태에서 멈춰 이후 측정 불가
- **원인**: 상태 전환 조건이 외부 신호(edge 감지)에만 의존하여 신호 미수신 시 무한 대기
- **해결**: Watchdog 타이머 추가, 약 1초 초과 시 강제 IDLE 복귀 및 이전 유효 값 유지
- **결과**: 비정상 상황에서도 자동 복구 가능

### 3. SR04 통합 후 Negative Slack 발생

- **문제**: 거리 계산에 나눗셈 연산자 사용 시 WNS −2.194ns, Failing Endpoints 13개 발생
- **원인**: 나눗셈 연산자가 조합 논리로 합성될 때 깊은 경로를 생성하여 타이밍 위반
- **해결**: 반복 감산 방식으로 대체 (`distance_x10 <= distance_x10 - 19'd58`)
- **결과**: WNS 4.029ns, Failing Endpoints 0

### 4. FND Controller 7-Segment 표시 오류

- **문제**: 기존 Stopwatch의 `time_sel` 비트 폭 부족으로 센서 데이터 표시 범위 제한
- **원인**: 기존 MUX 구조가 6-bit 입력 기준으로 설계되어 상위 자릿수 표시 불가
- **해결**: 각 자릿수 전용 MUX 추가, 입력 비트 폭 26-bit로 통일
- **결과**: SR04 / DHT11 데이터 정상 출력, 문자 매핑 확장

---

## ⚙️ 타이밍 결과 (Vivado Implementation)

| 항목 | 값 |
|------|----|
| Worst Negative Slack (WNS) | 4.029 ns |
| Total Negative Slack (TNS) | 0 ns |
| Failing Endpoints | 0 |

> SR04 나눗셈 연산자를 반복 감산 방식으로 교체 후 모든 타이밍 제약 조건 만족

---

## 📚 배운 점

- **센서 인터페이스 설계**: Single-Wire Half-Duplex 방식의 DHT11처럼 타이밍에 민감한 프로토콜은 FSM 상태 전환 조건을 외부 신호에만 의존하면 안 된다는 것을 체득. Watchdog처럼 하드웨어 레벨의 예외 처리가 실제 시스템 안정성을 결정함.
- **합성 타이밍 의식**: 나눗셈 연산자처럼 RTL에서 자연스러운 표현이 합성 후 깊은 조합 경로를 만들 수 있다는 것을 직접 수치로 확인. 반복 감산 방식으로의 전환이 단순 최적화가 아니라 타이밍 클로저의 핵심임을 이해.
- **시스템 통합 설계**: 개별 모듈이 독립적으로 동작해도 버스 구조로 연결하면 새로운 문제가 생긴다는 것을 경험. FND 비트 폭 문제처럼 인터페이스 계약을 모듈 설계 단계에서 명확히 정의해야 통합 시 충돌이 없음.
- **통합 검증의 중요성**: UART Loopback, Watchdog 복구, TIMEOUT 분기처럼 정상 동작만이 아니라 오류 경로를 시뮬레이션으로 명시적으로 검증하는 습관이 하드웨어 설계의 신뢰성을 결정함.

---

## 📌 향후 개선 방향

- UART RX 명령의 에지 검출 기반 처리 (현재 레벨 기반으로 FSM 재진입 가능성 존재)
- DHT11 60초 자동 측정 주기 UART 명령으로 동적 변경 미지원
- SR04 측정 범위 400cm 초과 시 표시 처리 미구현
- PC 명령 `Q` 출력에 CC(centisecond) 포함 여부 정책 미확정

---

## 🖥️ 개발 환경

| 항목 | 내용 |
|------|------|
| HDL | Verilog |
| EDA Tool | Xilinx Vivado |
| 타겟 보드 | Digilent Basys3 (Artix-7) |
| 시뮬레이터 | Vivado Simulator (XSim) |
| UART 설정 | 9600 bps / 8-bit / Stop 1-bit / No parity |

---

## 📁 파일 구성

```text
sources_1/
├── top_stopwatch_clock.v        # 최상위 모듈 — 전체 서브모듈 연결
├── control_unit.v               # 모드/버튼/스위치 제어 및 PC 명령 처리
├── fnd_controller.v             # 7-Segment 표시 제어 (BCD 변환, 소수점 포함)
├── uart_top.v                   # UART TX/RX + FIFO + ASCII SENDER 통합
├── fifo.v                       # RX / TX 버퍼용 FIFO
├── sr04_ctrl_top.v              # SR04 초음파 거리 측정 FSM
├── dht11_ctrl.v                 # DHT11 온습도 센서 FSM
└── btn_debounce.v               # 버튼 디바운스 처리

constrs_1/
└── (Basys3 pin constraint .xdc)
```
