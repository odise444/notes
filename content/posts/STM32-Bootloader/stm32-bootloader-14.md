---
title: "STM32 부트로더 개발기 #14 - 상태 머신 구현"
date: 2024-12-17
tags: ["STM32", "Bootloader", "상태머신", "CAN"]
categories: ["개발기"]
series: ["STM32 부트로더 개발기"]
summary: "프로토콜 상태를 체계적으로 관리하자."
---

CAN 메시지 오는 대로 처리하면 코드가 스파게티가 된다.

상태 머신으로 정리하자.

---

## 상태 정의

```c
typedef enum {
    IAP_IDLE,           // 대기 (연결 대기)
    IAP_CONNECTED,      // 연결됨 (인증 대기)
    IAP_AUTHENTICATED,  // 인증됨 (크기 대기)
    IAP_RECEIVING,      // 데이터 수신 중
    IAP_VERIFYING,      // 검증 중
    IAP_COMPLETE,       // 완료
    IAP_ERROR           // 에러
} IapState;
```

---

## 상태 전이

```
         CONNECT
IDLE ──────────────> CONNECTED
                         │
                    AUTH │
                         ▼
                   AUTHENTICATED
                         │
                    SIZE │
                         ▼
                    RECEIVING ◄──┐
                         │       │
                   PAGE_END     │ PAGE_REQ
                         │       │
                         ▼       │
                    VERIFYING ───┘
                         │
                   VERIFY_OK
                         ▼
                    COMPLETE
```

---

## 상태 구조체

```c
typedef struct {
    IapState state;
    uint32_t seed;          // 인증용
    uint32_t fw_size;       // 펌웨어 크기
    uint32_t received;      // 받은 바이트
    uint16_t current_page;  // 현재 페이지
    uint16_t total_pages;   // 전체 페이지
    uint32_t page_offset;   // 페이지 내 오프셋
    uint32_t crc;           // CRC 계산값
    uint32_t last_tick;     // 타임아웃용
} IapContext;

IapContext iap;
```

---

## 메인 루프

```c
void iap_process(void) {
    // 타임아웃 체크
    if (iap_check_timeout()) {
        iap_reset();
        return;
    }
    
    // CAN 메시지 있으면 처리
    if (can_rx_available()) {
        CAN_RxHeaderTypeDef header;
        uint8_t data[8];
        can_receive(&header, data);
        
        iap_handle_message(header.StdId, data, header.DLC);
    }
}
```

---

## 메시지 핸들러

```c
void iap_handle_message(uint32_t id, uint8_t *data, uint8_t len) {
    if (id != CAN_ID_PC_TO_BMS) return;
    
    uint8_t cmd = data[0];
    
    switch (iap.state) {
        case IAP_IDLE:
            if (cmd == CMD_CONNECT) {
                iap_handle_connect();
            }
            break;
            
        case IAP_CONNECTED:
            if (cmd == CMD_AUTH) {
                iap_handle_auth(data);
            }
            break;
            
        case IAP_AUTHENTICATED:
            if (cmd == CMD_SIZE) {
                iap_handle_size(data);
            }
            break;
            
        case IAP_RECEIVING:
            if (cmd == CMD_PAGE_START) {
                iap_handle_page_start(data);
            } else if (cmd == CMD_PAGE_END) {
                iap_handle_page_end(data);
            } else {
                // raw data
                iap_handle_data(data, len);
            }
            break;
    }
    
    iap.last_tick = HAL_GetTick();
}
```

---

## 연결 처리

```c
void iap_handle_connect(void) {
    // 랜덤 seed 생성
    iap.seed = generate_seed();
    
    // CHALLENGE 전송
    uint8_t resp[8] = {0};
    resp[0] = RESP_CHALLENGE;
    memcpy(&resp[1], &iap.seed, 4);
    can_send(CAN_ID_BMS_TO_PC, resp, 5);
    
    iap.state = IAP_CONNECTED;
}
```

---

## 인증 처리

```c
void iap_handle_auth(uint8_t *data) {
    uint32_t key;
    memcpy(&key, &data[1], 4);
    
    uint32_t expected = calc_key(iap.seed);
    
    if (key == expected) {
        // 인증 성공
        uint8_t resp[1] = {RESP_AUTH_OK};
        can_send(CAN_ID_BMS_TO_PC, resp, 1);
        
        // 크기 요청
        uint8_t req[1] = {RESP_SIZE_REQ};
        can_send(CAN_ID_BMS_TO_PC, req, 1);
        
        iap.state = IAP_AUTHENTICATED;
    } else {
        // 인증 실패
        iap_send_error(ERR_AUTH);
        iap_reset();
    }
}
```

---

## 타임아웃 처리

```c
#define IAP_TIMEOUT_MS  10000

bool iap_check_timeout(void) {
    if (iap.state == IAP_IDLE) {
        return false;  // IDLE은 타임아웃 없음
    }
    
    if ((HAL_GetTick() - iap.last_tick) > IAP_TIMEOUT_MS) {
        return true;
    }
    return false;
}

void iap_reset(void) {
    memset(&iap, 0, sizeof(iap));
    iap.state = IAP_IDLE;
}
```

---

다음 글에서 데이터 수신 및 쓰기.

[#15 - 데이터 수신 및 쓰기](/posts/stm32-bootloader/stm32-bootloader-15/)
