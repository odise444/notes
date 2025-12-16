---
title: "AD7280A BMS 개발기 #16 - 보호 로직 통합"
date: 2024-12-13
draft: false
tags: ["AD7280A", "BMS", "STM32", "보호", "상태머신"]
categories: ["BMS 개발"]
series: ["AD7280A BMS 개발"]
summary: "과전압, 저전압, 과온, 과전류. 여러 보호 조건을 상태 머신으로 통합한다."
---

BMS 보호 조건이 여러 개다.

- 셀 과전압 (>3.65V)
- 셀 저전압 (<2.5V)
- 팩 과온 (>60°C)
- 팩 저온 (<-20°C)
- 충전 과전류 (>50A)
- 방전 과전류 (>100A)

이걸 상태 머신으로 관리한다.

---

상태 정의:

```c
typedef enum {
    BMS_STATE_INIT,
    BMS_STATE_IDLE,
    BMS_STATE_CHARGING,
    BMS_STATE_DISCHARGING,
    BMS_STATE_FAULT_OV,
    BMS_STATE_FAULT_UV,
    BMS_STATE_FAULT_OT,
    BMS_STATE_FAULT_OC,
} BMS_State_t;
```

---

상태 전이 로직:

```c
void BMS_UpdateState(void) {
    switch (bms_state) {
    case BMS_STATE_IDLE:
        if (CheckOverVoltage()) {
            bms_state = BMS_STATE_FAULT_OV;
            OpenMainContactor();
        }
        else if (CheckUnderVoltage()) {
            bms_state = BMS_STATE_FAULT_UV;
            OpenMainContactor();
        }
        else if (CheckOverTemp()) {
            bms_state = BMS_STATE_FAULT_OT;
            OpenMainContactor();
        }
        else if (pack_current > 0.5f) {
            bms_state = BMS_STATE_CHARGING;
        }
        else if (pack_current < -0.5f) {
            bms_state = BMS_STATE_DISCHARGING;
        }
        break;
        
    case BMS_STATE_FAULT_OV:
        // 전압이 정상화되고 일정 시간 지나면 복귀
        if (!CheckOverVoltage() && fault_timer > 30000) {
            bms_state = BMS_STATE_IDLE;
            CloseMainContactor();
        }
        break;
    // ...
    }
}
```

---

폴트 복구 조건도 중요하다. 바로 복구하면 ON/OFF 반복될 수 있어서 타이머를 둔다.

```c
#define FAULT_RECOVERY_TIME_MS  30000  // 30초

if (fault_cleared && (now - fault_time) > FAULT_RECOVERY_TIME_MS) {
    // 복구 허용
}
```

---

다음 글에서 CAN 통신을 다룬다.

[#17 - CAN 통신](/posts/bms/ad7280a-bms-dev-17/)
