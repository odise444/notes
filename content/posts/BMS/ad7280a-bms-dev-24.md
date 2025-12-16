---
title: "AD7280A BMS 개발기 #24 - 절연 모니터링"
date: 2024-12-13
draft: false
tags: ["AD7280A", "BMS", "STM32", "절연", "IMD"]
categories: ["BMS 개발"]
series: ["AD7280A BMS 개발"]
summary: "고전압 배터리는 절연이 중요하다. 누설 전류를 감지해야 한다."
---

72V 배터리가 차체(GND)에 닿으면 감전 위험이 있다. 절연 상태를 모니터링해야 한다.

---

절연 저항 기준:

```
전기차 기준: 100Ω/V 이상
72V 시스템: 7.2kΩ 이상이면 OK
1kΩ 이하면 위험
```

---

간단한 측정 방법: 배터리 +/-와 차체 GND 사이 전압을 측정한다.

```
Battery+ ──┬── 1MΩ ──┬── ADC_ISO+
           │         │
           │       100kΩ
           │         │
          GND ───────┴── ADC_ISO-
           │
Battery- ──┴── 1MΩ ── ADC로 측정
```

정상이면 배터리 전압의 절반이 측정된다. 한쪽이 GND에 닿으면 전압이 치우친다.

```c
void Isolation_Check(void) {
    uint16_t adc_plus = ADC_Read(ADC_ISO_PLUS);
    uint16_t adc_minus = ADC_Read(ADC_ISO_MINUS);
    
    // 정상: 둘 다 배터리 전압의 ~50%
    // 비정상: 한쪽이 0V에 가깝거나 배터리 전압에 가까움
    
    float ratio = (float)adc_plus / (adc_plus + adc_minus);
    
    if (ratio < 0.3f || ratio > 0.7f) {
        SetFault(FAULT_ISOLATION);
    }
}
```

---

더 정밀하게 하려면 전용 IMD(Insulation Monitoring Device) IC를 쓴다. Bender사 제품이 유명한데 비싸다.

우리는 간단한 방법으로 구현했다. 정밀도는 좀 떨어지지만 심각한 절연 파괴는 감지한다.

---

이걸로 고급 기능 끝. 다음 글부터 실전 적용편이다.

[#25 - 72V 팩 첫 연결](/posts/bms/ad7280a-bms-dev-25/)
