---
title: "AD7280A BMS 개발기 #13 - 과전압/저전압 알람"
date: 2024-12-13
draft: false
tags: ["AD7280A", "BMS", "STM32", "알람", "보호"]
categories: ["BMS 개발"]
series: ["AD7280A BMS 개발"]
summary: "셀 전압이 범위를 벗어나면 알람이 뜬다. 하드웨어 보호 기능."
---

소프트웨어가 죽어도 IC 자체가 과전압/저전압을 감지해서 알람을 띄워준다. 안전을 위한 하드웨어 보호 기능이다.

---

LiFePO4 기준으로 임계값 설정:

```
과전압: 3.65V (충전 상한)
저전압: 2.5V (방전 하한)
```

AD7280A 레지스터 값으로 변환:

```c
// 임계값 = (전압_mV - 1000) / 6
// 과전압: (3650 - 1000) / 6 = 441.6 → 442
// 저전압: (2500 - 1000) / 6 = 250

#define OV_THRESHOLD    442
#define UV_THRESHOLD    250

void AD7280A_SetThresholds(void) {
    AD7280A_WriteAll(REG_CELL_OVERVTG, OV_THRESHOLD);
    AD7280A_WriteAll(REG_CELL_UNDERVTG, UV_THRESHOLD);
}
```

해상도가 6mV라서 정확한 값은 못 맞춘다. 그래서 약간 마진을 둔다.

---

처음에 임계값 계산을 잘못해서 알람이 계속 떴다.

```
잘못된 계산:
OV = 3650 / 6 = 608  (틀림)
UV = 2500 / 6 = 416  (틀림)

→ 정상 전압에서도 알람 발생
```

1000mV 오프셋을 빼야 한다는 걸 놓쳤다.

---

알람이 발생하면 ALERT 핀이 Low로 떨어진다. 이걸 MCU 인터럽트로 받아서 처리한다.

```c
void EXTI_ALERT_Callback(void) {
    // 알람 상태 읽기
    uint8_t alert_a = AD7280A_ReadReg(REG_ALERT_A);
    uint8_t alert_b = AD7280A_ReadReg(REG_ALERT_B);
    uint8_t alert_c = AD7280A_ReadReg(REG_ALERT_C);
    
    if (alert_a & OV_MASK) {
        // 과전압 처리
        BMS_StopCharging();
    }
    if (alert_a & UV_MASK) {
        // 저전압 처리  
        BMS_StopDischarging();
    }
    
    // 알람 클리어
    AD7280A_WriteAll(REG_ALERT, 0xFF);
}
```

---

다음 글에서 Alert 핀 인터럽트 설정을 자세히 다룬다.

[#14 - Alert 인터럽트](/posts/bms/ad7280a-bms-dev-14/)
