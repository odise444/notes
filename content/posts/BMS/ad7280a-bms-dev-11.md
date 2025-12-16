---
title: "AD7280A BMS 개발기 #11 - 밸런싱 알고리즘"
date: 2024-12-11
draft: false
tags: ["AD7280A", "BMS", "STM32", "밸런싱", "알고리즘"]
categories: ["BMS 개발"]
series: ["AD7280A BMS 개발"]
summary: "어떤 셀을 언제 밸런싱할지 결정하는 로직. 생각보다 복잡하다."
---

밸런싱 하드웨어는 됐다. 이제 "언제 어떤 셀을 밸런싱할지" 결정하는 알고리즘이 필요하다.

---

가장 단순한 방법: 델타 밸런싱

최저 전압 셀을 기준으로, 차이가 임계값보다 큰 셀을 방전시킨다.

```c
#define BALANCE_THRESHOLD_MV    20   // 20mV 이상 차이나면 밸런싱
#define BALANCE_HYSTERESIS_MV   10   // 히스테리시스

void Balancing_Update(uint16_t *cell_mv, uint8_t *balance_mask) {
    // 최저 전압 찾기
    uint16_t min_v = cell_mv[0];
    for (int i = 1; i < 24; i++) {
        if (cell_mv[i] < min_v) min_v = cell_mv[i];
    }
    
    // 각 셀 판단
    for (int i = 0; i < 24; i++) {
        int16_t delta = cell_mv[i] - min_v;
        
        if (delta > BALANCE_THRESHOLD_MV) {
            balance_mask[i / 6] |= (1 << (i % 6));  // ON
        } 
        else if (delta < BALANCE_HYSTERESIS_MV) {
            balance_mask[i / 6] &= ~(1 << (i % 6)); // OFF
        }
        // 그 사이는 현재 상태 유지 (히스테리시스)
    }
}
```

---

히스테리시스가 왜 필요하냐면, 없으면 밸런싱이 ON/OFF를 반복한다.

```
히스테리시스 없을 때:
Cell 1: 3225mV → 밸런싱 ON
Cell 1: 3219mV → 밸런싱 OFF (기준보다 낮아짐)
Cell 1: 3221mV → 밸런싱 ON (다시 높아짐)
... 계속 반복
```

릴레이나 MOSFET 수명에 안 좋다.

---

밸런싱 동작 조건도 중요하다.

```c
bool Should_Balance(void) {
    // 충전 중일 때만 밸런싱
    if (bms_state != BMS_STATE_CHARGING) return false;
    
    // 전압이 일정 수준 이상일 때만
    if (min_cell_voltage < 3200) return false;
    
    // 온도가 적정 범위일 때만
    if (max_temp > 45 || min_temp < 0) return false;
    
    return true;
}
```

방전 중에 밸런싱하면 에너지 낭비다. 충전 막판에 하는 게 효율적.

---

실제 적용해보니 밸런싱 시간이 오래 걸렸다.

100mAh 셀 편차를 100mA로 밸런싱하면 1시간 걸린다. 500mAh 차이면 5시간.

그래서 충전할 때마다 조금씩 밸런싱하고, 완전히 맞추는 건 포기했다. 30mV 이내면 충분하다고 판단.

---

다음 글에서 밸런싱 중 발열 관리를 다룬다.

[#12 - 밸런싱 발열 관리](/posts/bms/ad7280a-bms-dev-12/)
