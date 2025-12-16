---
title: "AD7280A BMS 개발기 #4 - 레지스터 맵"
date: 2024-12-04
draft: false
tags: ["AD7280A", "BMS", "STM32", "레지스터"]
categories: ["BMS 개발"]
series: ["AD7280A BMS 개발"]
summary: "레지스터가 30개 넘는다. 자주 쓰는 것만 정리했다."
---

AD7280A 레지스터가 30개가 넘는다. 데이터시트 보면 머리 아프다.

자주 쓰는 것만 추리면 이 정도다.

```c
// 자주 쓰는 레지스터
#define REG_CELL_VTG1       0x00  // Cell 1 전압
#define REG_CELL_VTG2       0x01  // Cell 2 전압
#define REG_CELL_VTG3       0x02  // Cell 3 전압
#define REG_CELL_VTG4       0x03  // Cell 4 전압
#define REG_CELL_VTG5       0x04  // Cell 5 전압
#define REG_CELL_VTG6       0x05  // Cell 6 전압
#define REG_AUX_ADC1        0x06  // 보조 ADC (온도)
#define REG_AUX_ADC2        0x07
#define REG_AUX_ADC3        0x08
#define REG_AUX_ADC4        0x09
#define REG_AUX_ADC5        0x0A
#define REG_AUX_ADC6        0x0B
#define REG_SELF_TEST       0x0C  // 자가진단 결과
#define REG_CONTROL_HB      0x0D  // 제어 레지스터 상위
#define REG_CONTROL_LB      0x0E  // 제어 레지스터 하위
#define REG_CELL_OVERVTG    0x0F  // 과전압 임계값
#define REG_CELL_UNDERVTG   0x10  // 저전압 임계값
#define REG_CELL_BALANCE    0x14  // 밸런싱 제어
#define REG_ALERT           0x18  // 알람 상태
```

---

레지스터 구조를 보면 특징이 있다.

0x00~0x0B: 측정 결과 (읽기 전용)
0x0D~0x1B: 제어/설정 (읽기/쓰기)

Cell 전압 레지스터는 12비트 값이 들어있다. 이걸 mV로 바꾸려면 계산이 필요하다.

```c
// AD7280A 전압 변환
// Vout = (raw / 4096) * 5V + 1V
// 측정 범위: 1V ~ 6V

float raw_to_mv(uint16_t raw) {
    return ((float)raw / 4096.0f) * 5000.0f + 1000.0f;
}
```

1V 오프셋이 있는 게 특이하다. 리튬 셀 전압이 보통 2.5V~4.2V니까 1V~6V 범위면 충분하긴 하다.

---

Control 레지스터가 좀 복잡하다.

```c
// CONTROL_HB (0x0D)
// Bit 7: CONV_START - 변환 시작
// Bit 6: READ_ALL - 연속 읽기 모드
// Bit 5: SW_RESET - 소프트웨어 리셋
// Bit 4: PD - 파워 다운
// Bit 3-0: CONV_INPUT - 변환 채널 선택

// CONTROL_LB (0x0E)  
// Bit 7-5: AQ_TIME - Acquisition time
// Bit 4: THERMISTOR - 서미스터 모드
// Bit 3-0: 예약
```

변환을 시작하려면 CONTROL_HB에 CONV_START 비트를 세팅해야 한다. 이게 One-shot이라 매번 세팅해줘야 한다.

---

처음에 레지스터 주소를 잘못 써서 엉뚱한 값이 나온 적 있다. 

0x00이 Cell 1인데 나는 습관적으로 0x01부터 시작했다. 그래서 Cell 2 값이 Cell 1에 찍히고 그랬다. 데이터시트 인덱스가 0부터 시작하는지 1부터 시작하는지 확인해야 한다.

---

다음 글에서 읽기/쓰기 프레임 구조를 정리한다. 이게 좀 복잡하다.

[#5 - 프레임 구조](/posts/bms/ad7280a-bms-dev-5/)
