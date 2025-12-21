---
title: "Ghidra STM32 역분석 #30 - 빌드 및 비교"
date: 2024-12-14
draft: false
tags: ["Ghidra", "STM32", "리버스엔지니어링", "역분석", "빌드"]
categories: ["역분석"]
series: ["Ghidra 역분석"]
summary: "복원한 소스를 빌드하고 원본 바이너리와 비교."
---

복원한 소스를 빌드해서 원본이랑 비교해보자.

---

## 프로젝트 구조

```
bootloader/
├── Core/
│   ├── Src/
│   │   ├── main.c
│   │   ├── can_iap.c
│   │   └── flash.c
│   └── Inc/
│       ├── main.h
│       ├── can_iap.h
│       └── flash.h
├── Drivers/
│   └── CMSIS/
└── STM32F103VE.ld
```

---

## 링커 스크립트

원본이랑 같은 메모리 맵으로:

```ld
MEMORY
{
    FLASH (rx) : ORIGIN = 0x08000000, LENGTH = 16K
    RAM (rwx)  : ORIGIN = 0x20000000, LENGTH = 64K
}

SECTIONS
{
    .isr_vector : {
        . = ALIGN(4);
        KEEP(*(.isr_vector))
        . = ALIGN(4);
    } >FLASH
    
    .text : {
        *(.text*)
        *(.rodata*)
    } >FLASH
    
    /* ... */
}
```

---

## 빌드

```bash
arm-none-eabi-gcc -mcpu=cortex-m3 -mthumb -O2 \
    -DSTM32F103xE \
    -I Core/Inc \
    -I Drivers/CMSIS/Include \
    Core/Src/*.c \
    -T STM32F103VE.ld \
    -o bootloader.elf

arm-none-eabi-objcopy -O ihex bootloader.elf bootloader.hex
```

---

## 크기 비교

```bash
arm-none-eabi-size bootloader.elf
```

```
   text    data     bss     dec     hex filename
  14832      48     512   15392    3c20 bootloader.elf
```

원본: 16KB (0x4000)
복원: 약 15KB

비슷하다. 컴파일러 버전이나 옵션 차이로 약간 다를 수 있음.

---

## 바이너리 비교

```bash
# HEX를 BIN으로
objcopy -I ihex -O binary 200429.hex original.bin
objcopy -I ihex -O binary bootloader.hex rebuilt.bin

# 비교
xxd original.bin > original.txt
xxd rebuilt.bin > rebuilt.txt
diff original.txt rebuilt.txt | head -50
```

100% 같진 않다. 컴파일러 버전, 최적화 옵션, 빌드 환경 다르니까.

---

## 기능 테스트

바이너리가 다르더라도 기능이 같으면 성공:

1. 부트 조건 (버튼, 매직값) 동작 확인
2. CAN 연결 테스트
3. 펌웨어 업로드 테스트
4. 검증 및 점프 확인

전부 통과하면 역분석 + 복원 성공.

---

Part 7 끝. 다음은 검증 & 활용편.

[#31 - CAN 스니핑 검증](/posts/ghidra/ghidra-stm32-re-31/)
