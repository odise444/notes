---
title: "AD7280A BMS 개발기 #16 - 보호 로직 통합"
date: 2024-01-30
draft: false
tags: ["AD7280A", "BMS", "STM32", "보호", "FET"]
categories: ["BMS 개발"]
series: ["AD7280A BMS 개발"]
summary: "알람 감지까지 됐다. 이제 실제로 배터리를 보호해야 한다. FET 제어."
---

알람 감지는 된다. 이제 실제로 출력 차단해서 배터리를 보호해야 한다.

---

## 보호 항목

| 항목 | 조건 | 동작 |
|------|------|------|
| 과전압 | 셀 > 3.7V | 충전 차단 |
| 저전압 | 셀 < 2.4V | 방전 차단 |
| 과열 | 온도 > 60°C | 전체 차단 |
| 저온 | 온도 < 0°C | 충전 차단 |
| 과전류 | 전류 > 100A | 전체 차단 |

---

## FET 제어

충전/방전 FET 분리:

```
Battery+ ──[충전FET]──┬──[방전FET]── Load+
                      │
                   [션트]
                      │
Battery- ─────────────┴──────────── Load-
```

GPIO로 FET 제어:

```c
#define CHG_FET_PIN   GPIO_PIN_8   // PA8
#define DSG_FET_PIN   GPIO_PIN_9   // PA9

void BMS_EnableCharging(bool enable) {
    HAL_GPIO_WritePin(GPIOA, CHG_FET_PIN, 
        enable ? GPIO_PIN_SET : GPIO_PIN_RESET);
}

void BMS_EnableDischarging(bool enable) {
    HAL_GPIO_WritePin(GPIOA, DSG_FET_PIN,
        enable ? GPIO_PIN_SET : GPIO_PIN_RESET);
}
```

---

## 상태 머신

```c
typedef enum {
    STATE_NORMAL,       // 정상
    STATE_OV_FAULT,     // 과전압
    STATE_UV_FAULT,     // 저전압
    STATE_OT_FAULT,     // 과열
    STATE_OC_FAULT,     // 과전류
    STATE_SHUTDOWN      // 셧다운
} BMS_State_t;
```

상태별 FET 제어:

```c
void BMS_UpdateProtection(void) {
    switch (g_bms.state) {
        case STATE_NORMAL:
            BMS_EnableCharging(true);
            BMS_EnableDischarging(true);
            break;
            
        case STATE_OV_FAULT:
            BMS_EnableCharging(false);   // 충전만 차단
            BMS_EnableDischarging(true);
            break;
            
        case STATE_UV_FAULT:
            BMS_EnableCharging(true);
            BMS_EnableDischarging(false); // 방전만 차단
            break;
            
        case STATE_OT_FAULT:
        case STATE_OC_FAULT:
        case STATE_SHUTDOWN:
            BMS_EnableCharging(false);
            BMS_EnableDischarging(false); // 전체 차단
            break;
    }
}
```

---

## 폴트 진입

알람 발생 시 상태 전이:

```c
void BMS_CheckFaults(void) {
    // 과전압 체크
    if (BMS_GetMaxCellVoltage() > OV_THRESHOLD_MV) {
        g_bms.state = STATE_OV_FAULT;
        g_bms.fault_cell = BMS_GetMaxCellIndex();
        return;
    }
    
    // 저전압 체크
    if (BMS_GetMinCellVoltage() < UV_THRESHOLD_MV) {
        g_bms.state = STATE_UV_FAULT;
        g_bms.fault_cell = BMS_GetMinCellIndex();
        return;
    }
    
    // 과열 체크
    if (BMS_GetMaxTemperature() > OT_THRESHOLD) {
        g_bms.state = STATE_OT_FAULT;
        return;
    }
    
    // 정상
    if (g_bms.state != STATE_NORMAL) {
        // 복구 조건 확인
        BMS_TryRecovery();
    }
}
```

---

## 폴트 복구

조건 해제되면 자동 복구? 수동 리셋?

나는 자동 복구 + 히스테리시스로 했다:

```c
void BMS_TryRecovery(void) {
    switch (g_bms.state) {
        case STATE_OV_FAULT:
            // OV 임계값보다 100mV 낮아지면 복구
            if (BMS_GetMaxCellVoltage() < OV_THRESHOLD_MV - 100) {
                g_bms.state = STATE_NORMAL;
            }
            break;
            
        case STATE_UV_FAULT:
            // UV 임계값보다 200mV 높아지면 복구
            if (BMS_GetMinCellVoltage() > UV_THRESHOLD_MV + 200) {
                g_bms.state = STATE_NORMAL;
            }
            break;
            
        case STATE_OT_FAULT:
            // 10도 낮아지면 복구
            if (BMS_GetMaxTemperature() < OT_THRESHOLD - 100) {
                g_bms.state = STATE_NORMAL;
            }
            break;
    }
}
```

---

## 빠른 차단

과전류 같은 건 빨리 차단해야 한다. 폴링으로는 느리다.

하드웨어 비교기 + 인터럽트:

```c
// COMP 인터럽트
void HAL_COMP_TriggerCallback(COMP_HandleTypeDef *hcomp) {
    // 즉시 차단
    HAL_GPIO_WritePin(GPIOA, CHG_FET_PIN, GPIO_PIN_RESET);
    HAL_GPIO_WritePin(GPIOA, DSG_FET_PIN, GPIO_PIN_RESET);
    
    g_bms.state = STATE_OC_FAULT;
}
```

소프트웨어보다 하드웨어가 빠르다.

---

## 정리

- 충전/방전 FET 분리 제어
- 상태 머신으로 보호 로직
- 히스테리시스로 자동 복구
- 과전류는 하드웨어 비교기

---

Part 5 알람 & 보호편 끝.

다음은 CAN 통신.

[#17 - CAN 통신](/posts/bms/ad7280a-bms-dev-17/)
