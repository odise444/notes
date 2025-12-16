---
title: "AD7280A BMS 개발기 번외 #2 - Linux 드라이버 읽다가 자괴감 든 날"
date: 2024-12-15
draft: false
tags: ["AD7280A", "BMS", "Linux", "IIO", "커널"]
categories: ["BMS 개발"]
series: ["AD7280A BMS 개발"]
summary: "삽질하다 지쳐서 리눅스 커널 코드를 열어봤다. 내가 3주 걸린 걸 200줄로 끝내놨더라."
---

CRC에서 3일 날리고, 레지스터 순서에서 또 3일 날리고.

"나만 이렇게 힘든 건가?"

문득 궁금해져서 검색해봤다. Linux 커널에 AD7280A 드라이버가 있었다.

```
drivers/iio/adc/ad7280a.c
```

2011년에 올라온 코드. 내가 고등학생일 때.

---

## CRC 비교

내가 3일 헤맸던 CRC.

내 코드:
```c
uint8_t calc_crc(uint32_t data) {
    uint8_t crc = 0;
    for (int i = 31; i >= 0; i--) {
        if ((crc ^ (data >> i)) & 0x01) {
            crc = (crc >> 1) ^ 0x2F;
        } else {
            crc >>= 1;
        }
    }
    return crc;
}
```

커널 코드:
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
```

룩업 테이블이었다. 초기화할 때 테이블 만들어놓고 바이트 단위로 찾아쓴다.

내 방식은 매번 32비트 전부 계산. 커널 방식은 테이블 참조 3번.

속도도 빠르고 디버깅도 쉽다. 이걸 왜 생각 못 했지.

---

## 버스트 읽기

내 코드:
```c
for (int dev = 0; dev < 4; dev++) {
    for (int cell = 0; cell < 6; cell++) {
        uint16_t raw = AD7280A_ReadCell(dev, cell);
        cells[dev * 6 + cell] = raw;
    }
}
```

24번 SPI 트랜잭션. 약 50ms.

커널 코드는 변환 한 번 시작하고 연속으로 24개 읽어온다. 1+24 트랜잭션, 약 5ms.

10배 빠르다.

---

## IIO 서브시스템

리눅스는 ADC를 IIO 프레임워크로 다룬다.

```bash
$ cat /sys/bus/iio/devices/iio:device0/in_voltage0_raw
2543
```

sysfs 파일 읽기만으로 셀 전압 확인 가능. 드라이버가 알아서 SPI 통신하고 변환해서 보여준다.

이런 추상화가 가능한 이유가 표준 인터페이스를 따르기 때문이다. 어떤 ADC든 같은 방식으로 읽을 수 있다.

---

## 내 드라이버에 적용

커널 코드에서 배운 것들을 적용했다.

1. CRC 룩업 테이블 - 초기화 시 생성
2. 버스트 읽기 - 변환 한 번에 전체 읽기
3. 매크로 정의 - 비트필드 가독성

```c
#define AD7280A_WRITE_DEV(d)    (((d) & 0x1F) << 27)
#define AD7280A_WRITE_REG(r)    (((r) & 0x3F) << 21)
#define AD7280A_WRITE_DATA(v)   (((v) & 0xFF) << 13)
```

매직 넘버 대신 매크로 쓰니까 코드 읽기가 훨씬 편해졌다.

---

## 교훈

바퀴를 재발명하지 마라. 먼저 검색해라.

커널 코드를 먼저 봤으면 2주는 아꼈다. 2011년부터 있던 코드인데 왜 검색을 안 했을까.

오픈소스 참고할 때 라이선스 확인은 필수. 커널 드라이버는 GPL이라 그대로 쓰면 안 되고, 구조랑 알고리즘만 참고했다.

---

다음은 중국산 BMS IC 비교. 가격이 1/5인데 품질은?

[번외 #3 - 중국산 BMS IC 비교](/posts/bms/ad7280a-bms-dev-b3/)
