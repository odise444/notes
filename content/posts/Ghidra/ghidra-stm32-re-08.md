---
title: "Ghidra STM32 역분석 #8 - 전역 변수 영역 분석"
date: 2024-12-11
draft: false
tags: ["Ghidra", "STM32", "리버스엔지니어링", "역분석", "RAM"]
categories: ["역분석"]
series: ["Ghidra 역분석"]
summary: ".data, .bss가 뭔지 모르면 전역 변수 분석이 안 된다."
---

디컴파일에서 이런 코드가 나온다:

```c
*(uint32_t *)0x20000100 = 0;
*(uint32_t *)0x20000104 = rx_data;
buffer = (uint8_t *)0x20000200;
```

`0x200xxxxx`는 전부 RAM 영역. 전역 변수다.

---

## .data vs .bss

| 섹션 | 내용 | 예시 |
|------|------|------|
| .data | 초기값 있는 변수 | `int count = 100;` |
| .bss | 초기값 없는 변수 | `int buffer[256];` |

---

## 초기화 코드에서 찾기

Reset_Handler에 이런 패턴이 있다:

```c
// .data 초기화 (Flash → RAM 복사)
memcpy(0x20000000, 0x08003800, 0x100);

// .bss 초기화 (0으로 채움)
memset(0x20000100, 0, 0x400);
```

이걸 보면:
- `.data`: 0x20000000 ~ 0x200000FF (256바이트)
- `.bss`: 0x20000100 ~ 0x200004FF (1KB)

---

## Ghidra에서 라벨 지정

자주 접근하는 RAM 주소에 이름 붙이기 (L 키):

```
0x20000100 → g_rx_buffer
0x20000200 → g_can_msg
0x20000300 → g_flash_buffer
```

디컴파일 결과가 훨씬 읽기 쉬워진다.

---

다음 글에서 CAN 초기화 분석.

[#9 - CAN 초기화 분석](/posts/ghidra/ghidra-stm32-re-09/)
