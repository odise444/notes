---
title: "AD7280A BMS 개발 삽질기 번외 #2 - Linux 드라이버를 읽다"
date: 2024-12-15
draft: false
tags: ["AD7280A", "BMS", "Linux", "IIO", "커널", "드라이버"]
categories: ["BMS 개발"]
series: ["AD7280A BMS 개발"]
summary: "삽질하다 지쳐서 리눅스 커널 코드를 열어봤다. 거기엔 내가 3주 걸린 걸 200줄로 끝낸 코드가 있었다."
---

## 왜 커널 코드를 봤나

CRC 계산에서 3일을 날리고, 레지스터 순서에서 또 3일을 날리고...

**"나만 이렇게 힘든 건가?"**

문득 궁금해졌다. 다른 사람들은 AD7280A를 어떻게 썼을까?

검색해보니 **Linux 커널에 AD7280A 드라이버가 있었다.**

```bash
$ find /usr/src/linux -name "*ad7280*"
drivers/iio/adc/ad7280a.c
```

**커널 개발자가 짠 코드.** 공짜 레퍼런스.

---

## 드라이버 구조

파일을 열어봤다.

```c
// drivers/iio/adc/ad7280a.c
// SPDX-License-Identifier: GPL-2.0
/*
 * AD7280A Lithium Ion Battery Monitoring System
 *
 * Copyright 2011 Analog Devices Inc.
 */
```

**2011년!** 내가 고등학생일 때 이미 드라이버가 있었다.

```c
#define AD7280A_NUM_CH          6
#define AD7280A_BITS            12
#define AD7280A_ACQ_TIME_US     250
#define AD7280A_CONV_TIME_US    500

struct ad7280_state {
    struct spi_device *spi;
    int slave_num;
    int scan_cnt;
    int readback_delay_us;
    unsigned char crc_tab[256];
    unsigned char ctrl_hb;
    unsigned char ctrl_lb;
    unsigned char cell_threshhigh;
    unsigned char cell_threshlow;
    unsigned char aux_threshhigh;
    unsigned char aux_threshlow;
    unsigned char cb_mask[AD7280A_MAX_CHAIN];
};
```

깔끔하다. 내 드라이버보다 훨씬.

---

## CRC 구현 비교

내가 3일 헤맸던 CRC. 커널은 어떻게 했을까?

**내 코드 (처음 버전, 틀림):**

```c
// 삽질 버전 - 비트 순서 틀림
uint8_t calc_crc(uint32_t data) {
    uint8_t crc = 0;
    for (int i = 31; i >= 0; i--) {  // MSB부터
        if ((crc ^ (data >> i)) & 0x01) {
            crc = (crc >> 1) ^ 0x2F;
        } else {
            crc >>= 1;
        }
    }
    return crc;
}
```

**커널 코드:**

```c
static void ad7280_crc8_build_table(unsigned char *crc_tab)
{
    unsigned char bit, crc;
    int cnt, i;

    for (cnt = 0; cnt < 256; cnt++) {
        crc = cnt;
        for (i = 0; i < 8; i++) {
            bit = crc & HIGHBIT;
            crc <<= 1;
            if (bit)
                crc ^= POLYNOM;
        }
        crc_tab[cnt] = crc;
    }
}

static unsigned char ad7280_calc_crc8(unsigned char *crc_tab,
                                       unsigned int val)
{
    unsigned char crc;

    crc = crc_tab[val >> 16 & 0xFF];
    crc = crc_tab[crc ^ (val >> 8 & 0xFF)];
    return crc_tab[crc ^ (val & 0xFF)];
}
```

**차이점:**

1. **룩업 테이블 사용** - 실시간 계산 대신 미리 계산된 테이블
2. **바이트 단위 처리** - 비트 단위보다 빠름
3. **초기화 시 테이블 생성** - probe()에서 한 번만

```c
// 드라이버 초기화에서
static int ad7280_probe(struct spi_device *spi)
{
    // ...
    ad7280_crc8_build_table(st->crc_tab);  // 테이블 생성
    // ...
}
```

**배운 점:** 룩업 테이블이 성능도 좋고 디버깅도 쉽다.

---

## 읽기/쓰기 구현

**내 코드 (복잡):**

```c
uint32_t AD7280A_BuildWriteFrame(uint8_t dev, uint8_t reg, uint8_t data) {
    uint32_t frame = 0;
    
    frame |= ((uint32_t)(dev & 0x1F)) << 27;
    frame |= ((uint32_t)(reg & 0x3F)) << 21;
    frame |= ((uint32_t)data) << 13;
    frame |= (1 << 12);  // Write bit
    
    uint8_t crc = CalcCRC(frame >> 11);
    frame |= ((uint32_t)crc) << 3;
    
    return frame;
}
```

**커널 코드 (매크로 활용):**

```c
#define AD7280A_DEVADDR_MASTER      0
#define AD7280A_DEVADDR_ALL         0x1F

/* Bits and Masks */
#define AD7280A_WRITE_DEVADDR(addr) (((addr) & 0x1F) << 27)
#define AD7280A_WRITE_ADDR(addr)    (((addr) & 0x3F) << 21)
#define AD7280A_WRITE_DATA(data)    (((data) & 0xFF) << 13)
#define AD7280A_WRITE_BIT           BIT(12)

static int ad7280_write(struct ad7280_state *st, unsigned int devaddr,
                        unsigned int addr, bool all, unsigned int val)
{
    unsigned int reg = AD7280A_WRITE_DEVADDR(devaddr) |
                       AD7280A_WRITE_ADDR(addr) |
                       AD7280A_WRITE_DATA(val) |
                       AD7280A_WRITE_BIT;

    if (all)
        reg |= AD7280A_WRITE_ALL;

    reg |= ad7280_calc_crc8(st->crc_tab, reg >> 11) << 3;

    return spi_write(st->spi, &reg, sizeof(reg));
}
```

**배운 점:**

- 매크로로 비트 필드 정의하면 가독성 상승
- `BIT(n)` 매크로 사용 (리눅스 커널 스타일)
- 함수 하나에 모든 케이스 처리

---

## 데이지체인 처리

이게 진짜 배울 게 많았다.

**내 코드 (삽질):**

```c
// 디바이스별로 개별 읽기
for (int dev = 0; dev < 4; dev++) {
    for (int cell = 0; cell < 6; cell++) {
        uint16_t raw = AD7280A_ReadCell(dev, cell);
        cells[dev * 6 + cell] = raw;
    }
}
// 느림, 비효율적
```

**커널 코드 (버스트 읽기):**

```c
static int ad7280_read_all_channels(struct ad7280_state *st,
                                     unsigned int cnt,
                                     unsigned int *array)
{
    int i, ret;
    unsigned int tmp, sum = 0;

    /* 모든 디바이스에 동시에 변환 시작 */
    ret = ad7280_write(st, AD7280A_DEVADDR_MASTER,
                       AD7280A_CONTROL_HB, 1,
                       AD7280A_CTRL_HB_CONV_START_CNVST |
                       st->ctrl_hb);
    if (ret)
        return ret;

    /* 변환 완료 대기 */
    udelay(AD7280A_CONV_TIME_US);

    /* 연속 읽기 모드로 전체 데이터 수신 */
    for (i = 0; i < cnt; i++) {
        ret = spi_read(st->spi, &tmp, sizeof(tmp));
        if (ret)
            return ret;

        /* CRC 검증 */
        if (ad7280_check_crc(st, tmp))
            return -EIO;

        array[i] = (tmp >> 13) & 0xFFF;  /* 12비트 데이터 추출 */
        sum += array[i];
    }

    return 0;
}
```

**핵심 차이:**

```
내 방식:                   커널 방식:
                          
Read Cell 0                Convert ALL
Read Cell 1                Read ALL (burst)
Read Cell 2                
...                        
Read Cell 23               
                          
24번 SPI 트랜잭션          1+24 SPI 트랜잭션
약 50ms                    약 5ms
```

**배운 점:** 버스트 읽기는 10배 빠르다.

---

## IIO 서브시스템 구조

리눅스는 ADC를 **IIO (Industrial I/O)** 프레임워크로 다룬다.

```c
static const struct iio_chan_spec ad7280_channels[] = {
    AD7280_CHAN(AD7280A_CELL_VOLTAGE_1, 0, "cell1"),
    AD7280_CHAN(AD7280A_CELL_VOLTAGE_2, 1, "cell2"),
    AD7280_CHAN(AD7280A_CELL_VOLTAGE_3, 2, "cell3"),
    AD7280_CHAN(AD7280A_CELL_VOLTAGE_4, 3, "cell4"),
    AD7280_CHAN(AD7280A_CELL_VOLTAGE_5, 4, "cell5"),
    AD7280_CHAN(AD7280A_CELL_VOLTAGE_6, 5, "cell6"),
    AD7280_CHAN(AD7280A_AUX_ADC_1, 6, "aux1"),
    AD7280_CHAN(AD7280A_AUX_ADC_2, 7, "aux2"),
    // ...
};
```

**sysfs로 접근:**

```bash
$ cat /sys/bus/iio/devices/iio:device0/in_voltage0_raw
2543

$ cat /sys/bus/iio/devices/iio:device0/in_voltage0_scale
0.976562
```

사용자 공간에서 파일 읽기만으로 셀 전압을 알 수 있다.

**이런 추상화가 가능한 이유:**

```c
static int ad7280_read_raw(struct iio_dev *indio_dev,
                            struct iio_chan_spec const *chan,
                            int *val, int *val2, long m)
{
    struct ad7280_state *st = iio_priv(indio_dev);
    int ret;

    switch (m) {
    case IIO_CHAN_INFO_RAW:
        ret = ad7280_read_channel(st, chan->address, val);
        if (ret)
            return ret;
        return IIO_VAL_INT;

    case IIO_CHAN_INFO_SCALE:
        *val = 1000;   /* mV */
        *val2 = 976562;  /* 0.976562 mV/LSB */
        return IIO_VAL_FRACTIONAL;
    }

    return -EINVAL;
}
```

표준 인터페이스를 따르면 **어떤 ADC든 같은 방식**으로 읽을 수 있다.

---

## Alert 처리

커널 드라이버의 인터럽트 처리:

```c
static irqreturn_t ad7280_event_handler(int irq, void *private)
{
    struct iio_dev *indio_dev = private;
    struct ad7280_state *st = iio_priv(indio_dev);
    unsigned int status;
    int ret;

    /* Alert 상태 읽기 */
    ret = ad7280_read(st, AD7280A_DEVADDR_MASTER,
                      AD7280A_ALERT, &status);
    if (ret)
        goto out;

    /* 이벤트 발생 */
    if (status & AD7280A_ALERT_OV)
        iio_push_event(indio_dev,
                       IIO_EVENT_CODE(IIO_VOLTAGE, 0, 0,
                                      IIO_EV_DIR_RISING,
                                      IIO_EV_TYPE_THRESH, 0, 0, 0),
                       iio_get_time_ns(indio_dev));

    /* Alert 클리어 */
    ad7280_write(st, AD7280A_DEVADDR_MASTER,
                 AD7280A_ALERT, 1, status);

out:
    return IRQ_HANDLED;
}
```

**threaded IRQ 사용:**

```c
ret = devm_request_threaded_irq(&spi->dev, spi->irq,
                                 NULL,
                                 ad7280_event_handler,
                                 IRQF_TRIGGER_LOW | IRQF_ONESHOT,
                                 indio_dev->name,
                                 indio_dev);
```

- `NULL`: hardirq 핸들러 없음
- `ad7280_event_handler`: threaded 핸들러
- `IRQF_ONESHOT`: 핸들러 완료 전까지 인터럽트 마스킹

**배운 점:** SPI 통신이 필요하면 threaded IRQ가 필수.

---

## 에러 처리

커널 코드의 에러 처리가 꼼꼼하다.

```c
static int ad7280_probe(struct spi_device *spi)
{
    struct iio_dev *indio_dev;
    struct ad7280_state *st;
    int ret;

    indio_dev = devm_iio_device_alloc(&spi->dev, sizeof(*st));
    if (!indio_dev)
        return -ENOMEM;

    st = iio_priv(indio_dev);
    spi_set_drvdata(spi, indio_dev);
    st->spi = spi;

    /* CRC 테이블 초기화 */
    ad7280_crc8_build_table(st->crc_tab);

    /* 체인 검색 - 디바이스 개수 확인 */
    ret = ad7280_chain_setup(st);
    if (ret < 0)
        return ret;

    st->slave_num = ret;
    dev_info(&spi->dev, "Found %d devices in chain\n", st->slave_num);

    /* 초기 설정 */
    ret = ad7280_write(st, AD7280A_DEVADDR_MASTER,
                       AD7280A_CONTROL_LB, 1,
                       AD7280A_CTRL_LB_ACQ_TIME(2) |
                       AD7280A_CTRL_LB_THERMISTOR);
    if (ret)
        return ret;

    /* IIO 디바이스 등록 */
    ret = devm_iio_device_register(&spi->dev, indio_dev);
    if (ret)
        return ret;

    return 0;
}
```

**모든 함수 호출 후 에러 체크.** 실패하면 즉시 반환.

`devm_*` 함수들은 디바이스가 제거될 때 자동으로 정리된다.

---

## 내 드라이버에 적용

커널 코드에서 배운 것들을 내 드라이버에 적용:

**1. CRC 룩업 테이블:**

```c
// 초기화 시 테이블 생성
void AD7280A_BuildCRCTable(void) {
    for (int i = 0; i < 256; i++) {
        uint8_t crc = i;
        for (int j = 0; j < 8; j++) {
            if (crc & 0x80)
                crc = (crc << 1) ^ 0x2F;
            else
                crc <<= 1;
        }
        g_crc_table[i] = crc;
    }
}
```

**2. 버스트 읽기:**

```c
HAL_StatusTypeDef AD7280A_ReadAllCells(uint16_t *cells) {
    // 모든 디바이스 동시 변환
    AD7280A_StartConversionAll();
    HAL_Delay(1);
    
    // 연속 읽기
    uint32_t rx_buf[24];
    HAL_SPI_Receive(&hspi1, (uint8_t*)rx_buf, 24 * 4, 100);
    
    // 데이터 추출
    for (int i = 0; i < 24; i++) {
        cells[i] = (rx_buf[i] >> 13) & 0xFFF;
    }
    
    return HAL_OK;
}
```

**3. 매크로 정의:**

```c
#define AD7280A_WRITE_DEV(d)    (((d) & 0x1F) << 27)
#define AD7280A_WRITE_REG(r)    (((r) & 0x3F) << 21)
#define AD7280A_WRITE_DATA(v)   (((v) & 0xFF) << 13)
#define AD7280A_WRITE_BIT       (1 << 12)
```

---

## 정리: 커널 코드에서 배운 것들

| 항목 | 내 코드 | 커널 코드 | 개선점 |
|------|---------|-----------|--------|
| CRC | 매번 계산 | 룩업 테이블 | 속도 10배 |
| 읽기 | 셀별 개별 | 버스트 읽기 | 속도 10배 |
| 비트필드 | 매직 넘버 | 매크로 | 가독성 |
| 에러 | 대충 | 꼼꼼히 | 안정성 |
| 인터럽트 | hardirq | threaded | SPI 사용 가능 |

**가장 큰 교훈:**

> "바퀴를 재발명하지 마라. 먼저 검색해라."

커널 코드를 먼저 봤으면 2주는 아꼈다.

**다음 글에서**: 중국산 BMS IC 비교 - 가성비 대안은 있을까?

---

## 시리즈 네비게이션

**번외편**
- [B1 - AD7280A 단종 이야기](/posts/bms/ad7280a-bms-dev-b1/)
- **B2 - Linux IIO 드라이버 분석** ← 현재 글
- B3 - 중국산 BMS IC 비교
- B4 - 14S→24S 변환기

---

## 참고 자료

- [Linux IIO Subsystem](https://www.kernel.org/doc/html/latest/driver-api/iio/index.html)
- [AD7280A Linux Driver Source](https://elixir.bootlin.com/linux/latest/source/drivers/iio/adc/ad7280a.c)
- [Writing IIO Drivers](https://www.kernel.org/doc/html/latest/driver-api/iio/intro.html)
