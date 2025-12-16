---
title: "AD7280A BMS 개발기 #4 - 레지스터 맵 정복하기"
date: 2024-01-18
draft: false
tags: ["AD7280A", "BMS", "STM32", "레지스터"]
categories: ["BMS 개발"]
series: ["AD7280A BMS 개발"]
summary: "60페이지 데이터시트에서 진짜 필요한 레지스터는 10개 정도다."
---

데이터시트가 60페이지다. 처음엔 다 읽으려고 했는데 포기했다.

실제로 자주 쓰는 레지스터만 정리.

---

## 주요 레지스터

| 주소 | 이름 | 용도 |
|------|------|------|
| 0x00 | Cell Voltage 1 | 셀 1 전압 (읽기 전용) |
| 0x01 | Cell Voltage 2 | 셀 2 전압 |
| ... | ... | ... |
| 0x05 | Cell Voltage 6 | 셀 6 전압 |
| 0x06 | AUX ADC 1 | 보조 ADC (온도 등) |
| 0x0D | Self Test | 자가 진단 결과 |
| 0x0E | Control HB | 제어 레지스터 상위 |
| 0x0F | Control LB | 제어 레지스터 하위 |
| 0x10 | Cell OV | 과전압 임계값 |
| 0x11 | Cell UV | 저전압 임계값 |
| 0x14 | Cell Balance | 밸런싱 제어 |

셀 전압 읽고, 밸런싱 하고, 알람 설정하는 게 전부다. 10개 레지스터면 BMS 기본 기능 다 된다.

---

## Control 레지스터

CTRL_HB (0x0E):

```
Bit 7-6: Conversion input (00=Cell, 01=AUX, ...)
Bit 5: Conversion start method
Bit 4: Read type
Bit 3: Averaging enable
Bit 2-0: Reserved
```

CTRL_LB (0x0F):

```
Bit 7: Software reset
Bit 6: PD (Power Down)
Bit 5: Acquisition time
Bit 4: Thermistor enable
Bit 3-2: Increment device addr
Bit 1: Must set 1
Bit 0: ADDR_RD_EN
```

처음엔 이게 뭔 소린가 싶었는데, 필요할 때마다 찾아보면 된다.

자주 쓰는 설정:

```c
#define CTRL_HB_CONV_CELL    0x00  // 셀 전압 변환
#define CTRL_HB_CONV_AUX     0x40  // AUX ADC 변환
#define CTRL_HB_READ_ALL     0x10  // 전체 읽기

#define CTRL_LB_RESET        0x80  // 소프트웨어 리셋
#define CTRL_LB_THERM_EN     0x10  // 서미스터 활성화
#define CTRL_LB_MUST_SET     0x02  // 항상 1로 설정
```

---

## 셀 전압 레지스터

0x00~0x05가 Cell 1~6 전압.

12비트 값이 들어있다. mV로 변환하려면:

```c
// 12bit raw → mV
// 공식: V = (raw * 0.976mV) + 1000mV
uint16_t raw = ReadRegister(CELL_VOLTAGE_1);
float voltage_mv = raw * 0.976f + 1000.0f;
```

AD7280A는 1V~5V 범위를 측정한다. 리튬 셀이 보통 2.5V~4.2V니까 충분하다.

---

## 밸런싱 레지스터

CELL_BALANCE (0x14):

```
Bit 5: CB6 (Cell 6 밸런싱)
Bit 4: CB5
Bit 3: CB4
Bit 2: CB3
Bit 1: CB2
Bit 0: CB1
```

해당 비트 1로 설정하면 그 셀의 밸런싱 스위치가 ON.

```c
// Cell 1, 3 밸런싱 ON
WriteRegister(CELL_BALANCE, 0x05);  // 0b00000101
```

내부에 밸런싱 스위치가 있어서 외부 저항만 달면 된다. 편하다.

---

## Alert 레지스터

CELL_OV (0x10): 과전압 임계값
CELL_UV (0x11): 저전압 임계값

```c
// 임계값 계산
// 값 = (전압mV - 1000) / 6
// 예: 4.2V → (4200 - 1000) / 6 = 533

#define OV_4V2  533  // 4.2V
#define UV_2V5  250  // 2.5V
```

임계값 넘으면 Alert 핀이 Low로 떨어진다.

---

## 레지스터 정의

코드에서 쓰기 편하게 define:

```c
// Cell Voltage Registers
#define AD7280A_CELL_V1      0x00
#define AD7280A_CELL_V2      0x01
#define AD7280A_CELL_V3      0x02
#define AD7280A_CELL_V4      0x03
#define AD7280A_CELL_V5      0x04
#define AD7280A_CELL_V6      0x05

// AUX ADC
#define AD7280A_AUX_ADC1     0x06
#define AD7280A_AUX_ADC2     0x07

// Control
#define AD7280A_CTRL_HB      0x0E
#define AD7280A_CTRL_LB      0x0F

// Threshold
#define AD7280A_CELL_OV      0x10
#define AD7280A_CELL_UV      0x11

// Balance
#define AD7280A_CELL_BAL     0x14

// Alert
#define AD7280A_ALERT        0x18
```

---

다음은 프레임 구조. 32비트 안에 주소, 데이터, CRC가 어떻게 들어가는지.

[#5 - 프레임 구조](/posts/bms/ad7280a-bms-dev-5/)
