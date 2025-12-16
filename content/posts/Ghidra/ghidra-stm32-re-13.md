---
title: "Ghidra STM32 역분석 #13 - CAN 수신 핸들러 분석"
date: 2024-12-13
draft: false
tags: ["Ghidra", "STM32", "리버스엔지니어링", "역분석", "CAN", "인터럽트"]
categories: ["역분석"]
series: ["Ghidra 역분석"]
summary: "CAN 메시지가 도착하면 무슨 일이? 수신 핸들러를 파헤친다."
---

Vector Table에서 CAN1_RX0 핸들러 주소를 찾았다. `0x08000094` → `FUN_08003100`.

---

## 수신 핸들러

```c
void CAN1_RX0_IRQHandler(void) {
    // FIFO에 메시지 있는지 확인
    while ((*(uint32_t *)0x4000640C & 0x03) != 0) {
        // ID 읽기
        uint32_t id = (*(uint32_t *)0x400065B0 >> 21) & 0x7FF;
        
        // DLC 읽기
        uint32_t dlc = *(uint32_t *)0x400065B4 & 0x0F;
        
        // 데이터 읽기
        uint32_t data_l = *(uint32_t *)0x400065B8;
        uint32_t data_h = *(uint32_t *)0x400065BC;
        
        // FIFO 해제
        *(uint32_t *)0x4000640C = 0x20;
        
        // 메시지 처리
        if (id == 0x5FF) {
            ProcessIAPMessage(data_l, data_h, dlc);
        }
    }
}
```

---

## ID 필터링

`0x5FF`만 처리한다. 스니핑에서 봤던 PC→BMS 방향 ID다.

---

## 메시지 처리 함수

`ProcessIAPMessage` 안에 IAP 프로토콜 로직이 있다:

```c
void ProcessIAPMessage(uint32_t data_l, uint32_t data_h, uint8_t dlc) {
    uint8_t cmd = data_l & 0xFF;  // 첫 바이트가 명령
    
    switch (cmd) {
        case 0x30:  // Connection Request
            SendConnectionResponse();
            break;
        case 0x31:  // Key Calculation
            CheckConnectionKey(data_l);
            break;
        case 0x32:  // Size Info
            ReceiveSizeInfo(data_l, data_h);
            break;
        case 0x33:  // Data
            ReceiveData(data_l, data_h);
            break;
        case 0x34:  // Verify
            VerifyAndJump();
            break;
    }
}
```

헤더 파일에서 봤던 명령 코드들이다!

---

다음 글에서 IAP 프로토콜 상세 분석.

[#14 - IAP 프로토콜 분석](/posts/ghidra/ghidra-stm32-re-14/)
