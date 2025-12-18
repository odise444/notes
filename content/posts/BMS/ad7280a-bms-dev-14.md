---
title: "AD7280A BMS 개발기 #14 - Alert 핀 인터럽트 처리"
date: 2024-01-28
draft: false
tags: ["AD7280A", "BMS", "STM32", "인터럽트", "Alert"]
categories: ["BMS 개발"]
series: ["AD7280A BMS 개발"]
summary: "Alert 핀이 떨어지면 뭔가 문제가 생긴 거다. 인터럽트로 빠르게 잡자."
---

임계값 설정했으니 이제 Alert 핀 처리. 알람 발생하면 Alert 핀이 Low로 떨어진다.

---

## Alert 핀 특성

- Open Drain 출력
- Active Low
- 여러 디바이스 Alert를 Wired-OR로 연결 가능

외부에 풀업 저항 필요:

```
VDD (3.3V)
    │
   [10kΩ]
    │
    ├──── MCU GPIO (PB0)
    │
    └──── AD7280A Alert (모든 디바이스)
```

4개 디바이스 Alert를 한 핀에 연결. 어느 하나라도 알람 나면 Low.

---

## STM32 EXTI 설정

CubeMX에서:
- PB0: GPIO_EXTI0
- Mode: External Interrupt Mode with Falling edge trigger
- Pull-up: Enable

```c
// EXTI 콜백
void HAL_GPIO_EXTI_Callback(uint16_t GPIO_Pin) {
    if (GPIO_Pin == ALERT_PIN) {
        g_alert_flag = true;
    }
}
```

---

## 디바운싱

노이즈 때문에 가짜 인터럽트가 뜰 수 있다. 디바운싱 필요.

```c
void HAL_GPIO_EXTI_Callback(uint16_t GPIO_Pin) {
    if (GPIO_Pin == ALERT_PIN) {
        // 실제 Low인지 다시 확인
        if (HAL_GPIO_ReadPin(ALERT_GPIO, ALERT_PIN) == GPIO_PIN_RESET) {
            g_alert_flag = true;
        }
    }
}
```

ISR에서는 플래그만 세우고, 메인 루프에서 처리:

```c
void BMS_Task(void) {
    if (g_alert_flag) {
        g_alert_flag = false;
        
        // 50ms 후에도 Low면 진짜 알람
        HAL_Delay(50);
        if (HAL_GPIO_ReadPin(ALERT_GPIO, ALERT_PIN) == GPIO_PIN_RESET) {
            BMS_HandleAlert();
        }
    }
}
```

---

## 어떤 디바이스인지 확인

Alert가 떴는데 4개 중 어디서 난 건지 알아야 한다.

각 디바이스의 Alert 레지스터 읽기:

```c
void BMS_HandleAlert(void) {
    for (int dev = 0; dev < 4; dev++) {
        uint8_t alert = AD7280A_ReadRegister(dev, AD7280A_ALERT);
        
        if (alert & 0x3F) {  // 셀 알람 비트
            printf("Dev %d Alert: 0x%02X\n", dev, alert);
            BMS_ProcessAlert(dev, alert);
        }
    }
    
    // Alert 클리어
    AD7280A_Write(0x1F, AD7280A_ALERT, 0x00);
}
```

---

## Alert 비트 해석

Alert 레지스터(0x18) 비트맵:

```
Bit 5: Cell 6 알람
Bit 4: Cell 5 알람
Bit 3: Cell 4 알람
Bit 2: Cell 3 알람
Bit 1: Cell 2 알람
Bit 0: Cell 1 알람
```

OV인지 UV인지는 실제 전압 읽어서 판단:

```c
void BMS_ProcessAlert(uint8_t dev, uint8_t alert) {
    for (int cell = 0; cell < 6; cell++) {
        if (alert & (1 << cell)) {
            uint16_t mv = AD7280A_ReadCellMillivolt(dev, cell);
            
            if (mv > OV_THRESHOLD_MV) {
                BMS_SetFault(FAULT_OVERVOLTAGE, dev * 6 + cell);
            } else if (mv < UV_THRESHOLD_MV) {
                BMS_SetFault(FAULT_UNDERVOLTAGE, dev * 6 + cell);
            }
        }
    }
}
```

---

## 문제: Alert가 안 클리어됨

처음에 Alert 클리어가 안 됐다. 계속 Low 상태.

이유: 알람 조건이 해제 안 됐다. OV 상태에서 Alert 클리어해봤자 바로 다시 뜬다.

```c
void BMS_HandleAlert(void) {
    // 알람 원인 확인 및 조치
    BMS_ProcessAlert();
    
    // 조치 후에도 조건 유지되면 클리어 안 함
    if (BMS_IsAlarmConditionCleared()) {
        AD7280A_Write(0x1F, AD7280A_ALERT, 0x00);
    }
}
```

---

## 정리

- Alert: Open Drain, Active Low
- Wired-OR로 4개 연결 가능
- EXTI 인터럽트로 빠르게 감지
- 디바운싱 필요
- 클리어 전에 조건 해제 확인

---

다음은 Alert 레지스터 상세 분석.

[#15 - 알람 상태 읽기](/posts/bms/ad7280a-bms-dev-15/)
