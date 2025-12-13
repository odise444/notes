---
title: "AD7280A BMS 개발 삽질기 #17 - CAN 통신 프로토콜 설계"
date: 2024-12-13
draft: false
tags: ["AD7280A", "BMS", "STM32", "CAN", "프로토콜"]
categories: ["BMS 개발"]
series: ["AD7280A BMS 개발"]
summary: "BMS와 외부 시스템 간 CAN 통신 프로토콜을 설계한다. 메시지 ID 할당부터 DBC 파일까지."
---

## 지난 글 요약

[지난 글](/posts/bms/ad7280a-bms-dev-16/)에서 보호 로직을 통합했다. 상태 머신, FET 제어, 폴트 복구까지. 이제 **외부 시스템과 통신**할 차례다.

## 왜 CAN인가?

BMS가 혼자 동작하면 의미 없다:

```
┌─────────┐     CAN      ┌─────────┐
│   BMS   │◄────────────►│ Inverter│
└─────────┘              └─────────┘
     │                        │
     │         CAN Bus        │
     ▼                        ▼
┌─────────┐              ┌─────────┐
│ Charger │              │ Display │
└─────────┘              └─────────┘
```

**CAN의 장점**:
- 노이즈 강함 (차동 신호)
- 멀티 노드 지원
- 에러 검출/복구
- 산업 표준

## CAN 메시지 ID 설계

### ID 할당 전략

CAN ID는 **우선순위**를 결정한다 (낮을수록 높음):

```
ID 범위          용도              우선순위
────────────────────────────────────────────
0x100 ~ 0x1FF   긴급/안전 메시지      높음
0x200 ~ 0x2FF   상태 정보             중간
0x300 ~ 0x3FF   측정 데이터           중간
0x400 ~ 0x4FF   설정/명령             낮음
0x500 ~ 0x5FF   진단/디버그           낮음
```

### BMS CAN ID 정의

```c
// bms_can_id.h

// BMS → 외부 (Broadcast)
#define CAN_ID_BMS_STATUS       0x100   // BMS 상태
#define CAN_ID_BMS_FAULT        0x101   // 폴트 정보
#define CAN_ID_CELL_VOLT_1      0x200   // Cell 1~4 전압
#define CAN_ID_CELL_VOLT_2      0x201   // Cell 5~8 전압
#define CAN_ID_CELL_VOLT_3      0x202   // Cell 9~12 전압
#define CAN_ID_CELL_VOLT_4      0x203   // Cell 13~16 전압
#define CAN_ID_CELL_VOLT_5      0x204   // Cell 17~20 전압
#define CAN_ID_CELL_VOLT_6      0x205   // Cell 21~24 전압
#define CAN_ID_PACK_INFO        0x210   // 팩 전압, 전류
#define CAN_ID_TEMPERATURE      0x220   // 온도 정보
#define CAN_ID_SOC_SOH          0x230   // SOC, SOH

// 외부 → BMS (Command)
#define CAN_ID_BMS_CMD          0x400   // 제어 명령
#define CAN_ID_BMS_CONFIG       0x410   // 설정 변경
#define CAN_ID_BMS_DIAG         0x500   // 진단 요청/응답
```

## 메시지 포맷 설계

### 0x100: BMS Status

```c
typedef struct __attribute__((packed)) {
    uint8_t state;          // BMS 상태 (enum)
    uint8_t flags;          // 상태 플래그
    uint8_t soc;            // SOC (0~100%)
    uint8_t soh;            // SOH (0~100%)
    uint8_t charge_enable;  // 충전 허용
    uint8_t discharge_enable; // 방전 허용
    uint8_t balancing;      // 밸런싱 중
    uint8_t reserved;
} can_msg_bms_status_t;

// flags 비트 정의
#define FLAG_CHARGING       (1 << 0)
#define FLAG_DISCHARGING    (1 << 1)
#define FLAG_BALANCING      (1 << 2)
#define FLAG_FAULT          (1 << 3)
#define FLAG_PRECHARGE      (1 << 4)
```

### 0x101: Fault Info

```c
typedef struct __attribute__((packed)) {
    uint16_t fault_code;    // 폴트 코드
    uint8_t  fault_cell;    // 문제 셀 번호 (1~24)
    uint8_t  fault_device;  // 문제 디바이스 (0~3)
    uint16_t fault_value;   // 측정값 (mV 또는 0.1°C)
    uint16_t threshold;     // 임계값
} can_msg_fault_t;

// 폴트 코드
#define FAULT_NONE          0x0000
#define FAULT_CELL_OV       0x0001  // 셀 과전압
#define FAULT_CELL_UV       0x0002  // 셀 저전압
#define FAULT_PACK_OV       0x0003  // 팩 과전압
#define FAULT_PACK_UV       0x0004  // 팩 저전압
#define FAULT_OVER_TEMP     0x0010  // 과온
#define FAULT_UNDER_TEMP    0x0011  // 저온
#define FAULT_OVER_CURRENT  0x0020  // 과전류
#define FAULT_COMM_ERROR    0x0100  // 통신 오류
#define FAULT_BALANCE_ERROR 0x0200  // 밸런싱 오류
```

### 0x200~0x205: Cell Voltages

```c
typedef struct __attribute__((packed)) {
    uint16_t cell_1;    // Cell N*4+1 전압 (mV)
    uint16_t cell_2;    // Cell N*4+2 전압 (mV)
    uint16_t cell_3;    // Cell N*4+3 전압 (mV)
    uint16_t cell_4;    // Cell N*4+4 전압 (mV)
} can_msg_cell_volt_t;

// 24셀 → 6개 메시지 (4셀씩)
// 0x200: Cell 1~4
// 0x201: Cell 5~8
// ...
// 0x205: Cell 21~24
```

### 0x210: Pack Info

```c
typedef struct __attribute__((packed)) {
    uint16_t pack_voltage;  // 팩 전압 (0.1V 단위)
    int16_t  pack_current;  // 팩 전류 (0.1A, 부호 있음)
    uint16_t max_cell;      // 최고 셀 전압 (mV)
    uint16_t min_cell;      // 최저 셀 전압 (mV)
} can_msg_pack_info_t;
```

### 0x220: Temperature

```c
typedef struct __attribute__((packed)) {
    int8_t temp_1;          // 온도 센서 1 (°C)
    int8_t temp_2;          // 온도 센서 2
    int8_t temp_3;          // 온도 센서 3
    int8_t temp_4;          // 온도 센서 4
    int8_t temp_max;        // 최고 온도
    int8_t temp_min;        // 최저 온도
    int8_t temp_avg;        // 평균 온도
    uint8_t reserved;
} can_msg_temperature_t;
```

### 0x400: Command

```c
typedef struct __attribute__((packed)) {
    uint8_t command;        // 명령 코드
    uint8_t param[7];       // 파라미터
} can_msg_command_t;

// 명령 코드
#define CMD_NONE            0x00
#define CMD_CHARGE_ENABLE   0x01    // 충전 허용
#define CMD_CHARGE_DISABLE  0x02    // 충전 금지
#define CMD_DISCHARGE_ENABLE  0x03  // 방전 허용
#define CMD_DISCHARGE_DISABLE 0x04  // 방전 금지
#define CMD_BALANCE_START   0x10    // 밸런싱 시작
#define CMD_BALANCE_STOP    0x11    // 밸런싱 중지
#define CMD_CLEAR_FAULT     0x20    // 폴트 클리어
#define CMD_RESET           0xFF    // 리셋
```

## CAN 드라이버 구현

### 초기화

```c
// bms_can.c

static CAN_HandleTypeDef hcan;

void BMS_CAN_Init(void) {
    // GPIO: PA11=RX, PA12=TX (이미 설정됨)
    
    // CAN 설정: 500kbps @ 36MHz APB1
    hcan.Instance = CAN1;
    hcan.Init.Prescaler = 4;
    hcan.Init.Mode = CAN_MODE_NORMAL;
    hcan.Init.SyncJumpWidth = CAN_SJW_1TQ;
    hcan.Init.TimeSeg1 = CAN_BS1_13TQ;
    hcan.Init.TimeSeg2 = CAN_BS2_2TQ;
    hcan.Init.TimeTriggeredMode = DISABLE;
    hcan.Init.AutoBusOff = ENABLE;
    hcan.Init.AutoWakeUp = DISABLE;
    hcan.Init.AutoRetransmission = ENABLE;
    hcan.Init.ReceiveFifoLocked = DISABLE;
    hcan.Init.TransmitFifoPriority = DISABLE;
    
    HAL_CAN_Init(&hcan);
    
    // 필터: 0x400~0x4FF만 수신 (명령)
    CAN_FilterTypeDef filter;
    filter.FilterBank = 0;
    filter.FilterMode = CAN_FILTERMODE_IDMASK;
    filter.FilterScale = CAN_FILTERSCALE_32BIT;
    filter.FilterIdHigh = (0x400 << 5);
    filter.FilterIdLow = 0;
    filter.FilterMaskIdHigh = (0x700 << 5);  // 0x4XX만
    filter.FilterMaskIdLow = 0;
    filter.FilterFIFOAssignment = CAN_RX_FIFO0;
    filter.FilterActivation = ENABLE;
    
    HAL_CAN_ConfigFilter(&hcan, &filter);
    
    // 인터럽트 활성화
    HAL_CAN_ActivateNotification(&hcan, CAN_IT_RX_FIFO0_MSG_PENDING);
    
    HAL_CAN_Start(&hcan);
}
```

### 송신 함수

```c
HAL_StatusTypeDef BMS_CAN_Transmit(uint32_t id, uint8_t *data, uint8_t len) {
    CAN_TxHeaderTypeDef header;
    uint32_t mailbox;
    
    header.StdId = id;
    header.ExtId = 0;
    header.IDE = CAN_ID_STD;
    header.RTR = CAN_RTR_DATA;
    header.DLC = len;
    header.TransmitGlobalTime = DISABLE;
    
    return HAL_CAN_AddTxMessage(&hcan, &header, data, &mailbox);
}
```

### 수신 콜백

```c
void HAL_CAN_RxFifo0MsgPendingCallback(CAN_HandleTypeDef *hcan) {
    CAN_RxHeaderTypeDef header;
    uint8_t data[8];
    
    if (HAL_CAN_GetRxMessage(hcan, CAN_RX_FIFO0, &header, data) == HAL_OK) {
        BMS_CAN_ProcessMessage(header.StdId, data, header.DLC);
    }
}

void BMS_CAN_ProcessMessage(uint32_t id, uint8_t *data, uint8_t len) {
    switch (id) {
    case CAN_ID_BMS_CMD:
        BMS_ProcessCommand((can_msg_command_t *)data);
        break;
        
    case CAN_ID_BMS_CONFIG:
        BMS_ProcessConfig(data, len);
        break;
        
    case CAN_ID_BMS_DIAG:
        BMS_ProcessDiagRequest(data, len);
        break;
    }
}
```

### 명령 처리

```c
void BMS_ProcessCommand(can_msg_command_t *cmd) {
    switch (cmd->command) {
    case CMD_CHARGE_ENABLE:
        g_bms.charge_allowed = true;
        break;
        
    case CMD_CHARGE_DISABLE:
        g_bms.charge_allowed = false;
        BMS_FET_ChargeOff();
        break;
        
    case CMD_DISCHARGE_ENABLE:
        g_bms.discharge_allowed = true;
        break;
        
    case CMD_DISCHARGE_DISABLE:
        g_bms.discharge_allowed = false;
        BMS_FET_DischargeOff();
        break;
        
    case CMD_BALANCE_START:
        g_bms.balance_enabled = true;
        break;
        
    case CMD_BALANCE_STOP:
        g_bms.balance_enabled = false;
        ad7280a_stop_balancing(&g_ad7280a);
        break;
        
    case CMD_CLEAR_FAULT:
        BMS_ClearFault();
        break;
        
    case CMD_RESET:
        NVIC_SystemReset();
        break;
    }
}
```

## 주기적 메시지 송신

```c
// 송신 타이머 (100ms 주기)
static uint32_t tx_counter = 0;

void BMS_CAN_PeriodicTx(void) {
    tx_counter++;
    
    // 100ms: Status (10Hz)
    if (tx_counter % 1 == 0) {
        BMS_SendStatus();
    }
    
    // 200ms: Cell Voltages (5Hz)
    if (tx_counter % 2 == 0) {
        BMS_SendCellVoltages();
    }
    
    // 500ms: Pack Info, Temperature (2Hz)
    if (tx_counter % 5 == 0) {
        BMS_SendPackInfo();
        BMS_SendTemperature();
    }
    
    // 1000ms: SOC/SOH (1Hz)
    if (tx_counter % 10 == 0) {
        BMS_SendSocSoh();
    }
    
    // 폴트 발생 시 즉시
    if (g_bms.fault_pending) {
        BMS_SendFault();
        g_bms.fault_pending = false;
    }
}
```

### 상태 메시지 송신

```c
void BMS_SendStatus(void) {
    can_msg_bms_status_t msg = {0};
    
    msg.state = g_bms.state;
    msg.soc = g_bms.soc;
    msg.soh = g_bms.soh;
    msg.charge_enable = g_bms.charge_allowed && !g_bms.fault;
    msg.discharge_enable = g_bms.discharge_allowed && !g_bms.fault;
    msg.balancing = g_bms.balancing_active;
    
    // 플래그 설정
    if (g_bms.state == BMS_STATE_CHARGING) msg.flags |= FLAG_CHARGING;
    if (g_bms.state == BMS_STATE_DISCHARGING) msg.flags |= FLAG_DISCHARGING;
    if (g_bms.balancing_active) msg.flags |= FLAG_BALANCING;
    if (g_bms.fault) msg.flags |= FLAG_FAULT;
    
    BMS_CAN_Transmit(CAN_ID_BMS_STATUS, (uint8_t *)&msg, sizeof(msg));
}
```

### 셀 전압 메시지 송신

```c
void BMS_SendCellVoltages(void) {
    can_msg_cell_volt_t msg;
    
    for (int i = 0; i < 6; i++) {  // 6개 메시지
        msg.cell_1 = g_bms.cell_voltage[i * 4 + 0];
        msg.cell_2 = g_bms.cell_voltage[i * 4 + 1];
        msg.cell_3 = g_bms.cell_voltage[i * 4 + 2];
        msg.cell_4 = g_bms.cell_voltage[i * 4 + 3];
        
        BMS_CAN_Transmit(CAN_ID_CELL_VOLT_1 + i, (uint8_t *)&msg, sizeof(msg));
        
        // 메시지 간 간격 (버스 과부하 방지)
        HAL_Delay(1);
    }
}
```

## DBC 파일 생성

CANdb++ 등에서 사용할 DBC 파일:

```
VERSION ""

NS_ :

BS_:

BU_: BMS CHARGER INVERTER DISPLAY

BO_ 256 BMS_Status: 8 BMS
 SG_ State : 0|8@1+ (1,0) [0|15] "" Vector__XXX
 SG_ Flags : 8|8@1+ (1,0) [0|255] "" Vector__XXX
 SG_ SOC : 16|8@1+ (1,0) [0|100] "%" Vector__XXX
 SG_ SOH : 24|8@1+ (1,0) [0|100] "%" Vector__XXX
 SG_ ChargeEnable : 32|8@1+ (1,0) [0|1] "" Vector__XXX
 SG_ DischargeEnable : 40|8@1+ (1,0) [0|1] "" Vector__XXX
 SG_ Balancing : 48|8@1+ (1,0) [0|1] "" Vector__XXX

BO_ 257 BMS_Fault: 8 BMS
 SG_ FaultCode : 0|16@1+ (1,0) [0|65535] "" Vector__XXX
 SG_ FaultCell : 16|8@1+ (1,0) [0|24] "" Vector__XXX
 SG_ FaultDevice : 24|8@1+ (1,0) [0|3] "" Vector__XXX
 SG_ FaultValue : 32|16@1+ (1,0) [0|65535] "mV" Vector__XXX
 SG_ Threshold : 48|16@1+ (1,0) [0|65535] "mV" Vector__XXX

BO_ 512 BMS_CellVolt_1: 8 BMS
 SG_ Cell_1 : 0|16@1+ (1,0) [0|5000] "mV" Vector__XXX
 SG_ Cell_2 : 16|16@1+ (1,0) [0|5000] "mV" Vector__XXX
 SG_ Cell_3 : 32|16@1+ (1,0) [0|5000] "mV" Vector__XXX
 SG_ Cell_4 : 48|16@1+ (1,0) [0|5000] "mV" Vector__XXX

BO_ 528 BMS_PackInfo: 8 BMS
 SG_ PackVoltage : 0|16@1+ (0.1,0) [0|1000] "V" Vector__XXX
 SG_ PackCurrent : 16|16@1- (0.1,0) [-500|500] "A" Vector__XXX
 SG_ MaxCell : 32|16@1+ (1,0) [0|5000] "mV" Vector__XXX
 SG_ MinCell : 48|16@1+ (1,0) [0|5000] "mV" Vector__XXX

BO_ 1024 BMS_Command: 8 Vector__XXX
 SG_ Command : 0|8@1+ (1,0) [0|255] "" BMS

CM_ SG_ 256 State "0=INIT, 1=IDLE, 2=CHARGING, 3=DISCHARGING, 4=BALANCING, 5~9=FAULT";
CM_ SG_ 257 FaultCode "0x0001=CellOV, 0x0002=CellUV, 0x0010=OverTemp";

BA_DEF_ BO_ "GenMsgCycleTime" INT 0 10000;
BA_ "GenMsgCycleTime" BO_ 256 100;
BA_ "GenMsgCycleTime" BO_ 512 200;
BA_ "GenMsgCycleTime" BO_ 528 500;
```

## 삽질: 메시지 순서

셀 전압 6개를 한 번에 보내면 버스 과부하:

```c
// 잘못된 코드
for (int i = 0; i < 6; i++) {
    BMS_CAN_Transmit(...);  // 연속 송신 → TX 버퍼 오버플로
}

// 올바른 코드
for (int i = 0; i < 6; i++) {
    while (HAL_CAN_GetTxMailboxesFreeLevel(&hcan) == 0);  // 대기
    BMS_CAN_Transmit(...);
}
```

## 삽질: Endianness

CAN은 보통 **Big Endian**, STM32는 **Little Endian**:

```c
// Little Endian (STM32)
uint16_t voltage = 3300;  // 0x0CE4
// 메모리: E4 0C

// Big Endian (CAN 일반)
// 메모리: 0C E4

// 변환 필요시
msg.voltage = __REV16(voltage);  // 바이트 스왑
```

**하지만**: Intel 표준은 Little Endian이므로 DBC에서 `@1+`로 지정하면 변환 불필요!

## 정리

| 메시지 | ID | 주기 | 내용 |
|--------|-----|------|------|
| Status | 0x100 | 100ms | 상태, SOC, 플래그 |
| Fault | 0x101 | 이벤트 | 폴트 코드, 위치 |
| Cell Volt | 0x200~205 | 200ms | 셀 전압 (4셀씩) |
| Pack Info | 0x210 | 500ms | 팩 전압/전류 |
| Temperature | 0x220 | 500ms | 온도 정보 |
| Command | 0x400 | 수신 | 제어 명령 |

**다음 글에서**: SOC 추정 기초 - 쿨롱 카운팅과 OCV 테이블.

---

## 시리즈 네비게이션

**Part 6: 통신 & 진단편**
- **#17 - CAN 통신 프로토콜 설계** ← 현재 글
- #18 - SOC 추정 기초
- #19 - 데이터 로깅
- #20 - 진단 인터페이스

---

## 참고 자료

- [CAN Specification 2.0](http://esd.cs.ucr.edu/webres/can20.pdf)
- [DBC File Format](https://www.csselectronics.com/pages/can-dbc-file-database-intro)
- [STM32 bxCAN Reference](https://www.st.com/resource/en/reference_manual/rm0008.pdf)
