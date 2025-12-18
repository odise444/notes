---
title: "AD7280A BMS 개발기 #12 - 밸런싱 발열 관리"
date: 2024-01-26
draft: false
tags: ["AD7280A", "BMS", "STM32", "밸런싱", "발열"]
categories: ["BMS 개발"]
series: ["AD7280A BMS 개발"]
summary: "24셀 전부 밸런싱하면 8W 발열. 보드가 뜨거워진다."
---

밸런싱 알고리즘은 됐다. 근데 테스트하다 보니 보드가 뜨거워졌다.

---

## 발열 계산

33Ω 저항, 3.3V 셀 기준:

```
P = V² / R = 3.3² / 33 = 0.33W (셀당)
```

24셀 전부 밸런싱하면:

```
0.33W × 24 = 8W
```

8W가 좁은 PCB에서 발생. 저항 온도가 80°C 넘게 올라갔다.

---

## 체커보드 패턴

동시에 전부 켜지 말고 번갈아가며:

```
Phase A: Cell 1, 3, 5, 7, 9, 11, ...  (홀수)
Phase B: Cell 2, 4, 6, 8, 10, 12, ... (짝수)
```

A → B → A → B 반복. 한 번에 최대 12셀만 밸런싱.

```c
static uint8_t balance_phase = 0;

void BMS_ApplyBalanceCheckerboard(void) {
    uint8_t mask[4] = {0, 0, 0, 0};
    
    for (int i = 0; i < 24; i++) {
        if (!g_balancing[i]) continue;
        
        // 홀수 셀은 Phase 0, 짝수 셀은 Phase 1
        if ((i % 2) == balance_phase) {
            mask[i / 6] |= (1 << (i % 6));
        }
    }
    
    for (int dev = 0; dev < 4; dev++) {
        AD7280A_SetBalance(dev, mask[dev]);
    }
    
    // 다음 phase
    balance_phase = 1 - balance_phase;
}
```

발열이 절반으로 줄었다.

---

## 듀티 제어

더 줄이고 싶으면 듀티 사이클 제어:

```
ON 5초 → OFF 5초 → ON 5초 → ...
```

50% 듀티면 평균 발열도 50%.

```c
#define BALANCE_ON_TIME_MS   5000
#define BALANCE_OFF_TIME_MS  5000

void BMS_BalanceTask(void) {
    static uint32_t last_toggle = 0;
    static bool balance_enabled = true;
    
    uint32_t now = HAL_GetTick();
    
    if (balance_enabled) {
        if (now - last_toggle > BALANCE_ON_TIME_MS) {
            last_toggle = now;
            balance_enabled = false;
            BMS_StopAllBalance();
        }
    } else {
        if (now - last_toggle > BALANCE_OFF_TIME_MS) {
            last_toggle = now;
            balance_enabled = true;
            BMS_ApplyBalanceCheckerboard();
        }
    }
}
```

---

## 온도 기반 제어

보드 온도 모니터링해서 동적으로 조절:

```c
void BMS_BalanceTask(void) {
    int16_t board_temp = BMS_GetBoardTemperature();
    
    if (board_temp > 600) {  // 60°C 초과
        // 밸런싱 중지
        BMS_StopAllBalance();
        return;
    }
    
    if (board_temp > 500) {  // 50°C 초과
        // 듀티 낮춤
        g_balance_duty = 30;  // 30%
    } else {
        g_balance_duty = 50;  // 50%
    }
    
    // ...
}
```

---

## 저항 배치

PCB 설계 단계에서 고려해야 할 것:

1. **분산 배치**: 저항끼리 붙이지 말고 띄워서
2. **써멀 비아**: 저항 아래에 비아 추가, 뒷면 GND로 열 분산
3. **큰 저항**: 0603 대신 1206이나 2512

처음에 0603으로 빽빽하게 배치했다가 나중에 2512로 바꿨다. PCB 다시 뽑았다.

---

## 밸런싱 시간 계산

100mA로 밸런싱할 때 1mV 줄이는 데 걸리는 시간:

```
셀 용량: 10Ah
방전 전류: 0.1A
1mV당 용량: 10Ah / 4200mV ≈ 2.4mAh/mV

1mV 줄이는 시간: 2.4mAh / 100mA = 0.024h ≈ 1.4분
```

20mV 편차 줄이려면 약 30분. 생각보다 오래 걸린다.

밸런싱 전류를 높이면 빨라지지만 발열이 심해진다. 트레이드오프.

---

## 최종 설정

내가 쓰는 설정:

```c
#define BALANCE_RESISTOR_OHM     33    // 100mA @ 3.3V
#define BALANCE_THRESHOLD_MV     20
#define BALANCE_STOP_MV          10
#define BALANCE_ON_TIME_MS       5000
#define BALANCE_OFF_TIME_MS      5000
#define BALANCE_MAX_TEMP         600   // 60°C
```

체커보드 + 50% 듀티 + 온도 모니터링.

---

## 정리

- 체커보드 패턴: 홀수/짝수 번갈아
- 듀티 제어: ON/OFF 반복
- 온도 기반: 과열 시 중지
- PCB 설계: 분산 배치, 써멀 비아

밸런싱은 느린 작업이다. 급하게 하려고 전류 높이면 발열 문제 생긴다.

---

Part 4 밸런싱편 끝.

다음은 알람/보호 기능.

[#13 - 과전압/저전압 알람](/posts/bms/ad7280a-bms-dev-13/)
