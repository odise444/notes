---
title: "AD7280A BMS 개발 삽질기 #26 - 현장 트러블슈팅"
date: 2024-12-14
draft: false
tags: ["AD7280A", "BMS", "STM32", "트러블슈팅", "디버깅", "현장"]
categories: ["BMS 개발"]
series: ["AD7280A BMS 개발"]
summary: "랩에서는 멀쩡하던 BMS가 현장에서 말썽? 실전에서 마주친 문제들과 해결 과정을 공유한다."
---

## 지난 글 요약

[지난 글](/posts/bms/ad7280a-bms-dev-25/)에서 72V 팩 첫 연결에 성공했다. 하지만 **현장은 다르다**. 예상 못한 문제들과의 싸움을 시작하자.

## 현장은 실험실이 아니다

```
실험실:                   현장:
┌─────────────────┐      ┌─────────────────┐
│ • 온도: 25°C    │      │ • 온도: -20~60°C│
│ • 습도: 50%     │      │ • 습도: 10~95%  │
│ • 진동: 없음    │      │ • 진동: 심함    │
│ • 노이즈: 적음  │      │ • 노이즈: 극심  │
│ • 전원: 안정    │      │ • 전원: 불안정  │
│ • 먼지: 없음    │      │ • 먼지, 물, 기름│
└─────────────────┘      └─────────────────┘
```

## Case 1: 갑자기 통신 끊김

### 증상

```
[10:23:15] Cell voltages: OK
[10:23:16] Cell voltages: OK
[10:23:17] Cell voltages: OK
[10:23:18] SPI TIMEOUT ERROR!
[10:23:19] SPI TIMEOUT ERROR!
[10:23:20] System reset...
```

### 현장 상황

- 지게차 BMS
- 모터 구동 시 발생
- 랩에서는 재현 안 됨

### 원인 분석

```
모터 시작 →
┌────────────────────────────────────────────┐
│         대전류 (200A+)                      │
│              ↓                              │
│   ┌──────────────────┐                      │
│   │ 전원 전압 드롭    │                      │
│   │ 72V → 65V        │                      │
│   └────────┬─────────┘                      │
│            ↓                                │
│   ┌──────────────────┐                      │
│   │ AD7280A VDD 강하  │ 전원 불안정         │
│   │ 3.3V → 2.8V      │                      │
│   └────────┬─────────┘                      │
│            ↓                                │
│   ┌──────────────────┐                      │
│   │ IC 리셋/오동작    │                      │
│   └──────────────────┘                      │
└────────────────────────────────────────────┘
```

### 해결

```c
// 1. 전원 안정화 - 벌크 커패시터 추가
// 회로: VDD에 100μF 추가

// 2. 전압 모니터링 추가
float vdd_voltage = ADC_ReadVDD();
if (vdd_voltage < 3.0f) {
    Log_Event(EVT_LOW_VDD, 0, (uint16_t)(vdd_voltage * 1000), 0);
    // 대전류 동작 제한 요청
    CAN_SendPowerLimitRequest();
}

// 3. 통신 재시도 로직 강화
HAL_StatusTypeDef AD7280A_ReadWithRetry(uint8_t dev, uint8_t reg, 
                                         uint8_t *data, int retries) {
    for (int i = 0; i < retries; i++) {
        if (AD7280A_ReadRegister(dev, reg, data) == HAL_OK) {
            return HAL_OK;
        }
        HAL_Delay(1);
        
        // 2회 실패 시 재초기화
        if (i == 1) {
            AD7280A_SoftReset();
            HAL_Delay(10);
        }
    }
    return HAL_ERROR;
}
```

## Case 2: 온도 측정 이상

### 증상

```
현장 보고:
"온도가 -40°C로 표시됩니다. 여름인데요..."

로그 분석:
[12:45:00] NTC1: -40.0°C  ← 최소값
[12:45:01] NTC1: -40.0°C
[12:45:02] NTC1: 85.3°C   ← 갑자기 튐
[12:45:03] NTC1: -40.0°C
```

### 원인

```
NTC 연결 상태:
         정상                    단선
    ┌─────────┐              ┌─────────┐
    │         │              │         │
   VDD       ADC            VDD       ADC
    │    ┌────┤              │    ┌────┤
    └─R──┤NTC │              └─R──┤    │← 끊어짐
         └────┘                   └────┘
    
    ADC = 분압값              ADC = VDD (풀업)
    → 정상 온도              → 최소 온도 (-40°C)
```

### 해결

```c
// NTC 연결 상태 감지
typedef enum {
    NTC_OK = 0,
    NTC_OPEN,       // 단선 (ADC 최대)
    NTC_SHORT,      // 단락 (ADC 최소)
    NTC_UNSTABLE    // 불안정
} ntc_status_t;

ntc_status_t NTC_CheckConnection(int channel) {
    uint16_t adc = ADC_Read(channel);
    
    // 단선: ADC 값이 거의 최대 (풀업 저항으로)
    if (adc > 4000) {
        return NTC_OPEN;
    }
    
    // 단락: ADC 값이 거의 최소
    if (adc < 100) {
        return NTC_SHORT;
    }
    
    // 불안정 체크: 연속 측정 편차
    static uint16_t last_adc[4] = {0};
    int diff = abs((int)adc - (int)last_adc[channel]);
    last_adc[channel] = adc;
    
    if (diff > 500) {  // 급격한 변화
        return NTC_UNSTABLE;
    }
    
    return NTC_OK;
}

// 온도 읽기 시 상태 확인
float GetTemperature(int channel) {
    ntc_status_t status = NTC_CheckConnection(channel);
    
    switch (status) {
    case NTC_OPEN:
        Log_Event(EVT_NTC_OPEN, channel, 0, 0);
        return TEMP_SENSOR_ERROR;  // 특수값 반환
        
    case NTC_SHORT:
        Log_Event(EVT_NTC_SHORT, channel, 0, 0);
        return TEMP_SENSOR_ERROR;
        
    case NTC_UNSTABLE:
        // 커넥터 접촉 불량 의심
        return TEMP_SENSOR_ERROR;
        
    default:
        return CalculateTemperature(ADC_Read(channel));
    }
}
```

## Case 3: 밸런싱이 안 됨

### 증상

```
밸런싱 ON 명령 후에도 셀 편차 그대로:
  밸런싱 전: 3.42V, 3.35V, 3.38V ... (편차 70mV)
  1시간 후:  3.42V, 3.35V, 3.38V ... (변화 없음)
```

### 디버깅 과정

```c
// 1단계: 레지스터 확인
void DebugBalancing(void) {
    for (int dev = 0; dev < 4; dev++) {
        uint8_t cell_bal = AD7280A_ReadRegister(dev, REG_CELL_BALANCE);
        printf("Dev %d CELL_BALANCE: 0x%02X\n", dev, cell_bal);
        
        uint8_t ctrl = AD7280A_ReadRegister(dev, REG_CTRL_HB);
        printf("Dev %d CTRL_HB: 0x%02X\n", dev, ctrl);
    }
}

// 결과:
// Dev 0 CELL_BALANCE: 0x3F  ← 전부 ON? 이상!
// Dev 0 CTRL_HB: 0x00       ← CB Timer 비활성화!
```

### 원인

밸런싱 타이머 설정 누락!

```c
// 잘못된 코드
void EnableBalancing(uint8_t cells) {
    AD7280A_WriteRegister(dev, REG_CELL_BALANCE, cells);
    // CB_TIMER 설정 안 함 → 즉시 OFF
}

// 올바른 코드
void EnableBalancing(uint8_t dev, uint8_t cells) {
    // 1. 밸런싱 타이머 설정 (약 36초)
    uint8_t ctrl_hb = AD7280A_ReadRegister(dev, REG_CTRL_HB);
    ctrl_hb |= (0x1F << 3);  // CB_TIMER = 31 (최대)
    AD7280A_WriteRegister(dev, REG_CTRL_HB, ctrl_hb);
    
    // 2. 밸런싱 셀 선택
    AD7280A_WriteRegister(dev, REG_CELL_BALANCE, cells);
}
```

### 추가 확인: 하드웨어

```
밸런싱 회로 확인:
     VCell
       │
      [R] 33Ω 밸런싱 저항
       │
      [SW] AD7280A 내부 스위치
       │
      GND

체크 포인트:
□ 저항 값 (33Ω ±5%)
□ 저항 정격 (0.5W 이상)
□ 납땜 상태
□ 패턴 단선
```

## Case 4: CAN 통신 간헐적 실패

### 증상

```
CAN 모니터 로그:
[14:00:00.000] 0x200: 4C 2D 00 00 32 01 00 00  OK
[14:00:00.100] 0x201: 00 00 00 00 00 00 00 00  OK
[14:00:00.200] 0x200: (missing)                 ← 누락!
[14:00:00.300] 0x201: 00 00 00 00 00 00 00 00  OK
[14:00:00.400] 0x200: 4C 2D 00 00 32 01 00 00  OK
```

### 원인 추적

```c
// CAN 에러 카운터 모니터링
void CAN_DiagnoseErrors(void) {
    uint32_t esr = CAN1->ESR;
    
    uint8_t tec = (esr >> 16) & 0xFF;  // TX Error Counter
    uint8_t rec = (esr >> 24) & 0xFF;  // RX Error Counter
    
    printf("CAN Error Status:\n");
    printf("  TX Error Count: %d\n", tec);
    printf("  RX Error Count: %d\n", rec);
    printf("  Last Error Code: %d\n", (esr >> 4) & 0x07);
    
    // 에러 코드 해석
    switch ((esr >> 4) & 0x07) {
    case 0: printf("  → No error\n"); break;
    case 1: printf("  → Stuff error\n"); break;
    case 2: printf("  → Form error\n"); break;
    case 3: printf("  → ACK error\n"); break;      // 수신자 없음
    case 4: printf("  → Bit recessive error\n"); break;
    case 5: printf("  → Bit dominant error\n"); break;
    case 6: printf("  → CRC error\n"); break;
    case 7: printf("  → Custom\n"); break;
    }
}
```

### 원인: 종단 저항

```
CAN 버스 토폴로지:
                      
  BMS ─────┬────── ECU ─────┬────── 모터컨트롤러
          [R]               [R]
          120Ω              120Ω
           │                 │
          GND               GND

문제: 한쪽 종단저항 없음 → 반사파 발생 → 비트 에러
```

### 해결

```c
// 1. 종단저항 확인 (CAN_H와 CAN_L 사이)
// 멀티미터로 60Ω 측정되어야 함 (120Ω // 120Ω)

// 2. 케이블 길이 점검
// 500kbps에서 최대 100m 권장

// 3. 소프트웨어 재전송 로직
HAL_StatusTypeDef CAN_TransmitWithRetry(CAN_TxHeaderTypeDef *header,
                                         uint8_t *data, int retries) {
    uint32_t mailbox;
    
    for (int i = 0; i < retries; i++) {
        if (HAL_CAN_AddTxMessage(&hcan1, header, data, &mailbox) == HAL_OK) {
            // 전송 완료 대기
            uint32_t start = HAL_GetTick();
            while (HAL_CAN_IsTxMessagePending(&hcan1, mailbox)) {
                if (HAL_GetTick() - start > 10) {
                    // 타임아웃 - 취소 후 재시도
                    HAL_CAN_AbortTxRequest(&hcan1, mailbox);
                    break;
                }
            }
            
            // 성공 확인
            if (!HAL_CAN_IsTxMessagePending(&hcan1, mailbox)) {
                return HAL_OK;
            }
        }
        HAL_Delay(1);
    }
    return HAL_ERROR;
}
```

## Case 5: 부팅 시 랜덤 리셋

### 증상

```
전원 ON 후:
- 70%: 정상 부팅
- 30%: 부팅 중 리셋 반복
```

### 디버깅

```c
// 리셋 원인 분석
void AnalyzeResetCause(void) {
    if (__HAL_RCC_GET_FLAG(RCC_FLAG_PORRST)) {
        printf("Reset: Power-On\n");
    }
    if (__HAL_RCC_GET_FLAG(RCC_FLAG_PINRST)) {
        printf("Reset: External Pin\n");
    }
    if (__HAL_RCC_GET_FLAG(RCC_FLAG_SFTRST)) {
        printf("Reset: Software\n");
    }
    if (__HAL_RCC_GET_FLAG(RCC_FLAG_IWDGRST)) {
        printf("Reset: Watchdog\n");  // ← 이것!
    }
    if (__HAL_RCC_GET_FLAG(RCC_FLAG_LPWRRST)) {
        printf("Reset: Low Power\n");
    }
    
    __HAL_RCC_CLEAR_RESET_FLAGS();
}
```

### 원인: 초기화 시간 초과

```c
// 문제 코드
void SystemInit(void) {
    HAL_Init();
    SystemClock_Config();
    
    IWDG_Init(1000);  // 1초 워치독 시작
    
    // 여기서 AD7280A 초기화 (4개 디바이스)
    // 최악의 경우 2초 이상 소요!
    AD7280A_Init();  // ← 워치독 타임아웃!
}

// 수정 코드
void SystemInit(void) {
    HAL_Init();
    SystemClock_Config();
    
    // 긴 초기화 먼저
    AD7280A_Init();  // 충분한 시간 확보
    
    // 초기화 완료 후 워치독 시작
    IWDG_Init(1000);
}

// 또는 초기화 중 워치독 리프레시
void AD7280A_Init(void) {
    for (int dev = 0; dev < 4; dev++) {
        AD7280A_InitDevice(dev);
        IWDG_Refresh();  // 각 디바이스마다 리프레시
    }
}
```

## 트러블슈팅 도구

### 현장 진단 CLI

```c
// UART CLI 명령어
const cli_cmd_t diagnostic_cmds[] = {
    {"status",   CMD_Status,   "Show system status"},
    {"cells",    CMD_Cells,    "Read all cell voltages"},
    {"temps",    CMD_Temps,    "Read all temperatures"},
    {"balance",  CMD_Balance,  "Control balancing"},
    {"can",      CMD_CAN,      "CAN diagnostics"},
    {"reset",    CMD_Reset,    "Reset system"},
    {"log",      CMD_Log,      "Show event log"},
    {"dump",     CMD_Dump,     "Dump all registers"},
    {"test",     CMD_Test,     "Run self-test"},
};

// 현장에서 유용한 명령
// > status
// Pack: 76.8V, 23.5°C, SOC 85%
// Errors: 0, Warnings: 1 (cell imbalance)

// > cells
// Cell  0: 3.21V  Cell  1: 3.20V  Cell  2: 3.21V ...

// > dump
// REG 0x00: 0x12
// REG 0x01: 0x34
// ...
```

### 데이터 로깅

```c
// 문제 재현을 위한 상세 로깅
void DetailedLog(void) {
    static uint32_t log_count = 0;
    
    printf("[%08lu] ", log_count++);
    printf("V:%.2f ", g_bms.pack_voltage_mv / 1000.0f);
    printf("I:%.1f ", g_bms.current_a);
    printf("T:%d ", g_bms.max_temp);
    printf("SOC:%d ", g_bms.soc);
    printf("BAL:0x%02X ", g_bms.balancing_status);
    printf("ERR:0x%04X ", g_bms.error_flags);
    printf("CAN_E:%d ", g_bms.can_error_count);
    printf("SPI_E:%d", g_bms.spi_error_count);
    printf("\n");
}
```

## 정리

| 문제 | 증상 | 원인 | 해결 |
|------|------|------|------|
| 통신 끊김 | SPI 타임아웃 | 전원 드롭 | 커패시터, 재시도 |
| 온도 이상 | -40°C 고정 | NTC 단선 | 연결 감지 |
| 밸런싱 안 됨 | 편차 유지 | 타이머 미설정 | CB_TIMER 설정 |
| CAN 누락 | 간헐적 실패 | 종단저항 | 120Ω 추가 |
| 부팅 리셋 | 랜덤 리셋 | WDG 타임아웃 | 초기화 순서 |

**교훈**:
1. 현장 환경은 가혹하다
2. 에러 처리가 핵심이다
3. 로깅을 충분히 해라
4. 진단 도구를 만들어라

**다음 글에서**: EMC 대응 - 노이즈와의 전쟁.

---

## 시리즈 네비게이션

**Part 8: 실전 적용편**
- [#25 - 72V 팩 첫 연결](/posts/bms/ad7280a-bms-dev-25/)
- **#26 - 현장 트러블슈팅** ← 현재 글
- #27 - EMC 대응
- #28 - 양산 검사 지그

---

## 참고 자료

- [Embedded Systems Debugging](https://www.embedded.com/debugging-embedded-systems/)
- [CAN Bus Troubleshooting](https://www.csselectronics.com/pages/can-bus-errors-intro-tutorial)
- [Battery System Field Issues](https://www.mdpi.com/1996-1073/14/18/5766)
