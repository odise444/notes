---
title: "Ghidra STM32 역분석 #28 - CAN IAP 모듈 복원"
date: 2024-12-14
draft: false
tags: ["Ghidra", "STM32", "리버스엔지니어링", "역분석", "CAN"]
categories: ["역분석"]
series: ["Ghidra 역분석"]
summary: "CAN 통신과 IAP 프로토콜 처리 코드 복원."
---

IAP 프로토콜 분석 결과를 소스코드로 복원한다.

---

## can_iap.h

```c
#ifndef CAN_IAP_H
#define CAN_IAP_H

#define IAP_RX_ID       0x5FF
#define IAP_TX_ID       0x5FE

#define CMD_CONNECT     0x30
#define CMD_KEY         0x31
#define CMD_SIZE        0x32
#define CMD_DATA        0x33
#define CMD_PAGE_END    0x34
#define CMD_VERIFY      0x35
#define CMD_JUMP        0x36

#define RSP_CONNECT     0x40
#define RSP_KEY         0x41
#define RSP_SIZE        0x42
#define RSP_DATA        0x43
#define RSP_PAGE_END    0x44
#define RSP_VERIFY      0x45
#define RSP_JUMP        0x46
#define RSP_ERROR       0x4F

typedef enum {
    STATE_IDLE = 0,
    STATE_WAIT_KEY,
    STATE_CONNECTED,
    STATE_WAIT_DATA,
    STATE_PROGRAMMING,
    STATE_VERIFY,
    STATE_COMPLETE
} IAP_State_t;

void CAN_Init(void);
void IAP_Process(void);

extern volatile uint8_t g_iap_complete;

#endif
```

---

## CAN 초기화

```c
void CAN_Init(void) {
    // 클럭 활성화
    RCC->APB1ENR |= RCC_APB1ENR_CAN1EN;
    RCC->APB2ENR |= RCC_APB2ENR_AFIOEN;
    
    // CAN GPIO (PA11: RX, PA12: TX)
    GPIOA->CRH &= ~0x000FF000;
    GPIOA->CRH |= 0x000B8000;  // TX: AF PP, RX: Input
    
    // 초기화 모드 진입
    CAN1->MCR |= CAN_MCR_INRQ;
    while (!(CAN1->MSR & CAN_MSR_INAK));
    
    // 500kbps (36MHz / 4 / 18 = 500k)
    CAN1->BTR = 0x001C0003;
    
    // 필터 설정 (모든 ID 수신)
    CAN1->FMR |= CAN_FMR_FINIT;
    CAN1->FA1R |= 0x01;
    CAN1->sFilterRegister[0].FR1 = 0;
    CAN1->sFilterRegister[0].FR2 = 0;
    CAN1->FMR &= ~CAN_FMR_FINIT;
    
    // Normal 모드
    CAN1->MCR &= ~CAN_MCR_INRQ;
    while (CAN1->MSR & CAN_MSR_INAK);
    
    // RX 인터럽트 활성화
    CAN1->IER |= CAN_IER_FMPIE0;
    NVIC_EnableIRQ(USB_LP_CAN1_RX0_IRQn);
}
```

---

## 메시지 처리

```c
void IAP_Process(void) {
    if (!g_msg_pending) return;
    g_msg_pending = 0;
    
    uint8_t cmd = g_rx_data[0];
    
    switch (g_iap_state) {
    case STATE_IDLE:
        if (cmd == CMD_CONNECT) Handle_Connect();
        break;
        
    case STATE_WAIT_KEY:
        if (cmd == CMD_KEY) Handle_Key();
        break;
        
    case STATE_CONNECTED:
        if (cmd == CMD_SIZE) Handle_Size();
        break;
        
    case STATE_WAIT_DATA:
        Handle_Data();
        break;
        
    case STATE_VERIFY:
        if (cmd == CMD_VERIFY) Handle_Verify();
        break;
        
    case STATE_COMPLETE:
        if (cmd == CMD_JUMP) Handle_Jump();
        break;
    }
}
```

---

## 핵심 핸들러

```c
void Handle_Connect(void) {
    uint8_t resp[8];
    resp[0] = RSP_CONNECT;
    memcpy(&resp[1], g_fw_check, 4);
    memcpy(&resp[5], g_fw_date, 3);
    
    CAN_Send(resp, 8);
    g_iap_state = STATE_WAIT_KEY;
}

void Handle_Key(void) {
    uint32_t rx_key = *(uint32_t *)&g_rx_data[1];
    uint32_t calc_key = CalcKey();
    
    if (rx_key == calc_key) {
        CAN_Send_Byte(RSP_KEY);
        g_iap_state = STATE_CONNECTED;
    } else {
        CAN_Send_Byte(RSP_ERROR);
        g_iap_state = STATE_IDLE;
    }
}
```

---

다음 글에서 Flash 드라이버 복원.

[#29 - Flash 드라이버 복원](/posts/ghidra/ghidra-stm32-re-29/)
