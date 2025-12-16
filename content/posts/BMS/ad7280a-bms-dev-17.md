---
title: "AD7280A BMS 개발기 #17 - CAN 통신 설계"
date: 2024-12-13
draft: false
tags: ["AD7280A", "BMS", "STM32", "CAN"]
categories: ["BMS 개발"]
series: ["AD7280A BMS 개발"]
summary: "외부 장치랑 통신하려면 CAN이 필수. 메시지 ID 설계가 중요하다."
---

BMS 데이터를 외부로 보내려면 CAN 통신을 쓴다. 자동차나 산업용 장비에서 표준이다.

---

메시지 ID 설계:

```c
// Base ID: 0x100
#define CAN_ID_BMS_STATUS      0x100  // 상태 정보
#define CAN_ID_CELL_V_1_4      0x101  // Cell 1~4 전압
#define CAN_ID_CELL_V_5_8      0x102  // Cell 5~8 전압
#define CAN_ID_CELL_V_9_12     0x103  // Cell 9~12 전압
#define CAN_ID_CELL_V_13_16    0x104  // Cell 13~16 전압
#define CAN_ID_CELL_V_17_20    0x105  // Cell 17~20 전압
#define CAN_ID_CELL_V_21_24    0x106  // Cell 21~24 전압
#define CAN_ID_TEMPERATURE     0x107  // 온도
#define CAN_ID_CURRENT_SOC     0x108  // 전류, SOC
```

---

메시지 구조 예시:

```c
// 0x100 BMS Status
// Byte 0: State
// Byte 1: Fault code
// Byte 2-3: Pack voltage (0.1V)
// Byte 4-5: Pack current (0.1A, signed)
// Byte 6: SOC (%)
// Byte 7: Reserved

void CAN_SendStatus(void) {
    uint8_t data[8];
    data[0] = bms_state;
    data[1] = fault_code;
    data[2] = (pack_voltage_mv / 100) >> 8;
    data[3] = (pack_voltage_mv / 100) & 0xFF;
    data[4] = ((int16_t)(pack_current * 10)) >> 8;
    data[5] = ((int16_t)(pack_current * 10)) & 0xFF;
    data[6] = soc_percent;
    data[7] = 0;
    
    CAN_Transmit(CAN_ID_BMS_STATUS, data, 8);
}
```

---

셀 전압은 8바이트에 4개씩 담는다.

```c
// Cell voltage: 16bit each, 0~65535mV
void CAN_SendCellVoltages(void) {
    uint8_t data[8];
    
    for (int msg = 0; msg < 6; msg++) {
        for (int i = 0; i < 4; i++) {
            int cell = msg * 4 + i;
            data[i*2] = cell_mv[cell] >> 8;
            data[i*2+1] = cell_mv[cell] & 0xFF;
        }
        CAN_Transmit(CAN_ID_CELL_V_1_4 + msg, data, 8);
    }
}
```

---

처음에 CAN이 안 됐다. 종단저항을 한쪽만 달았더니 통신 에러가 계속 났다. CAN 버스 양 끝에 120Ω씩 달아야 한다.

---

다음 글에서 SOC 추정을 다룬다.

[#18 - SOC 추정](/posts/bms/ad7280a-bms-dev-18/)
