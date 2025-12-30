---
title: "STM32 부트로더 개발기 #9 - Flash Unlock/Lock"
date: 2024-12-17
tags: ["STM32", "Bootloader", "Flash", "프로그래밍"]
categories: ["개발기"]
series: ["STM32 부트로더 개발기"]
summary: "Flash에 쓰려면 먼저 잠금 해제해야 한다."
---

CAN으로 펌웨어 받았다. 이제 Flash에 써야 한다.

근데 그냥 쓰면 안 됨. 잠금 해제부터.

---

## Flash 레지스터

STM32F103의 Flash 제어 레지스터:

```
FLASH_ACR    (0x4002 2000)  - Access Control
FLASH_KEYR   (0x4002 2004)  - Key Register (Unlock용)
FLASH_OPTKEYR(0x4002 2008)  - Option Key
FLASH_SR     (0x4002 200C)  - Status Register
FLASH_CR     (0x4002 2010)  - Control Register
FLASH_AR     (0x4002 2014)  - Address Register
```

---

## 기본 상태: 잠김

리셋 후 `FLASH_CR`의 `LOCK` 비트가 1.

이 상태에서 Flash 쓰기/삭제 시도하면 무시됨.

---

## Unlock 시퀀스

특정 키 값을 `FLASH_KEYR`에 순서대로 써야 함.

```c
#define FLASH_KEY1  0x45670123
#define FLASH_KEY2  0xCDEF89AB

void flash_unlock(void) {
    if (FLASH->CR & FLASH_CR_LOCK) {
        FLASH->KEYR = FLASH_KEY1;
        FLASH->KEYR = FLASH_KEY2;
    }
}
```

두 키를 연속으로 쓰면 `LOCK` 비트가 0이 됨.

---

## Lock

작업 끝나면 다시 잠금.

```c
void flash_lock(void) {
    FLASH->CR |= FLASH_CR_LOCK;
}
```

잠금 안 하면 실수로 Flash 덮어쓸 수 있음.

---

## HAL 함수

HAL 라이브러리 쓰면:

```c
HAL_FLASH_Unlock();  // Unlock
// ... Flash 작업 ...
HAL_FLASH_Lock();    // Lock
```

내부적으로 위 시퀀스 수행.

---

## Status Register 확인

Flash 작업 중엔 `BSY` 비트가 1.

```c
void flash_wait_busy(void) {
    while (FLASH->SR & FLASH_SR_BSY);
}
```

BSY가 0이 될 때까지 대기해야 함.

---

## 에러 비트

작업 후 에러 체크:

```c
#define FLASH_SR_PGERR   (1 << 2)  // Programming Error
#define FLASH_SR_WRPRTERR (1 << 4) // Write Protection Error

bool flash_check_error(void) {
    if (FLASH->SR & FLASH_SR_PGERR) {
        FLASH->SR = FLASH_SR_PGERR;  // 클리어
        return true;
    }
    if (FLASH->SR & FLASH_SR_WRPRTERR) {
        FLASH->SR = FLASH_SR_WRPRTERR;
        return true;
    }
    return false;
}
```

에러 비트는 1을 써서 클리어.

---

## 전체 흐름

```c
bool flash_write_page(uint32_t addr, uint8_t *data, uint32_t len) {
    flash_unlock();
    
    // 페이지 삭제
    if (!flash_erase_page(addr)) {
        flash_lock();
        return false;
    }
    
    // 데이터 쓰기
    if (!flash_program(addr, data, len)) {
        flash_lock();
        return false;
    }
    
    flash_lock();
    return true;
}
```

Unlock → Erase → Program → Lock.

---

다음 글에서 Page Erase.

[#10 - Page Erase](/posts/stm32-bootloader/stm32-bootloader-10/)
