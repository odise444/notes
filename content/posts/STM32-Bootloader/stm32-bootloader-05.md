---
title: "STM32 부트로더 개발기 #5 - 앱 유효성 검사"
date: 2024-12-17
tags: ["STM32", "Bootloader", "검증", "Stack Pointer"]
categories: ["개발기"]
series: ["STM32 부트로더 개발기"]
summary: "앱 영역에 진짜 펌웨어가 있는지 확인해야 한다."
---

부트로더가 앱으로 점프하기 전에 확인할 게 있다.

앱 영역에 진짜 유효한 펌웨어가 있나?

---

## 왜 검사하나

앱 영역이 비어있으면 (0xFF로 채워져 있으면)?

점프하면 HardFault. 보드 벽돌.

---

## 검사 항목

1. **Stack Pointer 값** - RAM 범위 내인가?
2. **Reset Handler 주소** - Flash 범위 내인가?
3. **Thumb 비트** - LSB가 1인가?
4. **(선택) CRC** - 데이터 무결성

---

## Stack Pointer 검사

앱 시작 주소의 첫 4바이트 = Initial Stack Pointer.

```c
#define APP_ADDR  0x08042800
#define RAM_START 0x20000000
#define RAM_END   0x20010000  // 64KB

uint32_t app_sp = *(volatile uint32_t *)APP_ADDR;

bool is_valid_sp(uint32_t sp) {
    return (sp >= RAM_START) && (sp <= RAM_END);
}
```

SP가 RAM 범위 밖이면 유효하지 않음.

---

## Reset Handler 검사

앱 시작 주소 + 4 = Reset Handler 주소.

```c
uint32_t app_reset = *(volatile uint32_t *)(APP_ADDR + 4);

bool is_valid_reset(uint32_t reset) {
    // Thumb 비트 제거하고 검사
    uint32_t addr = reset & ~1;
    return (addr >= APP_ADDR) && (addr < 0x08080000);
}
```

---

## Thumb 비트

ARM Cortex-M은 Thumb 모드만 지원.

함수 주소의 LSB가 1이어야 함.

```c
bool is_thumb(uint32_t addr) {
    return (addr & 1) == 1;
}
```

Reset Handler 주소의 LSB가 0이면 뭔가 잘못된 거.

---

## 전체 검사 함수

```c
bool is_app_valid(void) {
    uint32_t app_sp = *(volatile uint32_t *)APP_ADDR;
    uint32_t app_reset = *(volatile uint32_t *)(APP_ADDR + 4);
    
    // 1. SP가 RAM 범위 내
    if (app_sp < RAM_START || app_sp > RAM_END) {
        return false;
    }
    
    // 2. Reset Handler가 Flash 범위 내
    uint32_t reset_addr = app_reset & ~1;
    if (reset_addr < APP_ADDR || reset_addr >= 0x08080000) {
        return false;
    }
    
    // 3. Thumb 비트 확인
    if ((app_reset & 1) == 0) {
        return false;
    }
    
    return true;
}
```

---

## Flash가 비어있으면?

지워진 Flash는 0xFF로 채워짐.

```
0x08042800: FF FF FF FF  ← SP = 0xFFFFFFFF (RAM 범위 밖)
0x08042804: FF FF FF FF  ← Reset = 0xFFFFFFFF
```

SP 검사에서 걸림.

---

## CRC 검사 (선택)

더 확실하게 하려면 CRC.

펌웨어 끝에 CRC 값 저장해두고, 부트로더에서 계산해서 비교.

```c
// 펌웨어 빌드 시 CRC를 마지막에 붙임
// 부트로더에서 검증
uint32_t stored_crc = *(volatile uint32_t *)(APP_ADDR + app_size - 4);
uint32_t calc_crc = calculate_crc32((uint8_t *)APP_ADDR, app_size - 4);

if (stored_crc != calc_crc) {
    return false;
}
```

일단은 SP/Reset 검사만으로 충분.

---

다음 글에서 Vector Table 재배치.

[#6 - Vector Table 재배치](/posts/stm32-bootloader/stm32-bootloader-06/)
