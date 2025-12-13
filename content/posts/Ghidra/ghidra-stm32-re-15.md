---
title: "Ghidra로 STM32 부트로더 역분석한 썰 #15 - 상태 머신 복원"
date: 2024-12-13
draft: false
tags: ["Ghidra", "STM32", "리버스엔지니어링", "역분석", "상태머신", "FSM"]
categories: ["역분석"]
summary: "부트로더의 전체 흐름을 상태 머신으로 복원한다. 상태 전이와 타임아웃 처리."
---

## 지난 글 요약

[지난 글](/posts/ghidra/ghidra-stm32-re-14/)에서 명령 코드를 분석했다. 0x30~0x36 요청, 0x40~0x46 응답. 이제 **상태 머신**을 복원해서 전체 흐름을 이해하자.

## 상태 변수 찾기

명령 처리 함수에서 공통으로 참조하는 주소:

```c
// 상태 체크 패턴
if (*(uint8_t *)0x20000300 != 3) {
    return;
}
```

**0x20000300 = g_iap_state**

## 상태값 추출

각 함수에서 설정/체크하는 값:

| 값 | 설정 위치 | 다음 상태 조건 |
|----|-----------|---------------|
| 0 | 초기화, 에러 시 | 0x30 수신 시 → 1 |
| 1 | 0x30 처리 후 | 0x31 수신 시 → 2 |
| 2 | 0x31 성공 후 | 0x32 수신 시 → 3 |
| 3 | 0x32 처리 후 | 0x33 수신 시 → 4 |
| 4 | 0x33 처리 후 | 0x34 수신 시 → 3 or 5 |
| 5 | 모든 페이지 후 | 0x36 수신 시 → 6 |
| 6 | 0x36 성공 후 | 완료 |

## 상태 다이어그램

```
                    ┌─────────────────────────────────┐
                    │                                 │
                    ▼                                 │
            ┌───────────────┐                        │
            │    IDLE (0)   │◄───────────────────────┤
            └───────┬───────┘      Timeout/Error     │
                    │ 0x30                           │
                    ▼                                │
            ┌───────────────┐                        │
            │ WAIT_CALC (1) │────────────────────────┤
            └───────┬───────┘      Timeout           │
                    │ 0x31 (valid)                   │
                    ▼                                │
            ┌───────────────┐                        │
            │  AUTH'D (2)   │────────────────────────┤
            └───────┬───────┘      Error             │
                    │ 0x32                           │
                    ▼                                │
            ┌───────────────┐                        │
     ┌─────►│ RECEIVING (3) │────────────────────────┤
     │      └───────┬───────┘      Timeout           │
     │              │ 0x33                           │
     │              ▼                                │
     │      ┌───────────────┐                        │
     │      │ PAGE_DATA (4) │────────────────────────┤
     │      └───────┬───────┘      Error             │
     │              │ 0x34                           │
     │              ▼                                │
     │      ┌───────────────┐                        │
     │      │  More pages?  │                        │
     │      └───────┬───────┘                        │
     │         Yes/│ \No                             │
     └─────────────┘  \                              │
                       ▼                             │
            ┌───────────────┐                        │
            │ VERIFYING (5) │────────────────────────┘
            └───────┬───────┘      Verify Fail
                    │ 0x36 (OK)
                    ▼
            ┌───────────────┐
            │ COMPLETE (6)  │
            └───────┬───────┘
                    │ Reset or Jump
                    ▼
              ┌──────────┐
              │   App    │
              └──────────┘
```

## 타임아웃 처리

`FUN_08003400` (타임아웃 체크 함수):

```c
void FUN_08003400(void) {
    uint8_t state = *(uint8_t *)0x20000300;
    
    // IDLE 상태면 타임아웃 없음
    if (state == 0 || state == 6) {
        return;
    }
    
    // 타임아웃 카운터 증가
    *(uint32_t *)0x20000304 += 1;  // g_timeout_counter
    
    // 10초 타임아웃 (100ms 주기 × 100)
    if (*(uint32_t *)0x20000304 >= 100) {
        // 타임아웃 발생
        *(uint8_t *)0x20000300 = 0;  // STATE_IDLE
        *(uint32_t *)0x20000304 = 0;
        
        // 에러 응답
        uint8_t response[1] = {0x4F};
        FUN_08003C00(0x5FE, response, 1);
    }
}
```

## 복원된 상태 머신 코드

```c
// iap_state_machine.c

typedef enum {
    STATE_IDLE = 0,
    STATE_WAIT_CALC,
    STATE_AUTHENTICATED,
    STATE_RECEIVING,
    STATE_PAGE_DATA,
    STATE_VERIFYING,
    STATE_COMPLETE
} iap_state_t;

typedef struct {
    iap_state_t state;
    uint32_t timeout_counter;
    uint32_t fw_size;
    uint32_t total_pages;
    uint32_t current_page;
    uint32_t page_offset;
    uint8_t page_buffer[2048];
} iap_context_t;

static iap_context_t g_iap;

void IAP_Init(void) {
    g_iap.state = STATE_IDLE;
    g_iap.timeout_counter = 0;
    g_iap.fw_size = 0;
    g_iap.total_pages = 0;
    g_iap.current_page = 0;
    g_iap.page_offset = 0;
}

void IAP_ProcessCommand(uint8_t cmd, uint8_t *data, uint8_t len) {
    // 타임아웃 리셋
    g_iap.timeout_counter = 0;
    
    switch (cmd) {
    case CMD_CONNECT:
        IAP_HandleConnect();
        break;
        
    case CMD_CALC:
        IAP_HandleCalc(data, len);
        break;
        
    case CMD_SIZE:
        IAP_HandleSize(data, len);
        break;
        
    case CMD_PAGE_START:
        IAP_HandlePageStart(data, len);
        break;
        
    case CMD_PAGE_END:
        IAP_HandlePageEnd();
        break;
        
    case CMD_VERIFY:
        IAP_HandleVerify(data, len);
        break;
        
    default:
        // 데이터 프레임 처리
        if (g_iap.state == STATE_PAGE_DATA) {
            IAP_HandlePageData(data, len);
        }
        break;
    }
}

void IAP_HandleConnect(void) {
    if (g_iap.state != STATE_IDLE) {
        return;  // 이미 진행 중
    }
    
    // FW 정보 응답
    uint8_t response[8];
    response[0] = RSP_FWINFO;
    memcpy(&response[1], g_fw_check, 4);
    memcpy(&response[5], g_fw_date, 3);
    
    CAN_Transmit(CAN_ID_BMS_TO_PC, response, 8);
    
    g_iap.state = STATE_WAIT_CALC;
}

void IAP_HandleCalc(uint8_t *data, uint8_t len) {
    if (g_iap.state != STATE_WAIT_CALC) {
        return;
    }
    
    uint32_t received = data[1] | (data[2] << 8) | 
                        (data[3] << 16) | (data[4] << 24);
    uint32_t expected = IAP_CalcKey();
    
    if (received == expected) {
        // 인증 성공
        uint8_t response[1] = {RSP_AUTH_OK};
        CAN_Transmit(CAN_ID_BMS_TO_PC, response, 1);
        
        // 크기 요청
        uint8_t size_req[1] = {RSP_SIZE_REQ};
        CAN_Transmit(CAN_ID_BMS_TO_PC, size_req, 1);
        
        g_iap.state = STATE_AUTHENTICATED;
    } else {
        // 인증 실패
        uint8_t response[1] = {RSP_ERROR};
        CAN_Transmit(CAN_ID_BMS_TO_PC, response, 1);
        
        g_iap.state = STATE_IDLE;
    }
}

void IAP_HandleSize(uint8_t *data, uint8_t len) {
    if (g_iap.state != STATE_AUTHENTICATED) {
        return;
    }
    
    g_iap.fw_size = data[1] | (data[2] << 8) | 
                    (data[3] << 16) | (data[4] << 24);
    
    // 크기 검증
    if (g_iap.fw_size > MAX_FW_SIZE) {
        uint8_t response[1] = {RSP_ERROR};
        CAN_Transmit(CAN_ID_BMS_TO_PC, response, 1);
        g_iap.state = STATE_IDLE;
        return;
    }
    
    g_iap.total_pages = (g_iap.fw_size + PAGE_SIZE - 1) / PAGE_SIZE;
    g_iap.current_page = 0;
    
    // 첫 페이지 요청
    uint8_t response[2] = {RSP_PAGE_REQ, 0};
    CAN_Transmit(CAN_ID_BMS_TO_PC, response, 2);
    
    g_iap.state = STATE_RECEIVING;
}

void IAP_HandlePageStart(uint8_t *data, uint8_t len) {
    if (g_iap.state != STATE_RECEIVING) {
        return;
    }
    
    uint8_t page_num = data[1];
    if (page_num != g_iap.current_page) {
        return;  // 순서 오류
    }
    
    g_iap.page_offset = 0;
    memset(g_iap.page_buffer, 0xFF, PAGE_SIZE);
    
    g_iap.state = STATE_PAGE_DATA;
}

void IAP_HandlePageData(uint8_t *data, uint8_t len) {
    if (g_iap.page_offset + len > PAGE_SIZE) {
        return;  // 오버플로 방지
    }
    
    memcpy(&g_iap.page_buffer[g_iap.page_offset], data, len);
    g_iap.page_offset += len;
}

void IAP_HandlePageEnd(void) {
    if (g_iap.state != STATE_PAGE_DATA) {
        return;
    }
    
    // Flash에 쓰기
    uint32_t addr = BUFFER_START + g_iap.current_page * PAGE_SIZE;
    Flash_WritePage(addr, g_iap.page_buffer);
    
    g_iap.current_page++;
    
    if (g_iap.current_page >= g_iap.total_pages) {
        // 모든 페이지 수신 완료
        uint8_t response[1] = {RSP_VERIFY_START};
        CAN_Transmit(CAN_ID_BMS_TO_PC, response, 1);
        
        g_iap.state = STATE_VERIFYING;
    } else {
        // 다음 페이지 요청
        uint8_t response[2] = {RSP_PAGE_REQ, g_iap.current_page};
        CAN_Transmit(CAN_ID_BMS_TO_PC, response, 2);
        
        g_iap.state = STATE_RECEIVING;
    }
}

void IAP_HandleVerify(uint8_t *data, uint8_t len) {
    if (g_iap.state != STATE_VERIFYING) {
        return;
    }
    
    uint8_t result = data[1];
    
    if (result == 0x01) {
        // 검증 성공 - 버퍼를 앱 영역으로 복사
        IAP_CopyBufferToApp();
        
        uint8_t response[1] = {RSP_COMPLETE};
        CAN_Transmit(CAN_ID_BMS_TO_PC, response, 1);
        
        g_iap.state = STATE_COMPLETE;
        
        // 잠시 후 리셋
        HAL_Delay(100);
        NVIC_SystemReset();
    } else {
        // 검증 실패
        uint8_t response[1] = {RSP_ERROR};
        CAN_Transmit(CAN_ID_BMS_TO_PC, response, 1);
        
        g_iap.state = STATE_IDLE;
    }
}

void IAP_CheckTimeout(void) {
    if (g_iap.state == STATE_IDLE || g_iap.state == STATE_COMPLETE) {
        return;
    }
    
    g_iap.timeout_counter++;
    
    if (g_iap.timeout_counter >= TIMEOUT_COUNT) {
        // 타임아웃
        uint8_t response[1] = {RSP_ERROR};
        CAN_Transmit(CAN_ID_BMS_TO_PC, response, 1);
        
        g_iap.state = STATE_IDLE;
        g_iap.timeout_counter = 0;
    }
}
```

## main() 복원

```c
int main(void) {
    // 시스템 초기화
    SystemInit();
    GPIO_Init();
    CAN_Init();
    Flash_Init();
    
    // IAP 초기화
    IAP_Init();
    
    // 부트 모드 체크
    if (!IsBootButtonPressed() && IsAppValid()) {
        // 앱으로 점프
        JumpToApp();
    }
    
    // IAP 모드 LED 표시
    LED_On(LED_BOOT);
    
    // 메인 루프
    while (1) {
        // CAN 메시지 처리
        CAN_ProcessMessages();
        
        // 타임아웃 체크 (100ms 주기)
        static uint32_t last_tick = 0;
        if (HAL_GetTick() - last_tick >= 100) {
            last_tick = HAL_GetTick();
            IAP_CheckTimeout();
            LED_Toggle(LED_STATUS);
        }
    }
}
```

## 에러 처리 복원

```c
void IAP_Error(uint8_t error_code) {
    // 에러 응답
    uint8_t response[2] = {RSP_ERROR, error_code};
    CAN_Transmit(CAN_ID_BMS_TO_PC, response, 2);
    
    // 상태 리셋
    g_iap.state = STATE_IDLE;
    g_iap.timeout_counter = 0;
    
    // 에러 LED
    LED_Off(LED_STATUS);
    for (int i = 0; i < error_code; i++) {
        LED_On(LED_ERROR);
        HAL_Delay(200);
        LED_Off(LED_ERROR);
        HAL_Delay(200);
    }
}
```

## 삽질: 상태 전이 누락

한 상태에서 여러 명령 처리 시 상태 체크 누락:

```c
// 잘못된 코드
void IAP_HandlePageEnd(void) {
    // 상태 체크 없이 바로 처리
    Flash_WritePage(...);  // STATE_PAGE_DATA 아니어도 실행됨!
}

// 올바른 코드
void IAP_HandlePageEnd(void) {
    if (g_iap.state != STATE_PAGE_DATA) {
        return;  // 상태 체크!
    }
    Flash_WritePage(...);
}
```

## 삽질: 타임아웃 리셋

유효한 명령 수신 시 타임아웃 리셋 누락:

```c
// 잘못된 코드
void IAP_ProcessCommand(...) {
    switch (cmd) {
        // ... 처리만 하고 타임아웃 안 건드림
    }
}

// 올바른 코드
void IAP_ProcessCommand(...) {
    g_iap.timeout_counter = 0;  // 먼저 리셋!
    switch (cmd) {
        // ...
    }
}
```

## 정리

| 상태 | 값 | 대기 명령 | 타임아웃 |
|------|-----|----------|----------|
| IDLE | 0 | 0x30 | 없음 |
| WAIT_CALC | 1 | 0x31 | 10초 |
| AUTH'D | 2 | 0x32 | 10초 |
| RECEIVING | 3 | 0x33 | 10초 |
| PAGE_DATA | 4 | data/0x34 | 10초 |
| VERIFYING | 5 | 0x36 | 10초 |
| COMPLETE | 6 | - | 없음 |

**다음 글에서**: Connection Key 알고리즘 - FwChk, FwDate로 Key 계산.

---

## 시리즈 목차

**Part 4: CAN IAP 프로토콜 역분석편**
- [#13 - CAN 수신 핸들러 분석](/posts/ghidra/ghidra-stm32-re-13/)
- [#14 - 명령 코드 체계 파악](/posts/ghidra/ghidra-stm32-re-14/)
- **#15 - 상태 머신 복원** ← 현재 글
- #16 - Connection Key 알고리즘
- #17 - 페이지 전송 프로토콜
- #18 - CRC 검증 함수 분석

---

## 참고 자료

- [Finite State Machine Design](https://www.embedded.com/design-patterns-for-state-machines/)
- [STM32 Bootloader Application Note](https://www.st.com/resource/en/application_note/an2606.pdf)
