# 🔹 Integrated Digital System Design

> Basys3(Xilinx FPGA) 기반 Stopwatch / Clock 시스템에  
> SR04 초음파 센서, DHT11 온습도 센서, UART PC 인터페이스를 통합한 디지털 시스템

---

## 📌 Project Overview

기존 Stopwatch / Clock 시스템을 확장하여 센서 및 UART 기능을 통합한 FPGA 기반 디지털 시스템입니다.

- SR04 초음파 센서를 이용한 거리 측정
- DHT11 센서를 이용한 온습도 측정
- UART 기반 PC 제어 및 데이터 출력
- 7-Segment(FND) 실시간 표시
- 물리 버튼 + UART 명령 기반 통합 제어

---

## 🎯 Main Features

- Stopwatch / Clock 모드 지원
- SR04 거리 측정 및 실시간 표시
- DHT11 온습도 측정 및 UART 출력
- UART ASCII 기반 PC 제어
- FIFO 기반 UART 데이터 버퍼링
- Auto Measurement / Watchdog Recovery
- Timing Optimization 적용

---

## 🕹️ Input Mapping

### Button Mapping

| Button | Stopwatch | Clock | SR04 | DHT11 |
|---|---|---|---|---|
| BTN_U | Run / Stop | Up | — | — |
| BTN_C | — | Time-set Select | — | — |
| BTN_D | Clear | Down | — | — |
| BTN_L | — | — | Distance Measure | — |
| BTN_R | — | — | — | Temperature/Humidity Measure |

---

### Switch Mapping

| Switch | Function |
|---|---|
| SW[1] & SW[4] | Mode Select |
| SW[0] | Stopwatch Up / Down |
| SW[2] | Display Format Change |
| SW[3] | Clock Time-set Mode |
| SW[15] | System Reset |

---

### UART ASCII Command Mapping

| Command | Function |
|---|---|
| `M` | PC Control Mode Toggle |
| `R` | Run / Stop / Up |
| `N` | Time-set Select |
| `C` | Clear / Down |
| `T` | SR04 Measurement Start |
| `H` | DHT11 Measurement Start |
| `Q` | Current Time + Sensor Data Output |
| `0~4` | Switch Toggle |

---

## 🏗️ System Architecture

```text
Clock / Stopwatch
        │
        ▼
 Control Unit
        │
 ┌──────┴──────┐
 ▼             ▼
SR04         DHT11
 ▼             ▼
   FND Controller
          │
          ▼
      UART Top
   (FIFO + ASCII)
```

---

## 👨‍💻 Role Distribution

### 김수빈

- DHT11 Controller 설계
- UART ASCII Sender 구현
- FND Controller 확장
- UART-PC 제어 통합
- Sensor/UART 통합 검증
- Timing Violation 개선
- Watchdog Recovery 로직 설계

---

## 🔧 Key Design Points

### UART + FIFO 기반 비동기 데이터 처리

UART 송수신 속도 차이로 인한 데이터 유실을 방지하기 위해 FIFO를 적용하였습니다.

- RX/TX Buffer 분리
- Full / Empty 상태 관리
- Overflow 방지 로직 구현

---

### DHT11 Single-Wire Half Duplex 처리

DHT11의 Single-Wire 통신 특성에 맞추어 Tri-state 기반 입출력 제어를 구현하였습니다.

- FPGA 송신 구간 → Output Mode
- Sensor 응답 구간 → Hi-Z Mode
- Edge Detection 기반 데이터 수신

---

### UART Oversampling

16배속 Tick 기반 Oversampling을 적용하여 UART 수신 안정성을 개선하였습니다.

- Start Bit Falling Edge 검출
- Bit 중앙 샘플링
- Noise 대응 안정성 확보

---

### Timing Optimization

SR04 거리 계산 과정에서 발생한 Timing Violation을 개선하였습니다.

- Divider(`/`) 제거
- 반복 감산 기반 거리 계산 적용
- Timing Slack 확보

---

### Fault Recovery 설계

센서 응답 오류 상황에서 FSM이 정지하지 않도록 Watchdog Timer를 추가하였습니다.

- 일정 시간 이상 응답 없을 경우 강제 IDLE 복귀
- 이전 유효 데이터 유지
- 시스템 자동 복구 지원

---

## 🐛 Trouble Shooting

### 1. DHT11 Reset 직후 False Edge Detection

| 항목 | 내용 |
|---|---|
| 문제 | Reset 직후 의도하지 않은 `rise edge`가 검출되어 FSM이 오동작 |
| 원인 | Synchronizer 초기값이 `0`으로 설정되어 DHT11 기본 Idle 상태(High)와 불일치 |
| 해결 | Synchronizer FF 초기값을 모두 `1'b1`로 수정 |
| 결과 | Reset 이후 초기 오동작 제거 및 안정적인 초기화 동작 확보 |

---

### 2. DHT11 통신 오류 시 FSM Deadlock

| 항목 | 내용 |
|---|---|
| 문제 | 센서 응답 실패 시 FSM이 특정 상태에서 멈추는 현상 발생 |
| 원인 | 상태 전환 조건이 외부 Edge 신호에만 의존 |
| 해결 | Watchdog Timer 추가 후 일정 시간 초과 시 강제 IDLE 복귀 |
| 결과 | 비정상 상황에서도 자동 복구 가능 |

---

### 3. SR04 Distance Calculation Timing Violation

| 항목 | 내용 |
|---|---|
| 문제 | 거리 계산 시 `/58` 나눗셈 연산으로 Negative Slack 발생 |
| 원인 | Divider가 깊은 조합 논리 경로 생성 |
| 해결 | 반복 감산 방식으로 거리 계산 변경 |
| 결과 | WNS `-2.194ns → 4.029ns`, Failing Endpoint 제거 |

---

### 4. FND Display Expansion Issue

| 항목 | 내용 |
|---|---|
| 문제 | 기존 Stopwatch 기반 FND 구조로 센서 데이터 표시 불가능 |
| 원인 | 입력 비트 폭 부족 및 기존 MUX 구조 한계 |
| 해결 | 입력 비트 폭 확장 및 센서 전용 Display MUX 추가 |
| 결과 | SR04 / DHT11 데이터 정상 출력 |

---

### 5. UART 입력과 물리 버튼 충돌 문제

| 항목 | 내용 |
|---|---|
| 문제 | UART 제어와 물리 버튼 입력이 동시에 들어오며 동작 충돌 발생 |
| 원인 | 동일 제어 신호를 두 입력이 동시에 제어 |
| 해결 | PC Control Mode를 분리하여 UART 입력 우선 처리 |
| 결과 | 물리 입력 / UART 입력 간 충돌 제거 |

---

## ✅ Verification

- FIFO Full / Empty 동작 검증
- UART Loopback 검증
- ASCII Sender 포맷 검증
- SR04 Distance Measurement 검증
- DHT11 Checksum 검증
- Timeout / Fault Recovery 검증
- UART-PC Control 검증
- 통합 시스템 동작 검증

---

## ⚙️ Timing Result

| Item | Result |
|---|---|
| Worst Negative Slack (WNS) | 4.029 ns |
| Total Negative Slack (TNS) | 0 ns |
| Failing Endpoints | 0 |

---

## 🖥️ Development Environment

| Item | Description |
|---|---|
| HDL | Verilog |
| Tool | Xilinx Vivado |
| FPGA Board | Digilent Basys3 |
| Simulator | Vivado XSim |
| UART | 9600bps / 8N1 |
