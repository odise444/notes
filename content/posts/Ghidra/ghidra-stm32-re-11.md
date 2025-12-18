---
title: "Ghidra STM32 역분석 #11 - CAN 초기화 분석"
date: 2024-12-12
draft: false
tags: ["Ghidra", "STM32", "리버스엔지니어링", "역분석", "CAN"]
categories: ["역분석"]
series: ["Ghidra 역분석"]
summary: "500kbps CAN 설정을 역분석. 비트 타이밍과 필터 구성."
---

IAP 부트로더의 핵심은 CAN 통신. 초기화 코드를 분석하자.

---

## CAN 레지스터

| 레지스터 | 주소 | 용도 |
|----------|------|------|
| CAN_MCR | 0x40006400 | Master Control |
| CAN_BTR | 0x4000641C | Bit Timing |
| CAN_FMR | 0x40006600 | Filter Master |

---

## Ghidra에서 찾기

Search → For Scalars → `0x40006400`

```c
void CAN_Init(void) {
    *(uint32_t *)0x40006400 |= 0x01;       // 초기화 모드
    while ((*(uint32_t *)0x40006404 & 0x01) == 0);
    
    *(uint32_t *)0x4000641C = 0x001C0003;  // BTR
    
    *(uint32_t *)0x40006400 &= ~0x01;      // 정상 모드
}
```

---

## BTR 분석

`0x001C0003` 해석:

```
SJW = 0 → 1 TQ
TS2 = 0x1C >> 20 = 1 → 2 TQ
TS1 = 0x1C >> 16 & 0xF = 0xC → 13 TQ
BRP = 3 → 분주 4

Total = 1 + 13 + 2 = 16 TQ
Baudrate = 36MHz / 4 / 16 = 562.5kbps
```

거의 500kbps. APB1이 36MHz라서 정확히 안 맞는다. 실제로는 HSE 설정 다를 수 있다.

---

## 필터 설정

```c
*(uint32_t *)0x40006640 = 0x00000000;  // ID 필터
*(uint32_t *)0x40006644 = 0x00000000;  // 마스크
```

0으로 설정하면 모든 ID 수신. 부트로더니까 특정 ID만 받도록 설정돼 있을 수도 있다.

---

다음 글에서 CAN 수신 인터럽트 분석.

[#12 - CAN 수신 핸들러](/posts/ghidra/ghidra-stm32-re-12/)
