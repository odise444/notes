---
title: "STM32 부트로더 개발기 #12 - 에러 처리"
date: 2024-12-17
tags: ["STM32", "Bootloader", "Flash", "에러"]
categories: ["개발기"]
series: ["STM32 부트로더 개발기"]
summary: "Flash 작업 실패하면 어떻게 해야 하나."
---

Flash 작업이 항상 성공하진 않는다.

에러 처리 안 하면 펌웨어 손상될 수 있음.

---

## Flash 에러 종류

`FLASH_SR` 레지스터:

| 비트 | 이름 | 의미 |
|------|------|------|
| BSY | Busy | 작업 중 |
| PGERR | Programming Error | 프로그래밍 에러 |
| WRPRTERR | Write Protection Error | 쓰기 보호 에러 |
| EOP | End of Operation | 작업 완료 |

---

## PGERR (Programming Error)

원인:
- 0xFF 아닌 곳에 쓰기 시도
- 정렬 안 된 주소에 쓰기
- Erase 중 프로그래밍 시도

```c
if (FLASH->SR & FLASH_SR_PGERR) {
    // 에러 클리어
    FLASH->SR = FLASH_SR_PGERR;
    return FLASH_ERROR_PG;
}
```

---

## WRPRTERR (Write Protection Error)

원인:
- 쓰기 보호된 영역에 쓰기/삭제 시도
- Option Bytes로 보호 설정됨

```c
if (FLASH->SR & FLASH_SR_WRPRTERR) {
    FLASH->SR = FLASH_SR_WRPRTERR;
    return FLASH_ERROR_WRP;
}
```

---

## 타임아웃

무한 대기 방지:

```c
#define FLASH_TIMEOUT_MS  5000

bool flash_wait_busy_timeout(void) {
    uint32_t start = HAL_GetTick();
    
    while (FLASH->SR & FLASH_SR_BSY) {
        if ((HAL_GetTick() - start) > FLASH_TIMEOUT_MS) {
            return false;  // 타임아웃
        }
    }
    return true;
}
```

---

## 에러 코드 정의

```c
typedef enum {
    FLASH_OK = 0,
    FLASH_ERROR_PG,       // Programming error
    FLASH_ERROR_WRP,      // Write protection
    FLASH_ERROR_TIMEOUT,  // Timeout
    FLASH_ERROR_VERIFY,   // Verification failed
    FLASH_ERROR_ADDR      // Invalid address
} FlashStatus;
```

---

## 검증 (Verify)

쓰기 후 읽어서 확인:

```c
FlashStatus flash_write_verify(uint32_t addr, uint8_t *data, uint32_t len) {
    // 쓰기
    FlashStatus status = flash_program_buffer(addr, data, len);
    if (status != FLASH_OK) {
        return status;
    }
    
    // 검증
    if (memcmp((void *)addr, data, len) != 0) {
        return FLASH_ERROR_VERIFY;
    }
    
    return FLASH_OK;
}
```

---

## 재시도

가끔 실패하면 재시도:

```c
#define MAX_RETRY  3

FlashStatus flash_write_with_retry(uint32_t addr, uint8_t *data, uint32_t len) {
    for (int i = 0; i < MAX_RETRY; i++) {
        FlashStatus status = flash_write_verify(addr, data, len);
        if (status == FLASH_OK) {
            return FLASH_OK;
        }
        
        // 재시도 전 페이지 다시 삭제
        flash_erase_page(addr);
    }
    
    return FLASH_ERROR_VERIFY;
}
```

---

## 업데이트 실패 시 복구

이중 버퍼 전략:

1. 새 펌웨어를 Buffer 영역에 수신
2. 검증 완료 후 App 영역에 복사
3. 복사 중 실패하면 이전 버전 유지

```c
bool safe_update(void) {
    // 1. Buffer에 수신 (Buffer는 이미 검증됨)
    
    // 2. App 영역 삭제
    if (!flash_erase_app_area()) {
        return false;  // 실패해도 Buffer에 펌웨어 있음
    }
    
    // 3. Buffer → App 복사
    if (!flash_copy(BUFFER_ADDR, APP_ADDR, fw_size)) {
        // 실패 - 하지만 Buffer에 펌웨어 있으니
        // 재부팅 후 다시 시도 가능
        return false;
    }
    
    return true;
}
```

---

다음 글에서 CAN IAP 프로토콜 설계.

[#13 - 프로토콜 설계](/posts/stm32-bootloader/stm32-bootloader-13/)
