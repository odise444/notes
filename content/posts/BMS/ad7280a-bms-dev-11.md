---
title: "AD7280A BMS 개발기 #11 - 밸런싱 알고리즘"
date: 2024-01-25
draft: false
tags: ["AD7280A", "BMS", "STM32", "밸런싱", "알고리즘"]
categories: ["BMS 개발"]
series: ["AD7280A BMS 개발"]
summary: "어떤 셀을 밸런싱할지 결정하는 로직. 단순한 것 같은데 고려할 게 많다."
---

밸런싱 하드웨어는 됐다. 이제 언제, 어떤 셀을 밸런싱할지 결정하는 알고리즘.

---

## 기본 아이디어

최소 전압 셀 기준으로 나머지를 맞춘다.

```
전압 분포:
Cell 0: 3.35V  ← 최소와 20mV 차이 → 밸런싱
Cell 1: 3.32V  ← 최소와 -10mV → 안 함
Cell 2: 3.33V  ← 최소와 0mV (최소) → 안 함
Cell 3: 3.37V  ← 최소와 40mV 차이 → 밸런싱
Cell 4: 3.34V  ← 최소와 10mV → 안 함
Cell 5: 3.36V  ← 최소와 30mV 차이 → 밸런싱
```

최소 전압보다 일정 값(예: 20mV) 이상 높은 셀만 밸런싱.

---

## 델타 전압 방식

```c
#define BALANCE_THRESHOLD_MV  20  // 밸런싱 시작 기준

void BMS_UpdateBalancing(void) {
    uint16_t min_mv = UINT16_MAX;
    
    // 최소 전압 찾기
    for (int i = 0; i < 24; i++) {
        if (g_cells_mv[i] < min_mv) {
            min_mv = g_cells_mv[i];
        }
    }
    
    // 밸런싱 대상 결정
    for (int i = 0; i < 24; i++) {
        int delta = g_cells_mv[i] - min_mv;
        
        if (delta > BALANCE_THRESHOLD_MV) {
            g_balance_mask[i / 6] |= (1 << (i % 6));
        }
    }
    
    // 밸런싱 적용
    for (int dev = 0; dev < 4; dev++) {
        AD7280A_SetBalance(dev, g_balance_mask[dev]);
    }
}
```

---

## 히스테리시스

단순 threshold만 쓰면 20mV 근처에서 밸런싱이 껐다 켜졌다 반복한다.

```
측정 1: 3.321V → 밸런싱 OFF (21mV, 기준 이하)
측정 2: 3.319V → 밸런싱 ON (19mV, 기준 초과)
측정 3: 3.321V → 밸런싱 OFF
... 반복
```

히스테리시스 추가:

```c
#define BALANCE_START_MV  20  // 시작 기준
#define BALANCE_STOP_MV   10  // 종료 기준

void BMS_UpdateBalancing(void) {
    for (int i = 0; i < 24; i++) {
        int delta = g_cells_mv[i] - min_mv;
        
        if (g_balancing[i]) {
            // 이미 밸런싱 중 → 종료 기준으로 판단
            if (delta < BALANCE_STOP_MV) {
                g_balancing[i] = false;
            }
        } else {
            // 밸런싱 안 함 → 시작 기준으로 판단
            if (delta > BALANCE_START_MV) {
                g_balancing[i] = true;
            }
        }
    }
}
```

---

## 충전 중만 밸런싱

밸런싱은 충전 중에만 하는 게 일반적이다.

방전 중에 밸런싱하면:
- 이미 방전 중인데 추가로 방전 → 셀 손상 위험
- 의미 없음 (방전하면 어차피 전압 떨어짐)

```c
void BMS_UpdateBalancing(void) {
    if (g_bms.state != STATE_CHARGING) {
        // 충전 중 아니면 밸런싱 OFF
        for (int dev = 0; dev < 4; dev++) {
            AD7280A_SetBalance(dev, 0x00);
        }
        return;
    }
    
    // 충전 중이면 밸런싱 로직 실행
    // ...
}
```

근데 나중에 LFP로 바꾸면서 방전 중에도 밸런싱하는 옵션을 추가했다. 상황에 따라 다르다.

---

## 전압 범위 체크

너무 낮은 전압에서는 밸런싱 안 함:

```c
#define BALANCE_MIN_VOLTAGE_MV  3000  // 3.0V 이하면 안 함

void BMS_UpdateBalancing(void) {
    uint16_t min_mv = GetMinCellVoltage();
    
    if (min_mv < BALANCE_MIN_VOLTAGE_MV) {
        // 전압 낮으면 밸런싱 하지 않음
        return;
    }
    
    // ...
}
```

저전압 셀을 더 방전시키면 셀 손상.

---

## 주기적 실행

1초마다 밸런싱 로직 실행:

```c
void BMS_Task(void) {
    static uint32_t last_balance_time = 0;
    
    if (HAL_GetTick() - last_balance_time > 1000) {
        last_balance_time = HAL_GetTick();
        
        // 밸런싱 OFF → 측정 → 밸런싱 결정 → 밸런싱 ON
        BMS_StopAllBalance();
        HAL_Delay(100);
        BMS_ReadAllCells();
        BMS_UpdateBalancing();
    }
}
```

---

## 테스트 결과

편차가 큰 배터리로 테스트:

```
초기 상태:
Cell 0: 3.450V
Cell 1: 3.320V
Cell 2: 3.380V
...
max-min: 130mV
```

1시간 충전 + 밸런싱 후:

```
Cell 0: 3.510V
Cell 1: 3.505V
Cell 2: 3.508V
...
max-min: 12mV
```

잘 맞춰진다.

---

## 정리

- 델타 전압 방식: 최소 전압 대비 높은 셀 밸런싱
- 히스테리시스: 시작/종료 기준 다르게
- 충전 중에만 밸런싱
- 저전압에서는 안 함

---

다음은 밸런싱 중 발열 관리.

[#12 - 발열 관리](/posts/bms/ad7280a-bms-dev-12/)
