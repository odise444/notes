---
title: "STM32 부트로더 개발기 #15 - 데이터 수신 및 쓰기"
date: 2024-12-17
tags: ["STM32", "Bootloader", "CAN", "Flash"]
categories: ["개발기"]
series: ["STM32 부트로더 개발기"]
summary: "CAN으로 8바이트씩 받아서 Flash에 쓴다."
---

이제 실제로 펌웨어 데이터를 받아서 Flash에 쓰자.

---

## 버퍼 전략

RAM에 페이지 버퍼:

```c
#define PAGE_SIZE  2048
uint8_t page_buffer[PAGE_SIZE];
uint32_t page_offset = 0;
```

8바이트씩 받아서 버퍼에 채움. 페이지 다 차면 Flash에 씀.

---

## 데이터 수신

```c
void iap_handle_data(uint8_t *data, uint8_t len) {
    if (iap.state != IAP_RECEIVING) return;
    
    // 버퍼에 복사
    if (page_offset + len <= PAGE_SIZE) {
        memcpy(&page_buffer[page_offset], data, len);
        page_offset += len;
        
        // CRC 업데이트
        iap.crc = crc32_update(iap.crc, data, len);
    }
}
```

---

## 페이지 끝 처리

```c
void iap_handle_page_end(uint8_t *data) {
    // PC가 보낸 CRC
    uint32_t pc_crc;
    memcpy(&pc_crc, &data[1], 4);
    
    // CRC 비교
    if (iap.crc != pc_crc) {
        iap_send_error(ERR_CRC);
        // 페이지 재요청
        iap_request_page(iap.current_page);
        return;
    }
    
    // Flash에 쓰기
    uint32_t addr = BUFFER_ADDR + (iap.current_page * PAGE_SIZE);
    
    flash_unlock();
    if (!flash_erase_page(addr)) {
        iap_send_error(ERR_FLASH);
        flash_lock();
        return;
    }
    
    if (!flash_program_buffer(addr, page_buffer, page_offset)) {
        iap_send_error(ERR_FLASH);
        flash_lock();
        return;
    }
    flash_lock();
    
    // 다음 페이지
    iap.current_page++;
    iap.received += page_offset;
    page_offset = 0;
    iap.crc = 0xFFFFFFFF;
    
    // ACK
    uint8_t resp[3] = {RESP_PAGE_ACK};
    resp[1] = iap.current_page & 0xFF;
    resp[2] = (iap.current_page >> 8) & 0xFF;
    can_send(CAN_ID_BMS_TO_PC, resp, 3);
    
    // 다 받았나?
    if (iap.received >= iap.fw_size) {
        iap.state = IAP_VERIFYING;
        iap_start_verify();
    } else {
        iap_request_page(iap.current_page);
    }
}
```

---

## 페이지 요청

```c
void iap_request_page(uint16_t page_num) {
    uint8_t req[3] = {RESP_PAGE_REQ};
    req[1] = page_num & 0xFF;
    req[2] = (page_num >> 8) & 0xFF;
    can_send(CAN_ID_BMS_TO_PC, req, 3);
    
    // 버퍼 초기화
    memset(page_buffer, 0xFF, PAGE_SIZE);
    page_offset = 0;
    iap.crc = 0xFFFFFFFF;
}
```

---

## 진행률

```c
uint8_t get_progress(void) {
    if (iap.fw_size == 0) return 0;
    return (iap.received * 100) / iap.fw_size;
}
```

LED 깜빡이거나 CAN으로 진행률 전송 가능.

---

## 속도

CAN 500kbps에서:
- 프레임당 약 250µs (8바이트)
- 2KB 페이지 = 256프레임 ≈ 64ms
- 246KB = 123페이지 ≈ 8초

실제론 Flash 쓰기 시간 포함해서 10~15초 정도.

---

## 흐름 제어

PC가 너무 빨리 보내면?

BMS가 PAGE_REQ 보낼 때까지 PC는 대기.

```
BMS: PAGE_REQ (page 0)
PC:  PAGE_START
PC:  data × 256
PC:  PAGE_END
     (여기서 대기)
BMS: PAGE_ACK
BMS: PAGE_REQ (page 1)  ← 이거 받으면 다음 페이지 전송
```

---

다음 글에서 CRC 검증.

[#16 - CRC 검증](/posts/stm32-bootloader/stm32-bootloader-16/)
