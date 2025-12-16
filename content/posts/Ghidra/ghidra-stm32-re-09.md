---
title: "Ghidra STM32 역분석 #9 - RCC 클럭 설정 분석"
date: 2024-12-12
draft: false
tags: ["Ghidra", "STM32", "리버스엔지니어링", "역분석", "RCC", "클럭"]
categories: ["역분석"]
series: ["Ghidra 역분석"]
summary: "MCU의 심장, 클럭 설정을 역분석. PLL 설정값을 찾아라."
---

모든 주변장치는 클럭이 있어야 동작한다. RCC 설정을 모르면 CAN 비트레이트도 UART 보레이트도 계산 못 한다.

---

## RCC 레지스터

| 레지스터 | 주소 | 용도 |
|----------|------|------|
| RCC_CR | 0x40021000 | HSE/PLL ON |
| RCC_CFGR | 0x40021004 | PLL 배수 설정 |
| RCC_APB2ENR | 0x40021018 | GPIO, SPI 클럭 |
| RCC_APB1ENR | 0x4002101C | CAN, UART 클럭 |

---

## Ghidra에서 찾기

Search → For Scalars → `0x40021000`

찾은 함수:

```c
void RCC_Init(void) {
    *(uint32_t *)0x40021000 |= 0x10000;   // HSEON
    while ((*(uint32_t *)0x40021000 & 0x20000) == 0);  // 대기
    
    *(uint32_t *)0x40021004 = 0x001D0402; // PLL 설정
    *(uint32_t *)0x40021000 |= 0x1000000; // PLLON
    while ((*(uint32_t *)0x40021000 & 0x2000000) == 0);
    
    *(uint32_t *)0x40021004 |= 0x02;      // PLL을 SYSCLK으로
}
```

---

## 비트 해석

`0x001D0402` 분석:

```
PLLMUL = 0x1C0000 >> 18 = 7 → ×9
PLLSRC = 0x10000 → HSE
PPRE1 = 0x400 → /2
```

HSE(8MHz) × 9 = 72MHz. APB1은 36MHz.

---

## 주변장치 클럭

```c
*(uint32_t *)0x40021018 |= 0x04;  // GPIOA 클럭
*(uint32_t *)0x4002101C |= 0x02000000; // CAN1 클럭
```

APB2ENR 비트 2 = GPIOA, APB1ENR 비트 25 = CAN1.

---

다음 글에서 CAN 초기화 분석.

[#10 - CAN 초기화 분석](/posts/ghidra/ghidra-stm32-re-10/)
