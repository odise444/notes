---
title: "STM32 부트로더 개발기 #16 - CRC 검증"
date: 2024-12-17
tags: ["STM32", "Bootloader", "CRC", "검증"]
categories: ["개발기"]
series: ["STM32 부트로더 개발기"]
summary: "받은 펌웨어가 정말 맞는지 CRC로 확인."
---

데이터 전송 중 비트 에러 날 수 있다.

CRC로 무결성 검증하자.

---

## CRC32

가장 널리 쓰는 CRC-32 (IEEE 802.3).

```
다항식: 0x04C11DB7
초기값: 0xFFFFFFFF
XOR 출력: 0xFFFFFFFF
```

---

## 소프트웨어 CRC

```c
static const uint32_t crc32_table[256] = {
    0x00000000, 0x77073096, 0xEE0E612C, 0x990951BA,
    // ... 256개 테이블
};

uint32_t crc32_calc(uint8_t *data, uint32_t len) {
    uint32_t crc = 0xFFFFFFFF;
    
    for (uint32_t i = 0; i < len; i++) {
        uint8_t idx = (crc ^ data[i]) & 0xFF;
        crc = (crc >> 8) ^ crc32_table[idx];
    }
    
    return crc ^ 0xFFFFFFFF;
}
```

---

## 점진적 CRC

한 번에 계산 안 하고 데이터 받을 때마다:

```c
uint32_t crc32_update(uint32_t crc, uint8_t *data, uint32_t len) {
    for (uint32_t i = 0; i < len; i++) {
        uint8_t idx = (crc ^ data[i]) & 0xFF;
        crc = (crc >> 8) ^ crc32_table[idx];
    }
    return crc;
}

uint32_t crc32_final(uint32_t crc) {
    return crc ^ 0xFFFFFFFF;
}

// 사용
iap.crc = 0xFFFFFFFF;
iap.crc = crc32_update(iap.crc, data, len);  // 매 패킷
uint32_t final = crc32_final(iap.crc);       // 마지막에
```

---

## STM32 내장 CRC

STM32에 CRC 하드웨어 있음:

```c
// CRC 클럭 활성화
__HAL_RCC_CRC_CLK_ENABLE();

// 리셋
CRC->CR = CRC_CR_RESET;

// 계산 (Word 단위)
for (uint32_t i = 0; i < len/4; i++) {
    CRC->DR = ((uint32_t *)data)[i];
}

uint32_t crc = CRC->DR;
```

근데 STM32F103 내장 CRC는 다항식이 다름. 호환 안 될 수 있음.

소프트웨어 CRC가 호환성 좋음.

---

## 검증 단계

### 1. 페이지 단위 CRC

매 페이지 전송 후 CRC 체크:

```
PC:  PAGE_END + CRC(페이지 데이터)
BMS: CRC 비교 → 틀리면 재요청
```

### 2. 전체 CRC

모든 페이지 수신 후 전체 검증:

```c
void iap_start_verify(void) {
    // Buffer 영역 전체 CRC 계산
    uint32_t calc = crc32_calc((uint8_t *)BUFFER_ADDR, iap.fw_size);
    
    // PC에게 검증 시작 알림
    uint8_t resp[1] = {RESP_VERIFY_START};
    can_send(CAN_ID_BMS_TO_PC, resp, 1);
    
    // 계산된 CRC 전송 (PC가 가진 값과 비교)
    uint8_t crc_msg[5] = {RESP_VERIFY_OK};
    memcpy(&crc_msg[1], &calc, 4);
    can_send(CAN_ID_BMS_TO_PC, crc_msg, 5);
    
    iap.state = IAP_COMPLETE;
}
```

---

## Buffer → App 복사

검증 통과하면 복사:

```c
void iap_copy_to_app(void) {
    flash_unlock();
    
    // App 영역 삭제
    for (uint32_t addr = APP_ADDR; addr < APP_ADDR + iap.fw_size; addr += PAGE_SIZE) {
        flash_erase_page(addr);
    }
    
    // Buffer → App 복사
    uint8_t *src = (uint8_t *)BUFFER_ADDR;
    for (uint32_t i = 0; i < iap.fw_size; i += 2) {
        uint16_t data = src[i] | (src[i+1] << 8);
        flash_program_halfword(APP_ADDR + i, data);
    }
    
    flash_lock();
}
```

---

## 에러 복구

복사 중 에러나면?

Buffer에 펌웨어 아직 있음. 재부팅 후 다시 복사 시도 가능.

---

다음 글에서 Python 업로더.

[#17 - Python 업로더](/posts/stm32-bootloader/stm32-bootloader-17/)
