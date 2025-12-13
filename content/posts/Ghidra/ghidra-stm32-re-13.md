---
title: "Ghidra로 STM32 부트로더 역분석한 썰 #13 - CAN 수신 핸들러 분석"
date: 2024-12-13
draft: false
tags: ["Ghidra", "STM32", "리버스엔지니어링", "역분석", "CAN", "인터럽트"]
categories: ["역분석"]
summary: "CAN 메시지가 도착하면 무슨 일이? 수신 핸들러를 역분석해서 메시지 처리 로직을 파헤친다."
---

## 지난 글 요약

[Part 3](/posts/ghidra/ghidra-stm32-re-12/)에서 주변장치 초기화를 완료했다. RCC, GPIO, CAN, Flash. 이제 **실제 동작 로직**을 분석할 차례. CAN 수신 핸들러부터 시작하자.

## CAN 인터럽트 찾기

### Vector Table에서 핸들러 찾기

STM32F103의 CAN 인터럽트:

| IRQ# | 벡터 주소 | 이름 |
|------|-----------|------|
| 19 | 0x08000090 | USB_HP_CAN1_TX |
| 20 | 0x08000094 | USB_LP_CAN1_RX0 |
| 21 | 0x08000098 | CAN1_RX1 |
| 22 | 0x0800009C | CAN1_SCE |

**FIFO 0 수신 인터럽트**가 핵심: 0x08000094

Ghidra에서 확인:
```
08000094: .word 0x08003100   ; CAN1_RX0_IRQHandler
```

`FUN_08003100`이 수신 핸들러!

## 수신 핸들러 디컴파일

```c
void FUN_08003100(void) {
    uint32_t rf0r;
    uint32_t id;
    uint32_t dlc;
    uint32_t data[2];
    
    // FIFO 0에 메시지 있는지 확인
    rf0r = *(uint32_t *)0x4000640C;  // CAN_RF0R
    
    while ((rf0r & 0x03) != 0) {  // FMP0 != 0
        // ID 읽기 (Standard ID)
        id = (*(uint32_t *)0x400065B0 >> 21) & 0x7FF;  // RI0R
        
        // DLC 읽기
        dlc = *(uint32_t *)0x400065B4 & 0x0F;  // RDT0R
        
        // 데이터 읽기
        data[0] = *(uint32_t *)0x400065B8;  // RDL0R
        data[1] = *(uint32_t *)0x400065BC;  // RDH0R
        
        // FIFO 해제
        *(uint32_t *)0x4000640C = 0x20;  // RFOM0 = 1
        
        // 메시지 처리
        FUN_08003200(id, (uint8_t *)data, dlc);
        
        // 다음 메시지 확인
        rf0r = *(uint32_t *)0x4000640C;
    }
}
```

## CAN RX 레지스터 분석

### RI0R (Receive Identifier Register)

```
RI0R = 0x400065B0

Bit [31:21] STID  - Standard Identifier
Bit [20:3]  EXID  - Extended Identifier (not used)
Bit 2       IDE   - Identifier Extension (0=Standard)
Bit 1       RTR   - Remote Transmission Request
```

코드 해석:
```c
id = (*(uint32_t *)0x400065B0 >> 21) & 0x7FF;
// STID 필드 추출 (11비트)
```

### RDT0R (Receive Data Length)

```
RDT0R = 0x400065B4

Bit [3:0]   DLC   - Data Length Code (0~8)
Bit [7:4]   Reserved
Bit [15:8]  FMI   - Filter Match Index
Bit [31:16] TIME  - Message Timestamp
```

### RDL0R / RDH0R (Receive Data)

```
RDL0R = 0x400065B8  - Data bytes 0~3
RDH0R = 0x400065BC  - Data bytes 4~7

Little Endian:
RDL0R = [Data0] [Data1] [Data2] [Data3]
RDH0R = [Data4] [Data5] [Data6] [Data7]
```

## 메시지 처리 함수

`FUN_08003200` 분석:

```c
void FUN_08003200(uint32_t id, uint8_t *data, uint32_t dlc) {
    // ID 필터링
    if (id != 0x5FF) {
        return;  // 0x5FF만 처리
    }
    
    // 첫 바이트가 명령 코드
    uint8_t cmd = data[0];
    
    // 전역 변수에 저장
    *(uint8_t *)0x20000200 = cmd;           // g_rx_cmd
    *(uint32_t *)0x20000204 = dlc;          // g_rx_dlc
    
    // 데이터 복사
    for (int i = 0; i < dlc; i++) {
        *(uint8_t *)(0x20000208 + i) = data[i];  // g_rx_data[]
    }
    
    // 수신 플래그 설정
    *(uint8_t *)0x20000210 = 1;             // g_rx_flag
}
```

**발견한 전역 변수**:

| 주소 | 이름 | 용도 |
|------|------|------|
| 0x20000200 | g_rx_cmd | 수신된 명령 코드 |
| 0x20000204 | g_rx_dlc | 데이터 길이 |
| 0x20000208 | g_rx_data[8] | 수신 데이터 |
| 0x20000210 | g_rx_flag | 수신 완료 플래그 |

## 메인 루프에서 처리

main() 함수에서 플래그 확인:

```c
void FUN_08002800(void) {  // main
    // 초기화...
    
    while (1) {
        // 수신 플래그 확인
        if (*(uint8_t *)0x20000210 != 0) {
            // 메시지 처리
            FUN_08003300();
            
            // 플래그 클리어
            *(uint8_t *)0x20000210 = 0;
        }
        
        // 타임아웃 체크
        FUN_08003400();
    }
}
```

**인터럽트 → 플래그 → 메인 루프 처리** 패턴!

```
┌─────────────────────────────────────────────────────┐
│                    Interrupt                        │
│  ┌─────────────┐     ┌──────────────┐              │
│  │ CAN RX IRQ  │────►│ Set g_rx_flag│              │
│  └─────────────┘     └──────────────┘              │
│                                                     │
├─────────────────────────────────────────────────────┤
│                    Main Loop                        │
│  ┌─────────────┐     ┌──────────────┐              │
│  │ Check flag  │────►│ Process msg  │              │
│  └─────────────┘     └──────────────┘              │
│         ▲                    │                      │
│         └────────────────────┘                      │
└─────────────────────────────────────────────────────┘
```

## 실제 메시지 처리 함수

`FUN_08003300` (명령 디스패처):

```c
void FUN_08003300(void) {
    uint8_t cmd = *(uint8_t *)0x20000200;
    uint8_t *data = (uint8_t *)0x20000208;
    
    switch (cmd) {
    case 0x30:
        FUN_08003500();  // Connection 요청
        break;
        
    case 0x31:
        FUN_08003600(data);  // Calc 값 수신
        break;
        
    case 0x32:
        FUN_08003700(data);  // 크기 응답
        break;
        
    case 0x33:
        FUN_08003800(data);  // 페이지 시작
        break;
        
    case 0x34:
        FUN_08003900();  // 페이지 끝
        break;
        
    case 0x35:
        FUN_08003A00();  // 전송 완료
        break;
        
    case 0x36:
        FUN_08003B00(data);  // 검증 결과
        break;
        
    default:
        // 무시
        break;
    }
}
```

**0x30 시리즈 = PC → BMS 명령!**

## CAN 송신 함수 찾기

응답을 보내는 함수 찾기:

```c
void FUN_08003C00(uint32_t id, uint8_t *data, uint8_t len) {
    uint32_t tsr;
    uint8_t mailbox;
    
    // 빈 메일박스 찾기
    tsr = *(uint32_t *)0x40006408;  // CAN_TSR
    
    if (tsr & 0x04000000) {         // TME0
        mailbox = 0;
    } else if (tsr & 0x08000000) {  // TME1
        mailbox = 1;
    } else if (tsr & 0x10000000) {  // TME2
        mailbox = 2;
    } else {
        return;  // 모두 사용 중
    }
    
    // 메일박스 주소 계산
    uint32_t mb_base = 0x40006580 + mailbox * 0x10;
    
    // ID 설정
    *(uint32_t *)(mb_base + 0) = (id << 21);  // TIxR
    
    // DLC 설정
    *(uint32_t *)(mb_base + 4) = len;  // TDTxR
    
    // 데이터 설정
    *(uint32_t *)(mb_base + 8) = 
        data[0] | (data[1] << 8) | (data[2] << 16) | (data[3] << 24);
    *(uint32_t *)(mb_base + 12) = 
        data[4] | (data[5] << 8) | (data[6] << 16) | (data[7] << 24);
    
    // 전송 요청
    *(uint32_t *)(mb_base + 0) |= 0x01;  // TXRQ
}
```

**0x5FE로 응답 송신 발견!**

```c
// Connection 요청 응답
void FUN_08003500(void) {
    uint8_t response[8];
    
    response[0] = 0x40;  // 응답 코드
    response[1] = *(uint8_t *)0x20000100;  // FwChk[0]
    response[2] = *(uint8_t *)0x20000101;  // FwChk[1]
    response[3] = *(uint8_t *)0x20000102;  // FwChk[2]
    response[4] = *(uint8_t *)0x20000103;  // FwChk[3]
    response[5] = *(uint8_t *)0x20000104;  // FwDate[0]
    response[6] = *(uint8_t *)0x20000105;  // FwDate[1]
    response[7] = *(uint8_t *)0x20000106;  // FwDate[2]
    
    FUN_08003C00(0x5FE, response, 8);  // BMS → PC
}
```

## 복원된 CAN 핸들러

```c
// can_handler.c

#define CAN_ID_PC_TO_BMS    0x5FF
#define CAN_ID_BMS_TO_PC    0x5FE

// 수신 버퍼
typedef struct {
    uint8_t  cmd;
    uint32_t dlc;
    uint8_t  data[8];
    volatile uint8_t flag;
} can_rx_buffer_t;

static can_rx_buffer_t g_rx_buffer;

// CAN RX0 인터럽트 핸들러
void CAN1_RX0_IRQHandler(void) {
    while (CAN1->RF0R & CAN_RF0R_FMP0) {
        // ID 체크
        uint32_t id = (CAN1->sFIFOMailBox[0].RIR >> 21) & 0x7FF;
        
        if (id == CAN_ID_PC_TO_BMS) {
            // 데이터 길이
            g_rx_buffer.dlc = CAN1->sFIFOMailBox[0].RDTR & 0x0F;
            
            // 데이터 복사
            uint32_t rdlr = CAN1->sFIFOMailBox[0].RDLR;
            uint32_t rdhr = CAN1->sFIFOMailBox[0].RDHR;
            
            g_rx_buffer.data[0] = rdlr & 0xFF;
            g_rx_buffer.data[1] = (rdlr >> 8) & 0xFF;
            g_rx_buffer.data[2] = (rdlr >> 16) & 0xFF;
            g_rx_buffer.data[3] = (rdlr >> 24) & 0xFF;
            g_rx_buffer.data[4] = rdhr & 0xFF;
            g_rx_buffer.data[5] = (rdhr >> 8) & 0xFF;
            g_rx_buffer.data[6] = (rdhr >> 16) & 0xFF;
            g_rx_buffer.data[7] = (rdhr >> 24) & 0xFF;
            
            g_rx_buffer.cmd = g_rx_buffer.data[0];
            g_rx_buffer.flag = 1;
        }
        
        // FIFO 해제
        CAN1->RF0R |= CAN_RF0R_RFOM0;
    }
}

// 메인 루프에서 호출
void CAN_ProcessMessages(void) {
    if (g_rx_buffer.flag) {
        IAP_ProcessCommand(g_rx_buffer.cmd, g_rx_buffer.data, g_rx_buffer.dlc);
        g_rx_buffer.flag = 0;
    }
}

// CAN 송신
void CAN_Transmit(uint32_t id, uint8_t *data, uint8_t len) {
    // 빈 메일박스 찾기
    uint8_t mailbox;
    if (CAN1->TSR & CAN_TSR_TME0) mailbox = 0;
    else if (CAN1->TSR & CAN_TSR_TME1) mailbox = 1;
    else if (CAN1->TSR & CAN_TSR_TME2) mailbox = 2;
    else return;
    
    // 설정 및 전송
    CAN1->sTxMailBox[mailbox].TIR = (id << 21);
    CAN1->sTxMailBox[mailbox].TDTR = len;
    CAN1->sTxMailBox[mailbox].TDLR = 
        data[0] | (data[1] << 8) | (data[2] << 16) | (data[3] << 24);
    CAN1->sTxMailBox[mailbox].TDHR = 
        data[4] | (data[5] << 8) | (data[6] << 16) | (data[7] << 24);
    CAN1->sTxMailBox[mailbox].TIR |= CAN_TI0R_TXRQ;
}
```

## Ghidra 라벨 정리

```
함수 → 이름
─────────────────────────────
FUN_08003100 → CAN1_RX0_IRQHandler
FUN_08003200 → CAN_StoreMessage
FUN_08003300 → IAP_ProcessCommand
FUN_08003C00 → CAN_Transmit

전역 변수 → 이름
─────────────────────────────
0x20000200 → g_rx_cmd
0x20000204 → g_rx_dlc
0x20000208 → g_rx_data
0x20000210 → g_rx_flag
```

## 삽질: volatile 누락

인터럽트에서 설정한 플래그를 메인에서 못 읽음:

```c
// 잘못된 코드
uint8_t g_rx_flag;  // 컴파일러가 최적화로 제거!

// 올바른 코드
volatile uint8_t g_rx_flag;  // 항상 메모리에서 읽기
```

## 삽질: FIFO 해제 순서

데이터 읽기 전에 해제하면 다음 메시지로 덮어씀:

```c
// 잘못된 순서
CAN1->RF0R |= CAN_RF0R_RFOM0;  // 해제 먼저
data = CAN1->sFIFOMailBox[0].RDLR;  // 이미 다음 데이터!

// 올바른 순서
data = CAN1->sFIFOMailBox[0].RDLR;  // 읽기 먼저
CAN1->RF0R |= CAN_RF0R_RFOM0;  // 그 다음 해제
```

## 정리

| 발견 내용 | 값 |
|-----------|-----|
| PC → BMS ID | 0x5FF |
| BMS → PC ID | 0x5FE |
| 명령 코드 | 0x30 시리즈 |
| 응답 코드 | 0x40 시리즈 |
| 처리 방식 | 인터럽트 + 플래그 + 폴링 |

**다음 글에서**: 명령 코드 체계 파악 - 0x30, 0x31, 0x32...의 의미.

---

## 시리즈 목차

**Part 4: CAN IAP 프로토콜 역분석편**
- **#13 - CAN 수신 핸들러 분석** ← 현재 글
- #14 - 명령 코드 체계 파악
- #15 - 상태 머신 복원
- #16 - Connection Key 알고리즘
- #17 - 페이지 전송 프로토콜
- #18 - CRC 검증 함수 분석

---

## 참고 자료

- [STM32F103 Reference Manual - bxCAN](https://www.st.com/resource/en/reference_manual/rm0008.pdf)
- [CAN Protocol Tutorial](https://www.kvaser.com/can-protocol-tutorial/)
