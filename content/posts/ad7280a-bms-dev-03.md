---
title: "AD7280A BMS 개발 삽질기 #3 - 첫 SPI 통신 성공까지"
date: 2024-12-07
draft: false
tags: ["BMS", "AD7280A", "임베디드", "SPI", "삽질"]
categories: ["BMS 개발"]
summary: "SPI Mode 1이 뭔지도 모르고 시작했다가 하루를 날렸다."
---

## 지난 글 요약

[지난 글](/posts/ad7280a-bms-dev-02/)에서 저항 분배기로 테스트 환경을 구성했다. 이제 진짜 SPI 통신을 해볼 차례다.

## AD7280A SPI 특징

데이터시트를 펼치면 SPI 관련 내용이 나온다. 핵심만 정리하면:

| 항목 | 값 |
|------|-----|
| 클럭 속도 | 최대 1MHz |
| SPI 모드 | **Mode 1** (CPOL=0, CPHA=1) |
| 데이터 길이 | **32비트** |
| 바이트 순서 | MSB First |
| CRC | 8비트 CRC 필수 |

여기서 삽질 포인트가 세 개나 숨어있었다.

## 삽질 1: SPI Mode

대부분의 SPI 디바이스는 Mode 0 (CPOL=0, CPHA=0)을 쓴다. 그래서 아무 생각 없이 Mode 0으로 설정했다.

**결과: 0xFFFFFFFF만 읽힘.**

데이터시트를 다시 보니 AD7280A는 **Mode 1**이다. CPHA=1이라서 클럭의 두 번째 엣지에서 데이터를 샘플링한다.

```c
// 잘못된 설정
spi_set_format(SPI_PORT, 8, SPI_CPOL_0, SPI_CPHA_0, SPI_MSB_FIRST);

// 올바른 설정
spi_set_format(SPI_PORT, 8, SPI_CPOL_0, SPI_CPHA_1, SPI_MSB_FIRST);
```

Mode 0과 Mode 1의 차이를 몰랐던 게 문제였다. 오실로스코프로 파형 찍어보니 클럭이랑 데이터 타이밍이 안 맞는 게 바로 보였다.

## 삽질 2: 32비트 전송

대부분의 MCU SPI 하드웨어는 8비트 또는 16비트 단위로 동작한다. AD7280A는 32비트를 한 번에 보내야 한다.

처음엔 이렇게 했다:

```c
// 잘못된 방식: CS를 4번 토글함
spi_write_blocking(spi, &byte1, 1);
spi_write_blocking(spi, &byte2, 1);
spi_write_blocking(spi, &byte3, 1);
spi_write_blocking(spi, &byte4, 1);
```

**결과: 역시 0xFFFFFFFF.**

AD7280A는 CS가 LOW인 동안 연속으로 32비트를 받아야 한다. CS를 중간에 토글하면 안 된다.

```c
// 올바른 방식: CS 한 번만 토글
uint32_t ad7280a_transfer_32bits(uint32_t data) {
    uint8_t tx[4], rx[4];
    
    tx[0] = (data >> 24) & 0xFF;
    tx[1] = (data >> 16) & 0xFF;
    tx[2] = (data >> 8) & 0xFF;
    tx[3] = data & 0xFF;

    gpio_put(PIN_CS, 0);  // CS LOW
    spi_write_read_blocking(SPI_PORT, tx, rx, 4);  // 4바이트 연속 전송
    gpio_put(PIN_CS, 1);  // CS HIGH

    return ((uint32_t)rx[0] << 24) | ((uint32_t)rx[1] << 16) |
           ((uint32_t)rx[2] << 8) | (uint32_t)rx[3];
}
```

## 삽질 3: CRC

AD7280A는 모든 통신에 8비트 CRC가 붙는다. CRC가 틀리면 명령을 무시한다.

처음엔 CRC 없이 보냈다.

**결과: 아무 반응 없음.**

데이터시트에 CRC 계산 공식이 있는데, 처음 보면 머리가 아프다. 다행히 Linux 커널 드라이버에 구현된 코드가 있어서 참고했다.

```c
// CRC 계산 (데이터시트 기반)
uint8_t ad7280a_calc_crc(uint32_t data) {
    uint8_t crc = 0;
    int i;
    
    // 21비트 데이터에 대해 CRC 계산
    for (i = 20; i >= 0; i--) {
        uint8_t bit = (data >> i) & 1;
        uint8_t xor_bit = (crc >> 7) ^ bit;
        crc = (crc << 1) & 0xFF;
        if (xor_bit)
            crc ^= 0x07;  // 다항식: x^8 + x^2 + x + 1
    }
    
    return crc;
}

// CRC 포함해서 32비트 명령 생성
uint32_t ad7280a_crc_write(uint32_t data) {
    uint8_t crc = ad7280a_calc_crc(data >> 11);
    return (data & 0xFFFFFF00) | (crc << 3) | 0x02;  // W 비트
}
```

CRC가 맞아야 비로소 AD7280A가 응답을 한다.

## 첫 통신 성공

세 가지 삽질을 해결하고 나니 드디어 응답이 왔다.

```c
void ad7280a_init(void) {
    // SPI 초기화 (1MHz, Mode 1)
    spi_init(SPI_PORT, 1000000);
    spi_set_format(SPI_PORT, 8, SPI_CPOL_0, SPI_CPHA_1, SPI_MSB_FIRST);
    
    // GPIO 설정
    gpio_init(PIN_CS);
    gpio_set_dir(PIN_CS, GPIO_OUT);
    gpio_put(PIN_CS, 1);
    
    gpio_init(PIN_CNVST);
    gpio_set_dir(PIN_CNVST, GPIO_OUT);
    gpio_put(PIN_CNVST, 1);
    
    gpio_init(PIN_PD);
    gpio_set_dir(PIN_PD, GPIO_OUT);
    gpio_put(PIN_PD, 1);  // PD HIGH = 정상 동작
    
    sleep_ms(1);  // 파워업 대기
    
    // Control LB 레지스터 설정
    uint8_t ctrl_lb = AD7280A_CTRL_LB_MUST_SET | 
                      AD7280A_CTRL_LB_LOCK_DEV_ADDR |
                      AD7280A_CTRL_LB_DAISY_CHAIN_RB_EN |
                      AD7280A_CTRL_LB_ACQ_TIME_1600ns;
    
    uint32_t cmd = ad7280a_crc_write(
        ((uint32_t)AD7280A_REG_CONTROL_LB << 21) |
        ((uint32_t)ctrl_lb << 13) |
        (1 << 12)  // All devices
    );
    ad7280a_transfer_32bits(cmd);
}
```

시리얼 모니터에 셀 전압 값이 찍히는 걸 보고 소리 질렀다.

## 디버깅 팁

SPI 통신이 안 될 때 확인할 것들:

1. **오실로스코프 필수** - 파형 안 보면 뭐가 문제인지 모름
2. **CS 타이밍 확인** - 32비트 전송 중 CS가 HIGH로 올라가면 안 됨
3. **클럭 속도** - 처음엔 100kHz 같은 느린 속도로 시작
4. **CRC 계산** - 데이터시트 예제로 먼저 검증

## 삽질 포인트 정리

1. **SPI Mode 1** - CPOL=0, CPHA=1. Mode 0으로 하면 안 됨
2. **32비트 연속 전송** - CS를 중간에 토글하면 안 됨
3. **CRC 필수** - CRC 없으면 AD7280A가 명령 무시

다음 글에서는 VIN5/VIN6에서 0만 읽히는 문제를 다룬다. 이것도 삽질이 좀 있었다.

---

## 참고 자료

- [AD7280A Datasheet](https://www.analog.com/media/en/technical-documentation/data-sheets/ad7280a.pdf)
- [Linux AD7280A Driver](https://github.com/torvalds/linux/blob/master/drivers/iio/adc/ad7280a.c)
- [Analog Devices no-OS](https://wiki.analog.com/resources/no-os)
