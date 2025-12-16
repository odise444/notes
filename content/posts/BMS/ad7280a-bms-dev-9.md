---
title: "AD7280A BMS 개발기 #9 - 온도 센서 연결"
date: 2024-12-09
draft: false
tags: ["AD7280A", "BMS", "STM32", "NTC", "온도"]
categories: ["BMS 개발"]
series: ["AD7280A BMS 개발"]
summary: "보조 ADC에 NTC 서미스터를 연결해서 온도를 측정한다."
---

배터리 온도 측정은 필수다. 과열되면 화재 위험이 있다.

AD7280A에는 보조 ADC 6채널이 있다. 여기에 NTC 서미스터를 연결하면 온도를 측정할 수 있다.

---

NTC 연결 회로:

```
VDD (5V)
  │
  ├── 10kΩ (풀업)
  │
  ├──── AUX_ADC_IN ──── AD7280A
  │
 NTC (10kΩ @ 25°C)
  │
 GND
```

분압 회로다. NTC 저항이 온도에 따라 바뀌면 AUX_ADC 입력 전압이 바뀐다.

---

ADC 값을 온도로 변환하는 게 좀 복잡하다.

```c
// NTC B=3950, R25=10kΩ 기준
// 1. ADC → 전압
// 2. 전압 → NTC 저항
// 3. 저항 → 온도

float ADC_to_Temperature(uint16_t adc_raw) {
    // ADC 전압 (AD7280A AUX는 0~5V, 12비트)
    float voltage = (float)adc_raw / 4096.0f * 5.0f;
    
    // NTC 저항 계산 (분압 공식)
    // V = VDD * Rntc / (Rpu + Rntc)
    // Rntc = Rpu * V / (VDD - V)
    float r_ntc = 10000.0f * voltage / (5.0f - voltage);
    
    // Steinhart-Hart 또는 B-equation
    // T = 1 / (1/T0 + 1/B * ln(R/R0))
    float t_kelvin = 1.0f / (1.0f/298.15f + (1.0f/3950.0f) * logf(r_ntc/10000.0f));
    
    return t_kelvin - 273.15f;  // 섭씨로 변환
}
```

처음에 Steinhart-Hart 계수를 잘못 넣어서 온도가 100도 넘게 나왔다. NTC 데이터시트에서 B값 확인이 중요하다.

---

실제로는 룩업 테이블을 쓰는 게 더 빠르다.

```c
// 10kΩ NTC, B=3950, 풀업 10kΩ, VDD=5V
// ADC raw → 온도 (0.1°C 단위)
const int16_t ntc_table[64] = {
    -400, -350, -300, -260, -220, -180, -150, -120,
    -90, -60, -40, -10, 20, 50, 70, 100,
    // ... 생략
};

int16_t ADC_to_Temp_Fast(uint16_t adc_raw) {
    uint8_t idx = adc_raw >> 6;  // 12비트 → 6비트 인덱스
    return ntc_table[idx];
}
```

MCU에서 log() 계산은 느리다. 룩업 테이블이 훨씬 효율적.

---

24셀 BMS에서 온도 센서를 몇 개 달아야 하나?

최소 4개 (각 모듈당 1개), 권장 8개 (위아래 각 1개). 우리는 6개 달았다.

---

다음 글에서 밸런싱 기능을 다룬다.

[#10 - 패시브 밸런싱](/posts/bms/ad7280a-bms-dev-10/)
