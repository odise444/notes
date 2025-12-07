---
title: "AD7280A BMS 개발 삽질기 #5 - Acquisition Time 삽질"
date: 2024-12-07
draft: false
tags: ["BMS", "AD7280A", "임베디드", "배터리", "삽질"]
categories: ["BMS 개발"]
summary: "비트 위치를 잘못 이해해서 하루를 날렸다. 데이터시트는 꼼꼼히 읽자."
---

## 지난 글 요약

[지난 글](/posts/ad7280a-bms-dev-04/)에서 VIN5/VIN6 채널 문제를 해결했다. 근데 또 다른 문제가 기다리고 있었다.

## 증상

셀 전압이 읽히긴 하는데, 가끔 튀는 값이 나온다. 특히 입력 임피던스가 높은 테스트 환경에서 심해졌다.

```
Cell 1: 4012 mV
Cell 2: 4008 mV
Cell 3: 2341 mV  ← 뭐지?
Cell 4: 4010 mV
Cell 5: 3876 mV
Cell 6: 4015 mV
```

재측정하면 또 정상으로 돌아온다. 간헐적 오류.

## Acquisition Time이 뭐길래

AD7280A는 SAR ADC 구조다. 입력 전압을 측정하려면:

1. 샘플링 커패시터에 전압 충전 (Acquisition)
2. ADC 변환 수행 (Conversion)

**Acquisition Time**은 1번 단계에서 커패시터가 입력 전압으로 충전되는 시간이다. 이 시간이 부족하면 커패시터가 완전히 충전되지 않아서 잘못된 값이 읽힌다.

## 데이터시트 계산 공식

```
tACQ = 10 × ((RSOURCE + R) × C)

RSOURCE: 외부 소스 임피던스 (100nF 커패시터 이후)
R: 내부 저항 = 300Ω
C: 샘플링 커패시터 = 15pF
```

표준 구성 (10kΩ + 100nF 필터)에서는 400ns면 충분하다. 근데 내 테스트 환경에서는 612Ω 저항을 썼고, 계산해보면:

```
tACQ = 10 × ((612 + 300) × 15pF)
     = 10 × (912 × 15 × 10⁻¹²)
     = 136.8ns
```

이론적으로 400ns면 충분한데... 왜 문제가 생겼을까?

## 삽질 1: 비트 위치 착각

Acquisition Time은 Control LB 레지스터의 **D6:D5** 비트로 설정한다. 처음에 D7:D6인 줄 알고 이렇게 정의했다:

```c
// 잘못된 정의 (D7:D6으로 착각)
#define AD7280A_CTRL_LB_ACQ_TIME_400NS    0x00  // OK, 우연히 맞음
#define AD7280A_CTRL_LB_ACQ_TIME_800NS    0x40  // 실제로는 1200ns!
#define AD7280A_CTRL_LB_ACQ_TIME_1200NS   0x80  // D7=1, 리셋 비트 건드림!
#define AD7280A_CTRL_LB_ACQ_TIME_1600NS   0xC0  // D7=1, 대참사!
```

**문제점:**
- 0x80은 D7=1인데, D7은 **Software Reset** 비트다
- 1600ns 설정한다고 0xC0 쓰면 IC가 리셋된다

```c
// 올바른 정의 (D6:D5)
#define AD7280A_CTRL_LB_ACQ_TIME_400NS    (0 << 5)  // 0x00
#define AD7280A_CTRL_LB_ACQ_TIME_800NS    (1 << 5)  // 0x20
#define AD7280A_CTRL_LB_ACQ_TIME_1200NS   (2 << 5)  // 0x40
#define AD7280A_CTRL_LB_ACQ_TIME_1600NS   (3 << 5)  // 0x60
```

## 삽질 2: Reserved 비트 무시

Control LB 레지스터의 D4 비트는 "Reserved; set to 1"이다. 처음에 이걸 무시하고 0으로 뒀다.

```c
// 잘못된 설정
uint8_t ctrl_lb = AD7280A_CTRL_LB_ACQ_TIME_1600NS |
                  AD7280A_CTRL_LB_LOCK_DEV_ADDR;
// D4 = 0, 문제 발생 가능

// 올바른 설정
uint8_t ctrl_lb = AD7280A_CTRL_LB_ACQ_TIME_1600NS |
                  (1 << 4) |  // Reserved D4 = 1 필수!
                  AD7280A_CTRL_LB_LOCK_DEV_ADDR;
```

데이터시트에 "must set to 1"이라고 적혀있으면 진짜로 1로 설정해야 한다.

## 삽질 3: 서미스터와 Acquisition Time

서미스터 종단 저항 옵션을 쓰면 Acquisition Time을 **반드시 1600ns**로 설정해야 한다.

> "The thermistor termination resistor option should only be used if the acquisition time of the AD7280A is set to its maximum value (1.6 μs) due to settling time requirements."

이걸 모르고 서미스터 옵션 켜고 400ns로 두면 온도 채널이 이상한 값을 뱉는다.

## Control LB 레지스터 정리

| 비트 | 이름 | 설명 |
|------|------|------|
| D7 | SWRST | Software Reset (1=리셋) |
| D6:D5 | ACQ_TIME | Acquisition Time 선택 |
| D4 | Reserved | **반드시 1로 설정** |
| D3 | THERM_EN | 서미스터 종단 저항 |
| D2 | LOCK_ADDR | 디바이스 주소 잠금 |
| D1 | INC_ADDR | 주소 증가 |
| D0 | DAISY_EN | 데이지체인 리드백 |

## Acquisition Time 선택 가이드

| 설정 | 시간 | 사용 조건 |
|------|------|----------|
| 00 | 400ns | 표준 구성 (10kΩ + 100nF) |
| 01 | 800ns | 높은 입력 임피던스 |
| 10 | 1200ns | 매우 높은 입력 임피던스 |
| 11 | 1600ns | 서미스터 사용 시 **필수** |

확실하지 않으면 그냥 1600ns로 설정하자. 변환 시간이 조금 길어지지만 안정성이 올라간다.

## 최종 코드

```c
// Control LB 레지스터 설정
uint8_t ctrl_lb = (AD7280A_ACQ_TIME_1600NS << 5) |  // D6:D5 = 11
                  (1 << 4) |                         // D4 = 1 (Reserved)
                  AD7280A_CTRL_LB_LOCK_DEV_ADDR |    // D2 = 1
                  AD7280A_CTRL_LB_DAISY_CHAIN_RB_EN; // D0 = 1

uint32_t cmd = ad7280a_crc_write(
    ((uint32_t)AD7280A_REG_CONTROL_LB << 21) |
    ((uint32_t)ctrl_lb << 13) |
    (1 << 12)  // All devices
);
ad7280a_transfer_32bits(cmd);
```

## 삽질 포인트 정리

1. **비트 위치 확인** - D7:D6 아니고 D6:D5!
2. **Reserved 비트** - "set to 1"이면 진짜 1로
3. **서미스터 = 1600ns** - 데이터시트에 명시됨
4. **안전하게 1600ns** - 확실하지 않으면 최대값 사용

다음 글에서는 Alert Register 설정과 미사용 채널 제외를 다룬다.

---

## 참고 자료

- [AD7280A Datasheet - Table 14: Control Register (p.28-29)](https://www.analog.com/media/en/technical-documentation/data-sheets/ad7280a.pdf)
- [AD7280A Datasheet - Acquisition Time (p.20-21)](https://www.analog.com/media/en/technical-documentation/data-sheets/ad7280a.pdf)
