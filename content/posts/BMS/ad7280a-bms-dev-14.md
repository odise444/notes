---
title: "AD7280A BMS 개발기 #14 - Alert 핀 인터럽트"
date: 2024-12-13
draft: false
tags: ["AD7280A", "BMS", "STM32", "인터럽트", "EXTI"]
categories: ["BMS 개발"]
series: ["AD7280A BMS 개발"]
summary: "Alert 핀이 Open-drain이라 풀업이 필요하다. 처음에 몰라서 한참 헤맸다."
---

AD7280A ALERT 핀은 알람 발생 시 Low로 떨어진다. 이걸 STM32 외부 인터럽트로 받는다.

---

연결 구성:

```
AD7280A ALERT (Open-drain)
       │
       ├── 10kΩ 풀업 ── 3.3V
       │
       └── STM32 GPIO (EXTI)
```

처음에 풀업 없이 직결했더니 인터럽트가 안 들어왔다. Open-drain은 High를 못 만들어서 풀업이 필수다.

---

STM32 설정:

```c
// GPIO 설정
GPIO_InitTypeDef GPIO_InitStruct = {0};
GPIO_InitStruct.Pin = ALERT_PIN;
GPIO_InitStruct.Mode = GPIO_MODE_IT_FALLING;  // 하강 에지
GPIO_InitStruct.Pull = GPIO_PULLUP;           // 내부 풀업도 가능
HAL_GPIO_Init(ALERT_GPIO_PORT, &GPIO_InitStruct);

// NVIC 설정
HAL_NVIC_SetPriority(EXTI_IRQn, 1, 0);
HAL_NVIC_EnableIRQ(EXTI_IRQn);
```

내부 풀업을 쓸 수도 있는데, 저항값이 좀 약해서(~40kΩ) 노이즈에 민감할 수 있다. 외부 10kΩ이 더 안정적.

---

데이지체인에서 ALERT 핀은 어떻게 되냐면, 모든 IC의 ALERT가 OR로 묶여있다. 

어느 IC에서든 알람이 발생하면 하나의 ALERT 라인이 Low가 된다. 그래서 인터럽트는 하나만 받고, 어떤 IC인지는 레지스터 읽어서 확인해야 한다.

```c
void HAL_GPIO_EXTI_Callback(uint16_t GPIO_Pin) {
    if (GPIO_Pin == ALERT_PIN) {
        // 어떤 디바이스에서 알람인지 확인
        for (int dev = 0; dev < 4; dev++) {
            uint8_t alert = AD7280A_ReadReg(dev, REG_ALERT);
            if (alert) {
                HandleAlert(dev, alert);
            }
        }
    }
}
```

---

디바운싱도 고려해야 한다. 노이즈로 오동작할 수 있어서.

```c
void HAL_GPIO_EXTI_Callback(uint16_t GPIO_Pin) {
    static uint32_t last_time = 0;
    uint32_t now = HAL_GetTick();
    
    if (now - last_time < 10) return;  // 10ms 디바운싱
    last_time = now;
    
    // 알람 처리...
}
```

---

다음 글에서 알람 상태 읽기와 비트 해석을 다룬다.

[#15 - 알람 상태 읽기](/posts/bms/ad7280a-bms-dev-15/)
