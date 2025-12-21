---
title: "Ghidra STM32 역분석 #29 - Flash 드라이버 복원"
date: 2024-12-14
draft: false
tags: ["Ghidra", "STM32", "리버스엔지니어링", "역분석", "Flash"]
categories: ["역분석"]
series: ["Ghidra 역분석"]
summary: "Flash Unlock/Erase/Write 함수 복원."
---

Flash 접근 함수들을 복원한다. STM32F1 레퍼런스 매뉴얼 보면서.

---

## flash.h

```c
#ifndef FLASH_H
#define FLASH_H

#include "stm32f1xx.h"

#define FLASH_KEY1      0x45670123
#define FLASH_KEY2      0xCDEF89AB

#define BUFFER_ADDR     0x08004000
#define APP_ADDR        0x08042800

void Flash_Unlock(void);
void Flash_Lock(void);
void Flash_ErasePage(uint32_t addr);
void Flash_WriteHalfWord(uint32_t addr, uint16_t data);
void Flash_WritePage(uint32_t page, uint8_t *data);

#endif
```

---

## Flash_Unlock / Lock

```c
void Flash_Unlock(void) {
    if (FLASH->CR & FLASH_CR_LOCK) {
        FLASH->KEYR = FLASH_KEY1;
        FLASH->KEYR = FLASH_KEY2;
    }
}

void Flash_Lock(void) {
    FLASH->CR |= FLASH_CR_LOCK;
}
```

키 순서 중요하다. 틀리면 락 안 풀림.

---

## Flash_ErasePage

```c
void Flash_ErasePage(uint32_t addr) {
    // BSY 대기
    while (FLASH->SR & FLASH_SR_BSY);
    
    // 페이지 삭제 설정
    FLASH->CR |= FLASH_CR_PER;
    FLASH->AR = addr;
    FLASH->CR |= FLASH_CR_STRT;
    
    // 완료 대기
    while (FLASH->SR & FLASH_SR_BSY);
    
    // PER 클리어
    FLASH->CR &= ~FLASH_CR_PER;
}
```

---

## Flash_WriteHalfWord

```c
void Flash_WriteHalfWord(uint32_t addr, uint16_t data) {
    while (FLASH->SR & FLASH_SR_BSY);
    
    FLASH->CR |= FLASH_CR_PG;
    
    *(volatile uint16_t *)addr = data;
    
    while (FLASH->SR & FLASH_SR_BSY);
    
    FLASH->CR &= ~FLASH_CR_PG;
}
```

16비트 단위로만 쓸 수 있다.

---

## Flash_WritePage

```c
void Flash_WritePage(uint32_t page, uint8_t *data) {
    uint32_t addr = BUFFER_ADDR + page * 2048;
    
    Flash_Unlock();
    Flash_ErasePage(addr);
    
    for (int i = 0; i < 2048; i += 2) {
        uint16_t halfword = data[i] | (data[i + 1] << 8);
        Flash_WriteHalfWord(addr + i, halfword);
    }
    
    Flash_Lock();
}
```

---

## 에러 체크 추가

원본엔 없었지만 안전하게:

```c
uint8_t Flash_ErasePage_Safe(uint32_t addr) {
    Flash_ErasePage(addr);
    
    // EOP 체크
    if (FLASH->SR & FLASH_SR_EOP) {
        FLASH->SR = FLASH_SR_EOP;  // 클리어
        return 0;  // 성공
    }
    
    return 1;  // 실패
}
```

---

다음 글에서 빌드하고 원본이랑 비교.

[#30 - 빌드 및 비교](/posts/ghidra/ghidra-stm32-re-30/)
