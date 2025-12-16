---
title: "AD7280A BMS 개발기 #21 - SOH 추정"
date: 2024-12-13
draft: false
tags: ["AD7280A", "BMS", "STM32", "SOH", "수명"]
categories: ["BMS 개발"]
series: ["AD7280A BMS 개발"]
summary: "배터리 건강 상태. 용량이 얼마나 줄었는지 추정한다."
---

SOH(State of Health)는 배터리 수명 상태다. 새 배터리가 100%고, 쓰다 보면 떨어진다. 80% 이하면 교체 시기.

---

SOH 추정 방법은 크게 두 가지.

1. 용량 기반: 실제 방전 가능 용량 / 정격 용량
2. 내부 저항 기반: 저항 증가율

용량 기반이 직관적인데, 완충→완방 사이클이 필요해서 실시간 측정이 어렵다.

```c
// 완충 후 완방했을 때 실측 용량
void SOH_UpdateFromCycle(float discharged_ah) {
    float nominal_ah = 100.0f;  // 정격 용량
    soh_capacity = (discharged_ah / nominal_ah) * 100.0f;
}
```

---

내부 저항은 전류 변화 시 전압 변화로 추정한다.

```c
// DC 펄스 방식
// 전류 스텝 인가 후 전압 변화 측정
// R = dV / dI

void SOH_MeasureResistance(void) {
    // 부하 OFF 상태 전압
    float v_rest = GetPackVoltage();
    
    // 부하 ON (10A)
    EnableLoad(10.0f);
    HAL_Delay(100);
    float v_load = GetPackVoltage();
    
    // 저항 계산
    float r_pack = (v_rest - v_load) / 10.0f;
    float r_cell = r_pack / 24;  // 셀당
    
    // 새 배터리 저항 대비 증가율
    soh_resistance = (r_cell_new / r_cell) * 100.0f;
}
```

---

두 방법을 합쳐서 종합 SOH를 계산한다.

```c
soh = (soh_capacity * 0.7f) + (soh_resistance * 0.3f);
```

---

다음 글에서 칼만 필터로 SOC 정밀도를 높인다.

[#22 - 칼만 필터](/posts/bms/ad7280a-bms-dev-22/)
