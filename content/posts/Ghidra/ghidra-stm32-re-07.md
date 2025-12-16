---
title: "Ghidra STM32 역분석 #7 - 부트로더 경계 찾기"
date: 2024-12-10
draft: false
tags: ["Ghidra", "STM32", "리버스엔지니어링", "역분석", "부트로더"]
categories: ["역분석"]
series: ["Ghidra 역분석"]
summary: "부트로더는 어디서 끝나고 앱은 어디서 시작하나?"
---

IAP 부트로더는 Flash를 영역별로 나눠서 사용한다:

```
┌─────────────────┐ 0x08080000
│   Application   │
├─────────────────┤ 0x08042800
│     Buffer      │
├─────────────────┤ 0x08004000
│   Bootloader    │
└─────────────────┘ 0x08000000
```

---

## HEX 파일로 추정

```bash
ls -la 200429.bin
# 16384 bytes = 16KB = 0x4000
```

부트로더: `0x08000000` ~ `0x08003FFF` (16KB)

---

## 앱 점프 주소 찾기

부트로더는 결국 앱으로 점프한다. Ghidra에서 Search → For Scalars → `0x08004000` 또는 `0x08042800` 검색.

```c
// 전형적인 점프 코드
void jump_to_app(void) {
    uint32_t stack = *(uint32_t *)0x08042800;
    uint32_t reset = *(uint32_t *)0x08042804;
    __set_MSP(stack);
    ((void(*)(void))reset)();
}
```

`0x08042800`이 앱 시작 주소다. 버퍼 영역이 `0x08004000` ~ `0x08042800`이고 거기에 펌웨어를 받아서 검증 후 복사하는 구조.

---

## Flash 레이아웃

| 영역 | 시작 | 크기 |
|------|------|------|
| Bootloader | 0x08000000 | 16KB |
| Buffer | 0x08004000 | 254KB |
| Application | 0x08042800 | 246KB |

---

다음 글에서 CAN 초기화 코드 분석.

[#8 - CAN 초기화 분석](/posts/ghidra/ghidra-stm32-re-08/)
