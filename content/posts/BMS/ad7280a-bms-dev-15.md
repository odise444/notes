---
title: "AD7280A BMS 개발기 #15 - 알람 레지스터 해석"
date: 2024-12-13
draft: false
tags: ["AD7280A", "BMS", "STM32", "알람"]
categories: ["BMS 개발"]
series: ["AD7280A BMS 개발"]
summary: "Alert 레지스터 A, B, C. 어떤 셀이 문제인지 비트맵으로 알려준다."
---

Alert가 발생하면 어떤 셀이 문제인지 알아야 한다. Alert 레지스터가 3개 있다.

```c
// Alert Register A (0x18) - 과전압
// Bit 5: Cell 6 OV
// Bit 4: Cell 5 OV
// ...
// Bit 0: Cell 1 OV

// Alert Register B (0x19) - 저전압
// Bit 5: Cell 6 UV
// ...

// Alert Register C (0x1A) - 기타
// Bit 5: Self-test fail
// Bit 4: Thermal shutdown
// ...
```

---

알람 처리 함수:

```c
void HandleAlert(uint8_t dev, uint8_t alert_a, uint8_t alert_b, uint8_t alert_c) {
    // 과전압 체크
    for (int i = 0; i < 6; i++) {
        if (alert_a & (1 << i)) {
            int cell_num = dev * 6 + i;
            LogEvent(EVT_OVERVOLTAGE, cell_num);
        }
    }
    
    // 저전압 체크
    for (int i = 0; i < 6; i++) {
        if (alert_b & (1 << i)) {
            int cell_num = dev * 6 + i;
            LogEvent(EVT_UNDERVOLTAGE, cell_num);
        }
    }
    
    // Self-test 실패
    if (alert_c & 0x20) {
        LogEvent(EVT_SELFTEST_FAIL, dev);
    }
}
```

---

알람은 읽어도 자동으로 클리어 안 된다. 명시적으로 클리어해줘야 한다.

```c
void ClearAlert(uint8_t dev) {
    AD7280A_WriteReg(dev, REG_ALERT, 0xFF);  // 모든 비트 클리어
}
```

---

다음 글에서 BMS 전체 보호 로직을 통합한다.

[#16 - 보호 로직 통합](/posts/bms/ad7280a-bms-dev-16/)
