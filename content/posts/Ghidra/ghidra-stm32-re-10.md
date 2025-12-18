---
title: "Ghidra STM32 역분석 #10 - GPIO 설정 분석"
date: 2024-12-12
draft: false
tags: ["Ghidra", "STM32", "리버스엔지니어링", "역분석", "GPIO"]
categories: ["역분석"]
series: ["Ghidra 역분석"]
summary: "GPIO 설정을 역분석해서 핀맵을 복원하자."
---

## GPIO 레지스터

| 포트 | 베이스 주소 |
|------|-------------|
| GPIOA | 0x40010800 |
| GPIOB | 0x40010C00 |
| GPIOC | 0x40011000 |

CRL은 Pin 0~7, CRH는 Pin 8~15. 핀당 4비트.

---

## 설정값 해석

각 핀 4비트 = CNF[1:0] + MODE[1:0]

```
0x4 = 0100 = Input Pull-up/down
0xB = 1011 = AF Open-Drain 50MHz
0x3 = 0011 = Output Push-Pull 50MHz
```

---

## Ghidra에서 찾기

Search → For Scalars → `0x40010800`

```c
void GPIO_Init(void) {
    *(uint32_t *)0x40010804 = 0x444444B4;  // GPIOA CRH
    *(uint32_t *)0x40010C04 = 0x44444444;  // GPIOB CRH
}
```

`0x444444B4` 분석:
- PA8: 0x4 = Input Pull-up
- PA9: 0xB = AF Open-Drain (CAN TX)
- PA10~15: 0x4 = Input

PA9가 CAN TX 핀이다.

---

## CAN 핀 확인

STM32F103 CAN 리맵:

| 리맵 | CAN_RX | CAN_TX |
|------|--------|--------|
| 없음 | PA11 | PA12 |
| 부분 | PB8 | PB9 |
| 전체 | PD0 | PD1 |

GPIO 설정 보면 어떤 핀 쓰는지 알 수 있다.

---

다음 글에서 CAN 레지스터 분석.

[#11 - CAN 레지스터 분석](/posts/ghidra/ghidra-stm32-re-11/)
