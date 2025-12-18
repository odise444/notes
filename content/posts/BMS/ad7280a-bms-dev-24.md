---
title: "AD7280A BMS 개발기 #24 - 절연 모니터링"
date: 2024-02-07
draft: false
tags: ["AD7280A", "BMS", "STM32", "절연", "IMD"]
categories: ["BMS 개발"]
series: ["AD7280A BMS 개발"]
summary: "72V 배터리는 감전 위험이 있다. 절연 상태를 모니터링해야 한다."
---

72V면 직접 만지면 위험하다. 차체(GND)랑 배터리 사이 절연이 중요하다.

절연 불량이면 감전 사고, 누설 전류로 인한 화재 위험.

---

## 절연 저항

배터리 +/-와 차체(새시 GND) 사이 저항.

```
Battery+ ─────┐
              │
           [R_iso+]
              │
          Chassis ─── 새시 GND
              │
           [R_iso-]
              │
Battery- ─────┘
```

신품 절연 저항: 수 MΩ
위험 수준: 100kΩ 이하

---

## 측정 원리

배터리와 새시 사이에 알려진 저항 연결하고 전압 측정.

```
Battery+ ──[Rm]──┬── ADC
                 │
              Chassis
```

```c
// 측정 저항 Rm = 1MΩ
// ADC 전압 Vadc

// 절연 저항 계산
// R_iso = Rm × Vadc / (Vbat - Vadc)
```

---

## 간단한 구현

```c
#define R_MEASURE_OHM  1000000  // 1MΩ

typedef struct {
    uint32_t r_iso_pos;  // + 측 절연 저항
    uint32_t r_iso_neg;  // - 측 절연 저항
    bool fault;
} IsoMonitor_t;

void ISO_Measure(void) {
    // + 측 측정
    GPIO_SwitchToPositive();
    HAL_Delay(100);  // RC 안정화
    uint16_t v_pos = ADC_Read();
    
    // - 측 측정
    GPIO_SwitchToNegative();
    HAL_Delay(100);
    uint16_t v_neg = ADC_Read();
    
    // 절연 저항 계산
    float v_bat = g_bms.pack_voltage_mv / 1000.0f;
    float v_p = v_pos * 3.3f / 4096;
    float v_n = v_neg * 3.3f / 4096;
    
    if (v_p > 0.1f) {
        g_iso.r_iso_pos = R_MEASURE_OHM * v_p / (v_bat - v_p);
    }
    if (v_n > 0.1f) {
        g_iso.r_iso_neg = R_MEASURE_OHM * v_n / (v_bat - v_n);
    }
}
```

---

## 폴트 판정

```c
#define ISO_FAULT_THRESHOLD  100000  // 100kΩ
#define ISO_WARNING_THRESHOLD 500000  // 500kΩ

void ISO_Check(void) {
    uint32_t r_min = min(g_iso.r_iso_pos, g_iso.r_iso_neg);
    
    if (r_min < ISO_FAULT_THRESHOLD) {
        g_iso.fault = true;
        BMS_SetFault(FAULT_ISOLATION);
    }
    else if (r_min < ISO_WARNING_THRESHOLD) {
        BMS_SetWarning(WARN_ISOLATION_LOW);
    }
}
```

---

## 전용 IC

정확한 측정이 필요하면 전용 IMD(Insulation Monitoring Device) IC 사용.

Bender ISOMETER, Littelfuse 등.

나는 간단한 용도라 자체 구현했는데, 자동차용은 전용 IC 쓰는 게 좋다.

---

## 주의사항

절연 측정 시 배터리에서 새시로 전류가 흐른다. 아주 작지만.

- 측정 중 아닐 때는 스위치 OFF
- 측정 주기 길게 (1분에 1회 정도)
- 고전압 작업 시 안전 주의

---

## 정리

- 절연 저항: 배터리↔새시 사이
- 100kΩ 이하면 위험
- 알려진 저항으로 분압 측정
- 정밀 측정은 전용 IC

---

Part 7 고급 기능편 끝.

다음은 실전 적용편.

[#25 - 실제 배터리 연결](/posts/bms/ad7280a-bms-dev-25/)
