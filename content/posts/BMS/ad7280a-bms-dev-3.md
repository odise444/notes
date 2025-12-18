---
title: "AD7280A BMS 개발기 #3 - SPI 통신, 첫 응답 받기"
date: 2024-01-17
draft: false
tags: ["AD7280A", "BMS", "STM32", "SPI", "HAL"]
categories: ["BMS 개발"]
series: ["AD7280A BMS 개발"]
summary: "STM32 SPI 설정하고 AD7280A랑 첫 통신 시도. 삽질의 시작."
---

데이지체인 구조 이해했으니 이제 실제로 통신해보자.

STM32F103의 SPI1 사용. CubeMX에서 설정.

---

## SPI 설정

AD7280A SPI 스펙:
- 클럭: 최대 1MHz
- CPOL=0, CPHA=1 (Mode 1)
- MSB First
- 32비트 프레임

CubeMX 설정:

```
Mode: Full-Duplex Master
Prescaler: 64 (72MHz/64 = 1.125MHz → 좀 빠름)
CPOL: Low
CPHA: 2 Edge
First Bit: MSB
Data Size: 8 bit
```

데이터 사이즈가 8비트인데 AD7280A는 32비트 프레임이다. 4번 연속으로 보내면 된다.

---

## 핀 연결

```
STM32          AD7280A
PA5 (SCK)  --> SCLK
PA6 (MISO) <-- DOUT (마지막 디바이스)
PA7 (MOSI) --> DIN
PA4 (GPIO) --> CS
```

CS는 하드웨어 NSS 안 쓰고 GPIO로 제어했다. 타이밍 제어하기 편해서.

---

## 첫 통신 시도

간단하게 레지스터 읽기 시도.

```c
uint8_t tx[4] = {0x00, 0x00, 0x00, 0x00};
uint8_t rx[4] = {0};

HAL_GPIO_WritePin(CS_GPIO, CS_PIN, GPIO_PIN_RESET);
HAL_SPI_TransmitReceive(&hspi1, tx, rx, 4, 100);
HAL_GPIO_WritePin(CS_GPIO, CS_PIN, GPIO_PIN_SET);

printf("RX: %02X %02X %02X %02X\n", rx[0], rx[1], rx[2], rx[3]);
```

결과: `RX: 00 00 00 00`

아무것도 안 온다.

---

## 삽질 1: CPOL/CPHA

로직 분석기로 찍어봤다.

![SPI 파형](/imgs/ad7280a-bms-dev-03-1.png)

클럭은 나가는데 MISO가 항상 Low. AD7280A가 응답을 안 한다.

CPOL/CPHA 조합을 다 바꿔봤다.

```
Mode 0 (CPOL=0, CPHA=0): 안 됨
Mode 1 (CPOL=0, CPHA=1): 안 됨
Mode 2 (CPOL=1, CPHA=0): 안 됨
Mode 3 (CPOL=1, CPHA=1): 안 됨
```

다 안 된다. CPOL/CPHA 문제가 아니었다.

---

## 삽질 2: 전원

멀티미터로 전원 확인.

VDD: 4.8V (5V 레귤레이터에서)
DVDD: 0V

DVDD가 안 들어가고 있었다. 회로도 다시 보니까 DVDD는 별도로 연결해야 했다. VDD에서 자동으로 나오는 게 아니었다.

점퍼 연결하고 다시 시도.

결과: `RX: FF FF FF FF`

뭔가 온다!

---

## 삽질 3: 프레임 구조

FF가 오긴 하는데 이게 정상인지 모르겠다.

데이터시트 다시 읽어보니까, 읽기 명령을 제대로 보내야 응답이 온다. 그냥 0x00 보내면 안 된다.

AD7280A 프레임 구조:

```
[31:27] Device Address (5bit)
[26:21] Register Address (6bit)
[20:13] Data (8bit)
[12]    Write/Read bit
[11:3]  Reserved + CRC
[2:0]   Reserved
```

32비트 중에 CRC까지 계산해서 보내야 한다. CRC가 틀리면 무시한다.

이게 다음 글 주제다. CRC 계산에서 3일 날렸다.

---

## 일단 여기까지

SPI 하드웨어 설정은 됐다. AD7280A가 응답은 한다.

근데 제대로 된 명령을 보내려면 프레임 구조랑 CRC를 알아야 한다.

```c
// 지금까지 코드
void AD7280A_Init(void) {
    // SPI 이미 CubeMX에서 초기화됨
    // CS High (Idle)
    HAL_GPIO_WritePin(CS_GPIO, CS_PIN, GPIO_PIN_SET);
}

uint32_t AD7280A_Transfer(uint32_t tx) {
    uint8_t tx_buf[4], rx_buf[4];
    
    // 32bit를 8bit 4개로 분할 (Big Endian)
    tx_buf[0] = (tx >> 24) & 0xFF;
    tx_buf[1] = (tx >> 16) & 0xFF;
    tx_buf[2] = (tx >> 8) & 0xFF;
    tx_buf[3] = tx & 0xFF;
    
    HAL_GPIO_WritePin(CS_GPIO, CS_PIN, GPIO_PIN_RESET);
    HAL_SPI_TransmitReceive(&hspi1, tx_buf, rx_buf, 4, 100);
    HAL_GPIO_WritePin(CS_GPIO, CS_PIN, GPIO_PIN_SET);
    
    return (rx_buf[0] << 24) | (rx_buf[1] << 16) | 
           (rx_buf[2] << 8) | rx_buf[3];
}
```

---

다음은 레지스터 맵이랑 프레임 구조. 32비트 안에 뭐가 들어가는지.

[#4 - 레지스터 맵](/posts/bms/ad7280a-bms-dev-4/)
