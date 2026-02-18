---
title: "STM32 부트로더 개발기 #3 - 부트로더 vs 앱 영역 분리"
date: 2024-12-17
tags: ["STM32", "Bootloader", "링커", "CubeMX"]
categories: ["개발기"]
series: ["STM32 부트로더 개발기"]
summary: "프로젝트 두 개 만들어야 한다. 부트로더용, 앱용."
---

부트로더와 앱은 별개 프로젝트다.

각각 빌드해서 각각 Flash에 굽는다.

---

## 프로젝트 구조

```
Workspace/
├── BMS_Bootloader/    ← 부트로더 프로젝트
│   ├── Core/
│   ├── Drivers/
│   └── STM32F103VE_FLASH.ld
│
└── BMS_App/           ← 앱 프로젝트
    ├── Core/
    ├── Drivers/
    └── STM32F103VE_FLASH.ld   ← 시작 주소 다름
```

---

## CubeMX 설정

두 프로젝트 다 CubeMX로 생성.

동일한 MCU (STM32F103VE) 선택.

차이점:
- 부트로더: CAN, 최소한의 GPIO
- 앱: 전체 기능 (ADC, TIM, UART 등)

---

## 링커 스크립트 - 부트로더

`STM32F103VE_FLASH.ld` 수정:

```ld
MEMORY
{
  RAM    (xrw) : ORIGIN = 0x20000000, LENGTH = 64K
  FLASH  (rx)  : ORIGIN = 0x08000000, LENGTH = 16K   /* 부트로더 영역만 */
}
```

부트로더는 16KB만 사용.

---

## 링커 스크립트 - 앱

```ld
MEMORY
{
  RAM    (xrw) : ORIGIN = 0x20000000, LENGTH = 64K
  FLASH  (rx)  : ORIGIN = 0x08042800, LENGTH = 246K  /* 앱 영역 */
}
```

앱은 0x08042800부터 시작.

---

## 빌드 확인

빌드 후 `.map` 파일 확인:

**부트로더:**
```
.text          0x08000000     0x2a4c
               0x08000000        Reset_Handler
```

**앱:**
```
.text          0x08042800     0x1234
               0x08042800        Reset_Handler
```

시작 주소가 다른 거 확인.

---

## bin 파일 생성

hex나 elf 말고 bin이 필요함. CAN으로 전송할 거니까.

Post-build 명령 추가:

```
arm-none-eabi-objcopy -O binary ${ProjName}.elf ${ProjName}.bin
```

STM32CubeIDE에서:
- Project → Properties → C/C++ Build → Settings
- Build Steps → Post-build steps

---

## Flash 굽기 순서

1. 부트로더 먼저 (0x08000000)
2. 앱은 부트로더 통해서 (또는 직접 0x08042800)

개발 중에는 ST-Link로 둘 다 직접 구울 수 있음.

```bash
# 부트로더
st-flash write bootloader.bin 0x08000000

# 앱
st-flash write app.bin 0x08042800
```

---

## 주의사항

앱 프로젝트에서 **system_stm32f1xx.c** 확인:

```c
#define VECT_TAB_OFFSET  0x00000000U
```

이거 나중에 바꿔야 함. 앱의 Vector Table 위치 알려줘야 하니까.

일단은 그대로 두고, 부트로더에서 VTOR 설정해서 점프.

---

다음 글에서 링커 스크립트 상세 설명.

[#4 - 링커 스크립트 설정](/posts/stm32-bootloader/stm32-bootloader-04/)
