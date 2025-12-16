---
title: "AD7280A BMS 개발기 #3 - SPI 통신 시작"
date: 2024-12-03
draft: false
tags: ["AD7280A", "BMS", "STM32", "SPI"]
categories: ["BMS 개발"]
series: ["AD7280A BMS 개발"]
summary: "STM32로 AD7280A랑 첫 통신. 생각보다 안 됐다."
---

STM32F103을 쓰기로 했다. 익숙하기도 하고 회사에 재고가 많아서.

CubeMX로 SPI 설정부터 했다.

```
SPI 설정:
- Mode: Full-Duplex Master
- Prescaler: 64 (클럭 느리게)
- CPOL: High
- CPHA: 2 Edge
- Data Size: 8 bit
- First Bit: MSB
```

AD7280A는 CPOL=1, CPHA=1이다. 데이터시트에 타이밍 다이어그램이 있는데 처음엔 뭔 소린지 모르겠어서 그냥 둘 다 시도해봤다.

---

첫 통신 시도.

```c
uint8_t tx_data[4] = {0x00, 0x00, 0x00, 0x00};
uint8_t rx_data[4] = {0};

HAL_GPIO_WritePin(CS_GPIO_Port, CS_Pin, GPIO_PIN_RESET);
HAL_SPI_TransmitReceive(&hspi1, tx_data, rx_data, 4, 100);
HAL_GPIO_WritePin(CS_GPIO_Port, CS_Pin, GPIO_PIN_SET);
```

결과: `rx_data = {0xFF, 0xFF, 0xFF, 0xFF}`

뭔가 잘못됐다. 0xFF면 보통 연결이 안 됐거나 설정이 틀린 거다.

---

오실로스코프로 파형을 찍어봤다.

CLK은 나가는데 MISO가 항상 High다. IC가 응답을 안 하는 거다.

확인해본 것들:
- 배선: 멀쩡함
- 전원: 5V 들어감
- CS: Low로 떨어짐

한참 헤매다가 데이터시트를 다시 읽었다. AD7280A는 전원 인가 후 초기화 시퀀스가 필요하다고 써있었다. 그냥 전원 넣고 바로 통신하면 안 된다.

---

초기화 시퀀스:

1. 전원 인가
2. 최소 1ms 대기
3. Software Reset 명령 전송
4. 다시 1ms 대기
5. 이제 통신 가능

```c
void AD7280A_Init(void) {
    HAL_Delay(10);  // 전원 안정화
    
    // Software Reset
    AD7280A_WriteReg(AD7280A_CONTROL_HB, 0x10);
    HAL_Delay(10);
    
    // 기본 설정
    AD7280A_WriteReg(AD7280A_CONTROL_LB, 0x00);
}
```

이렇게 하니까 드디어 응답이 왔다.

---

근데 응답 값이 이상했다.

```
보낸 값: 0x038101E2 (Read Device ID)
받은 값: 0x1C0800E4 (???)
```

나중에 알았는데 AD7280A 응답 포맷이 좀 특이하다. 32비트인데 그 안에 디바이스 주소, 레지스터 주소, 데이터, CRC가 다 들어있다.

이거 파싱하는 게 다음 과제다.

---

다음 글에서 레지스터 맵이랑 프레임 구조를 정리한다.

[#4 - 레지스터 맵](/posts/bms/ad7280a-bms-dev-4/)
