---
title: "Ghidra STM32 역분석 #13 - CAN 수신 핸들러 분석"
date: 2024-12-13
draft: false
tags: ["Ghidra", "STM32", "리버스엔지니어링", "역분석", "CAN", "인터럽트"]
categories: ["역분석"]
series: ["Ghidra 역분석"]
summary: "CAN 메시지가 도착하면 무슨 일이? 수신 핸들러 분석."
---

Vector Table에서 CAN1_RX0 핸들러 주소를 찾았다: `0x08003100`

---

## 수신 핸들러

```c
void CAN1_RX0_IRQHandler(void) {
    uint32_t id, dlc, data[2];
    
    // FIFO에 메시지 있는지 확인
    while ((*(uint32_t *)0x4000640C & 0x03) != 0) {
        // ID 읽기
        id = (*(uint32_t *)0x400065B0 >> 21) & 0x7FF;
        
        // DLC 읽기
        dlc = *(uint32_t *)0x400065B4 & 0x0F;
        
        // 데이터 읽기
        data[0] = *(uint32_t *)0x400065B8;
        data[1] = *(uint32_t *)0x400065BC;
        
        // FIFO 해제
        *(uint32_t *)0x4000640C = 0x20;
        
        // 메시지 처리
        Process_CAN_Message(id, data, dlc);
    }
}
```

---

## ID 필터링

```c
void Process_CAN_Message(uint32_t id, uint32_t *data, uint8_t dlc) {
    if (id != 0x5FF) return;  // IAP 전용 ID
    
    uint8_t cmd = data[0] & 0xFF;
    
    switch (cmd) {
        case 0x30:  // Connection Request
            Handle_ConnRequest(data);
            break;
        case 0x31:  // Key Calculate
            Handle_KeyCalc(data);
            break;
        case 0x32:  // Size Response
            Handle_SizeRes(data);
            break;
        case 0x33:  // Data Frame
            Handle_DataFrame(data);
            break;
    }
}
```

헤더 파일에 있던 명령 코드들이 나온다. `0x30`, `0x31`, `0x32`, `0x33`.

---

## 핵심 발견

- 수신 ID: `0x5FF`
- 송신 ID: `0x5FE` (응답)
- 명령 코드: 첫 바이트

---

다음 글에서 IAP 프로토콜 상세 분석.

[#14 - IAP 프로토콜 분석](/posts/ghidra/ghidra-stm32-re-14/)
