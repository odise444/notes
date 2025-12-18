---
title: "AD7280A BMS 개발기 #5 - 프레임 구조, 32비트의 비밀"
date: 2024-01-19
draft: false
tags: ["AD7280A", "BMS", "STM32", "SPI", "프레임"]
categories: ["BMS 개발"]
series: ["AD7280A BMS 개발"]
summary: "AD7280A는 32비트 프레임으로 통신한다. 각 비트가 뭔지 알아야 읽고 쓸 수 있다."
---

AD7280A 통신은 32비트 프레임 단위다. SPI로 4바이트씩 주고받는다.

이 32비트 안에 디바이스 주소, 레지스터 주소, 데이터, CRC가 다 들어간다.

---

## 쓰기 프레임

```
Bit 31-27: Device Address (5bit)
Bit 26-21: Register Address (6bit)
Bit 20-13: Data (8bit)
Bit 12:    Write bit (1=Write)
Bit 11-3:  CRC (8bit) + Reserved
Bit 2-0:   Reserved
```

예를 들어 Device 0의 CTRL_LB(0x0F)에 0x10을 쓰려면:

```
Device Addr: 00000 (0)
Register:    001111 (0x0F)
Data:        00010000 (0x10)
Write bit:   1
CRC:         계산해야 함
```

이걸 조합하면:

```c
uint32_t frame = 0;
frame |= (0 & 0x1F) << 27;      // Device 0
frame |= (0x0F & 0x3F) << 21;   // Register 0x0F
frame |= (0x10 & 0xFF) << 13;   // Data 0x10
frame |= (1 << 12);              // Write bit
// CRC는 나중에
```

---

## 읽기 프레임

읽을 때는 두 단계다.

1. 읽기 명령 전송 (어떤 레지스터 읽을지)
2. 더미 전송하고 응답 수신

```
읽기 명령 프레임:
Bit 31-27: Device Address
Bit 26-21: Register Address
Bit 20-13: Don't care (보통 0)
Bit 12:    Read bit (0=Read)
Bit 11-3:  CRC
Bit 2-0:   Reserved
```

응답 프레임:

```
Bit 31-27: Device Address (응답한 디바이스)
Bit 26-21: Register Address
Bit 20-13: Data (읽은 값)
Bit 12:    Write Ack 또는 Read data
Bit 11-3:  CRC
Bit 2-0:   Reserved
```

---

## 코드로 구현

```c
// 쓰기 프레임 생성
uint32_t AD7280A_BuildWriteFrame(uint8_t dev, uint8_t reg, uint8_t data) {
    uint32_t frame = 0;
    
    frame |= ((uint32_t)(dev & 0x1F)) << 27;
    frame |= ((uint32_t)(reg & 0x3F)) << 21;
    frame |= ((uint32_t)(data & 0xFF)) << 13;
    frame |= (1 << 12);  // Write bit
    
    // CRC 계산 (다음 글에서)
    uint8_t crc = AD7280A_CalcCRC(frame >> 11);
    frame |= (crc << 3);
    
    return frame;
}

// 읽기 프레임 생성
uint32_t AD7280A_BuildReadFrame(uint8_t dev, uint8_t reg) {
    uint32_t frame = 0;
    
    frame |= ((uint32_t)(dev & 0x1F)) << 27;
    frame |= ((uint32_t)(reg & 0x3F)) << 21;
    // Data는 don't care
    // Write bit = 0 (읽기)
    
    uint8_t crc = AD7280A_CalcCRC(frame >> 11);
    frame |= (crc << 3);
    
    return frame;
}
```

---

## 응답 파싱

응답 프레임에서 데이터 추출:

```c
// 응답에서 데이터 추출
uint8_t AD7280A_ParseData(uint32_t frame) {
    return (frame >> 13) & 0xFF;
}

// 응답에서 디바이스 주소 추출
uint8_t AD7280A_ParseDevice(uint32_t frame) {
    return (frame >> 27) & 0x1F;
}

// 응답에서 레지스터 주소 추출
uint8_t AD7280A_ParseRegister(uint32_t frame) {
    return (frame >> 21) & 0x3F;
}
```

---

## 삽질: 비트 순서

처음에 비트 위치를 잘못 이해했다.

데이터시트 그림이 이렇게 생겼다:

```
D31 D30 D29 D28 D27 | D26 D25 ... D21 | D20 ... D13 | ...
     Device Addr    |  Register Addr  |    Data     |
```

MSB가 D31이고 LSB가 D0이다. 근데 나는 반대로 생각해서 한참 헤맸다.

```c
// 틀린 코드
frame |= (dev & 0x1F) << 0;  // 잘못된 위치

// 맞는 코드
frame |= (dev & 0x1F) << 27;  // MSB 쪽
```

---

## 정리

- 쓰기: [DevAddr 5bit][RegAddr 6bit][Data 8bit][W=1][CRC 8bit][Rsv 3bit]
- 읽기: [DevAddr 5bit][RegAddr 6bit][X 8bit][R=0][CRC 8bit][Rsv 3bit]
- 응답: [DevAddr 5bit][RegAddr 6bit][Data 8bit][?][CRC 8bit][Rsv 3bit]

CRC가 틀리면 AD7280A가 명령을 무시한다. 그래서 CRC 계산이 중요하다.

---

다음은 CRC 계산. 여기서 3일 날렸다.

[#6 - CRC 계산](/posts/bms/ad7280a-bms-dev-6/)
