---
title: "AD7280A BMS 개발기 #21 - SOH 추정, 배터리 건강 상태"
date: 2024-02-04
draft: false
tags: ["AD7280A", "BMS", "STM32", "SOH", "수명"]
categories: ["BMS 개발"]
series: ["AD7280A BMS 개발"]
summary: "배터리가 얼마나 늙었는지 알아야 한다. SOH 추정하기."
---

SOC는 "지금 몇 % 남았나"고, SOH는 "원래 대비 얼마나 건강한가"다.

새 배터리 100%, 쓰다 보면 80%, 70%로 떨어진다.

---

## SOH란

State of Health. 건강 상태.

```
SOH = 현재 최대 용량 / 정격 용량 × 100
```

새 배터리 10Ah → 3년 후 8Ah면 SOH 80%.

---

## 측정 방법

### 1. 용량 기반

완충→완방 용량 측정:

```c
void SOH_UpdateCapacity(void) {
    static float discharge_ah = 0;
    static bool measuring = false;
    
    // 완충 상태에서 시작
    if (g_bms.soc >= 99 && !measuring) {
        measuring = true;
        discharge_ah = 0;
    }
    
    // 방전량 적산
    if (measuring && g_bms.pack_current_ma < 0) {
        float dt = 0.1f;  // 100ms
        discharge_ah += fabsf(g_bms.pack_current_ma / 1000.0f) * dt / 3600.0f;
    }
    
    // 완방 도달
    if (measuring && g_bms.soc <= 1) {
        measuring = false;
        
        g_bms.measured_capacity_ah = discharge_ah;
        g_bms.soh_capacity = (discharge_ah / g_bms.rated_capacity_ah) * 100;
    }
}
```

문제: 완충→완방 사이클이 필요. 일상 사용에서 잘 안 일어남.

### 2. 내부 저항 기반

배터리 노화하면 내부 저항 증가.

```c
void SOH_UpdateResistance(void) {
    // 부하 변화 시 저항 측정
    // R = ΔV / ΔI
    
    static int16_t prev_current = 0;
    static uint16_t prev_voltage = 0;
    
    int16_t di = g_bms.pack_current_ma - prev_current;
    int16_t dv = g_bms.pack_voltage_mv - prev_voltage;
    
    prev_current = g_bms.pack_current_ma;
    prev_voltage = g_bms.pack_voltage_mv;
    
    // 전류 변화가 충분히 클 때만
    if (abs(di) > 5000) {  // 5A 이상 변화
        float r_mohm = (float)abs(dv) / (abs(di) / 1000.0f);  // mΩ
        
        // 이동 평균
        g_bms.internal_resistance = g_bms.internal_resistance * 0.9f + r_mohm * 0.1f;
        
        // 저항 기반 SOH (신품 대비)
        g_bms.soh_resistance = (g_bms.new_resistance / g_bms.internal_resistance) * 100;
        if (g_bms.soh_resistance > 100) g_bms.soh_resistance = 100;
    }
}
```

---

## 복합 SOH

용량과 저항 둘 다 고려:

```c
void SOH_Update(void) {
    // 둘 다 있으면 가중 평균
    if (g_bms.soh_capacity > 0 && g_bms.soh_resistance > 0) {
        g_bms.soh = g_bms.soh_capacity * 0.7f + g_bms.soh_resistance * 0.3f;
    }
    // 하나만 있으면 그거 사용
    else if (g_bms.soh_capacity > 0) {
        g_bms.soh = g_bms.soh_capacity;
    }
    else if (g_bms.soh_resistance > 0) {
        g_bms.soh = g_bms.soh_resistance;
    }
}
```

---

## 사이클 카운트

충방전 사이클 수도 노화 지표:

```c
void SOH_CountCycle(void) {
    static uint8_t prev_soc = 0;
    static float partial_cycle = 0;
    
    int delta = prev_soc - g_bms.soc;
    prev_soc = g_bms.soc;
    
    if (delta > 0) {  // 방전
        partial_cycle += delta / 100.0f;
    }
    
    // 1 사이클 누적되면 카운트
    if (partial_cycle >= 1.0f) {
        partial_cycle -= 1.0f;
        g_bms.cycle_count++;
        
        // Flash에 저장
        Flash_WriteU32(FLASH_CYCLE_ADDR, g_bms.cycle_count);
    }
}
```

---

## SOH 표시

```
사이클:        500회
측정 용량:     9.2Ah / 10Ah (92%)
내부 저항:     25mΩ / 20mΩ (80%)
종합 SOH:      88%
```

---

## 정리

- 용량 SOH: 완충→완방 용량 측정
- 저항 SOH: 부하 변화 시 저항 측정
- 사이클 수: 누적 방전량
- 복합 SOH: 가중 평균

SOH 80% 이하면 배터리 교체 권장.

---

다음은 칼만 필터로 SOC 정확도 높이기.

[#22 - 칼만 필터](/posts/bms/ad7280a-bms-dev-22/)
