---
title: "STM32 부트로더 개발기 #10 - Page Erase"
date: 2024-12-17
tags: ["STM32", "Bootloader", "Flash", "Erase"]
categories: ["개발기"]
series: ["STM32 부트로더 개발기"]
summary: "Flash에 쓰기 전에 먼저 지워야 한다. 0xFF로."
---

Flash 특성: 1→0은 가능, 0→1은 불가능.

그래서 쓰기 전에 먼저 지워야 함. 지우면 0xFF가 됨.

---

## 페이지 단위

STM32F103은 페이지 단위로 삭제.

- **F103C8/RB (≤128KB)**: 1KB 페이지
- **F103VE/ZE (512KB)**: 2KB 페이지

한 바이트만 지우고 싶어도 페이지 전체가 날아감.

---

## Erase 절차

1. Unlock
2. `PER` 비트 설정 (Page Erase)
3. `FAR`에 페이지 주소 쓰기
4. `STRT` 비트 설정
5. BSY 대기
6. `PER` 클리어

---

## 코드

```c
bool flash_erase_page(uint32_t page_addr) {
    // BSY 대기
    while (FLASH->SR & FLASH_SR_BSY);
    
    // Page Erase 설정
    FLASH->CR |= FLASH_CR_PER;
    
    // 페이지 주소
    FLASH->AR = page_addr;
    
    // 시작
    FLASH->CR |= FLASH_CR_STRT;
    
    // 완료 대기
    while (FLASH->SR & FLASH_SR_BSY);
    
    // PER 클리어
    FLASH->CR &= ~FLASH_CR_PER;
    
    // 에러 체크
    if (FLASH->SR & (FLASH_SR_PGERR | FLASH_SR_WRPRTERR)) {
        return false;
    }
    
    return true;
}
```

---

## HAL 함수

```c
FLASH_EraseInitTypeDef erase;
uint32_t page_error;

erase.TypeErase = FLASH_TYPEERASE_PAGES;
erase.PageAddress = page_addr;
erase.NbPages = 1;

HAL_FLASHEx_Erase(&erase, &page_error);
```

---

## 여러 페이지 삭제

앱 영역 전체 삭제:

```c
bool flash_erase_app_area(void) {
    uint32_t addr = APP_ADDR;
    uint32_t end = APP_ADDR + APP_SIZE;
    uint32_t page_size = 2048;  // 2KB
    
    flash_unlock();
    
    while (addr < end) {
        if (!flash_erase_page(addr)) {
            flash_lock();
            return false;
        }
        addr += page_size;
    }
    
    flash_lock();
    return true;
}
```

---

## 삭제 시간

페이지 하나 삭제하는 데 약 20~40ms.

246KB = 123페이지 = 약 3~5초.

진행 상황 표시해주면 좋음.

---

## 주의: 부트로더 영역

부트로더 영역 (0x08000000 ~ 0x08003FFF) 지우면 벽돌.

주소 체크 필수:

```c
bool is_safe_to_erase(uint32_t addr) {
    // 부트로더 영역이면 거부
    if (addr < BOOTLOADER_SIZE) {
        return false;
    }
    return true;
}
```

---

## 삭제 확인

삭제 후 읽어보면 0xFF:

```c
bool verify_erased(uint32_t addr, uint32_t len) {
    uint8_t *p = (uint8_t *)addr;
    for (uint32_t i = 0; i < len; i++) {
        if (p[i] != 0xFF) {
            return false;
        }
    }
    return true;
}
```

---

다음 글에서 Half-Word Program.

[#11 - Half-Word Program](/posts/stm32-bootloader/stm32-bootloader-11/)
