---
title: "AD7280A BMS 개발기 #5 - 32비트 프레임 구조"
date: 2024-12-05
draft: false
tags: ["AD7280A", "BMS", "STM32", "SPI", "프레임"]
categories: ["BMS 개발"]
series: ["AD7280A BMS 개발"]
summary: "32비트 안에 주소, 데이터, CRC가 다 들어간다. 비트 조작이 좀 필요하다."
---

AD7280A 통신은 32비트 프레임 단위다. 8비트 SPI로 4번 전송하면 된다.

문제는 이 32비트 안에 여러 필드가 packed 되어있다는 거다.

---

쓰기 프레임 구조:

```
[31:27] Device Address (5비트)
[26:21] Register Address (6비트)
[20:13] Data (8비트)
[12]    Write bit (1=쓰기)
[11:3]  Reserved (항상 0x10)
[2:0]   CRC (3비트)

예시: Device 0, Register 0x0D에 0x10 쓰기
Device Addr: 00000 (5비트)
Register:    001101 (6비트) 
Data:        00010000 (8비트)
Write:       1
Reserved:    000010000
CRC:         ??? (계산 필요)

→ 0000_0001_1010_0010_0001_0000_1000_0XXX
```

이거 처음에 손으로 계산하다가 미칠 뻔했다. 함수로 만들어야 한다.

```c
uint32_t AD7280A_BuildWriteFrame(uint8_t dev, uint8_t reg, uint8_t data) {
    uint32_t frame = 0;
    
    frame |= ((uint32_t)(dev & 0x1F)) << 27;   // Device address
    frame |= ((uint32_t)(reg & 0x3F)) << 21;   // Register address
    frame |= ((uint32_t)data) << 13;           // Data
    frame |= (1 << 12);                        // Write bit
    frame |= (0x10 << 3);                      // Reserved bits
    
    // CRC는 bit[31:11] 기준으로 계산
    uint8_t crc = AD7280A_CalcCRC(frame >> 11);
    frame |= crc;
    
    return frame;
}
```

---

읽기 프레임은 좀 다르다.

먼저 읽을 레지스터 주소를 써야 한다.

```c
// 1. 읽을 레지스터 주소 설정
AD7280A_WriteReg(REG_READ, target_register);

// 2. 더미 데이터 보내면서 응답 받기
uint32_t response = AD7280A_Transfer(0x00000000);
```

응답 프레임 구조:

```
[31:27] Device Address
[26:21] Register Address  
[20:13] Data
[12]    Write Ack
[11:4]  Reserved
[3:0]   CRC (4비트, 쓰기와 다름!)
```

읽기 CRC는 4비트고 쓰기 CRC는 3비트다. 이거 데이터시트에서 찾느라 한참 걸렸다.

---

삽질했던 부분 중 하나가 비트 순서다.

STM32 SPI는 기본이 MSB first인데, AD7280A도 MSB first다. 근데 32비트를 4바이트로 쪼개서 보낼 때 순서가 헷갈린다.

```c
// 32비트 프레임을 4바이트로 전송
void AD7280A_SendFrame(uint32_t frame) {
    uint8_t tx[4];
    tx[0] = (frame >> 24) & 0xFF;  // MSB first
    tx[1] = (frame >> 16) & 0xFF;
    tx[2] = (frame >> 8) & 0xFF;
    tx[3] = frame & 0xFF;          // LSB last
    
    HAL_SPI_Transmit(&hspi1, tx, 4, 100);
}
```

처음에 tx[3]부터 넣어서 한참 안 됐다.

---

다음 글에서 CRC 계산을 다룬다. 이게 진짜 복잡하다.

[#6 - CRC 계산](/posts/bms/ad7280a-bms-dev-6/)
