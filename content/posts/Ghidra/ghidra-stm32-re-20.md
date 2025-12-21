---
title: "Ghidra STM32 역분석 #20 - 앱 유효성 검사"
date: 2024-12-14
draft: false
tags: ["Ghidra", "STM32", "리버스엔지니어링", "역분석", "부트로더"]
categories: ["역분석"]
series: ["Ghidra 역분석"]
summary: "앱이 제대로 들어있는지 어떻게 확인할까?"
---

펌웨어 업로드 후 앱이 정상인지 확인해야 한다. 깨진 펌웨어로 점프하면 HardFault다.

---

## Stack Pointer 검증

가장 기본적인 체크:

```c
uint8_t Validate_App(void) {
    uint32_t app_sp = *(uint32_t *)APP_START_ADDR;
    
    // SP가 RAM 범위인지
    if (app_sp < 0x20000000 || app_sp > 0x20010000) {
        return 0;  // 유효하지 않음
    }
```

Vector Table 첫 번째 엔트리는 Initial Stack Pointer다. RAM 범위(0x20000000 ~ 0x2000FFFF)가 아니면 펌웨어가 없거나 깨진 거다.

---

## Reset Handler 검증

```c
    uint32_t app_reset = *(uint32_t *)(APP_START_ADDR + 4);
    
    // Reset Handler가 Flash 범위인지
    if (app_reset < APP_START_ADDR || app_reset > 0x08080000) {
        return 0;
    }
    
    // Thumb 비트 체크
    if ((app_reset & 0x01) == 0) {
        return 0;  // ARM 모드면 안 됨
    }
```

두 번째 엔트리는 Reset_Handler 주소. 앱 영역 내에 있어야 하고, Cortex-M은 Thumb 모드라서 LSB가 1이어야 한다.

---

## CRC 검증

```c
    // 앱 CRC 계산
    uint32_t calc_crc = CRC32_Calc(APP_START_ADDR, g_app_size);
    uint32_t stored_crc = *(uint32_t *)(APP_START_ADDR + g_app_size);
    
    if (calc_crc != stored_crc) {
        return 0;
    }
    
    return 1;  // 유효함
}
```

업로드할 때 저장해둔 CRC와 비교한다. STM32 하드웨어 CRC 사용.

---

## 검증 순서

```
1. SP 범위 체크     → 빠름, 기본
2. Reset 주소 체크  → 빠름, 기본
3. CRC 계산        → 느림, 정확
```

SP랑 Reset 주소만 봐도 빈 Flash(0xFFFFFFFF)는 걸러진다. CRC는 무결성까지 확인.

---

다음 글에서 앱으로 점프하는 코드 분석.

[#21 - 앱 점프 코드](/posts/ghidra/ghidra-stm32-re-21/)
