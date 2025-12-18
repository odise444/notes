---
title: "AD7280A BMS 개발기 #17 - CAN 통신 프로토콜"
date: 2024-01-31
draft: false
tags: ["AD7280A", "BMS", "STM32", "CAN", "프로토콜"]
categories: ["BMS 개발"]
series: ["AD7280A BMS 개발"]
summary: "BMS 데이터를 외부로 보내야 한다. CAN으로 인버터랑 통신하기."
---

BMS 혼자 동작하면 의미 없다. 인버터, 충전기, 디스플레이랑 통신해야 한다.

자동차/산업용은 보통 CAN 쓴다.

---

## CAN 기본 설정

STM32F103은 CAN 컨트롤러 내장.

CubeMX 설정:
- Prescaler: 4
- Time Seg1: 13
- Time Seg2: 2
- Baud Rate: 500kbps

```c
CAN_HandleTypeDef hcan;

void CAN_Init(void) {
    hcan.Instance = CAN1;
    hcan.Init.Prescaler = 4;
    hcan.Init.Mode = CAN_MODE_NORMAL;
    hcan.Init.SyncJumpWidth = CAN_SJW_1TQ;
    hcan.Init.TimeSeg1 = CAN_BS1_13TQ;
    hcan.Init.TimeSeg2 = CAN_BS2_2TQ;
    HAL_CAN_Init(&hcan);
    
    HAL_CAN_Start(&hcan);
}
```

---

## 메시지 ID 설계

내가 정한 ID 체계:

| ID | 내용 | 주기 |
|----|------|------|
| 0x100 | 팩 전압/전류/SOC | 100ms |
| 0x101 | 셀 전압 1~8 | 100ms |
| 0x102 | 셀 전압 9~16 | 100ms |
| 0x103 | 셀 전압 17~24 | 100ms |
| 0x110 | 온도 | 1000ms |
| 0x120 | 알람/상태 | 이벤트 |

---

## 팩 정보 메시지

8바이트에 담을 정보:

```c
// 0x100: 팩 정보
typedef struct __attribute__((packed)) {
    uint16_t pack_voltage;  // 0.1V 단위
    int16_t  pack_current;  // 0.1A 단위, signed
    uint8_t  soc;           // %
    uint8_t  state;         // 상태
    uint8_t  fault;         // 폴트 코드
    uint8_t  reserved;
} CAN_PackInfo_t;

void CAN_SendPackInfo(void) {
    CAN_PackInfo_t msg;
    
    msg.pack_voltage = g_bms.pack_voltage_mv / 100;  // mV → 0.1V
    msg.pack_current = g_bms.pack_current_ma / 100;  // mA → 0.1A
    msg.soc = g_bms.soc;
    msg.state = g_bms.state;
    msg.fault = g_bms.fault_code;
    
    CAN_Send(0x100, (uint8_t*)&msg, sizeof(msg));
}
```

---

## 셀 전압 메시지

24셀을 3개 메시지에 나눠서:

```c
// 0x101: Cell 1~8
void CAN_SendCellVoltages1(void) {
    uint8_t data[8];
    
    // 셀당 16비트, 4셀 = 8바이트
    for (int i = 0; i < 4; i++) {
        uint16_t mv = g_cells_mv[i];
        data[i*2] = mv >> 8;
        data[i*2+1] = mv & 0xFF;
    }
    
    CAN_Send(0x101, data, 8);
}
```

근데 이러면 24셀에 6개 메시지 필요. 너무 많다.

셀당 12비트로 압축하면 8바이트에 5셀:

```c
// 더 효율적인 방식
// 12bit × 5 = 60bit, 8바이트에 5셀
```

복잡해서 그냥 16비트로 했다. 대역폭 여유 있으면 굳이 압축 안 해도 된다.

---

## 송신 코드

```c
HAL_StatusTypeDef CAN_Send(uint32_t id, uint8_t *data, uint8_t len) {
    CAN_TxHeaderTypeDef header;
    uint32_t mailbox;
    
    header.StdId = id;
    header.IDE = CAN_ID_STD;
    header.RTR = CAN_RTR_DATA;
    header.DLC = len;
    
    return HAL_CAN_AddTxMessage(&hcan, &header, data, &mailbox);
}
```

---

## 주기적 송신

타이머로 주기적 송신:

```c
void BMS_CANTask(void) {
    static uint32_t last_100ms = 0;
    static uint32_t last_1s = 0;
    
    uint32_t now = HAL_GetTick();
    
    if (now - last_100ms >= 100) {
        last_100ms = now;
        CAN_SendPackInfo();
        CAN_SendCellVoltages();
    }
    
    if (now - last_1s >= 1000) {
        last_1s = now;
        CAN_SendTemperatures();
    }
}
```

---

## 수신 처리

충전기에서 명령 받기:

```c
// CAN 수신 콜백
void HAL_CAN_RxFifo0MsgPendingCallback(CAN_HandleTypeDef *hcan) {
    CAN_RxHeaderTypeDef header;
    uint8_t data[8];
    
    HAL_CAN_GetRxMessage(hcan, CAN_RX_FIFO0, &header, data);
    
    switch (header.StdId) {
        case 0x200:  // 충전기 명령
            BMS_HandleChargerCommand(data);
            break;
        case 0x300:  // 설정 명령
            BMS_HandleConfigCommand(data);
            break;
    }
}
```

---

## 정리

- 500kbps 기본
- 메시지 ID 체계 설계
- 주기적 송신 + 이벤트 송신
- 필터 설정으로 필요한 ID만 수신

---

다음은 SOC 추정.

[#18 - SOC 추정](/posts/bms/ad7280a-bms-dev-18/)
