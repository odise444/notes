---
title: "AD7280A BMS 개발 삽질기 #11 - 밸런싱 알고리즘 구현"
date: 2024-12-09
draft: false
tags: ["BMS", "AD7280A", "임베디드", "배터리", "삽질", "밸런싱", "알고리즘"]
categories: ["BMS 개발"]
summary: "단순히 높은 셀만 방전시키면 될 줄 알았다. 현실은 더 복잡했다."
---

## 지난 글 요약

[지난 글](/posts/bms/ad7280a-bms-dev-10/)에서 패시브 밸런싱 기초를 다뤘다. 스위치 켜고 끄는 건 됐는데... 언제, 어떤 셀을 밸런싱할지 결정하는 게 문제다.

## 밸런싱 알고리즘 종류

| 방식 | 기준 | 장점 | 단점 |
|------|------|------|------|
| 델타 전압 | 셀 전압 차이 | 단순, 구현 쉬움 | 충전 상태 무시 |
| 절대 전압 | 고정 임계값 | 직관적 | 유연성 부족 |
| SOC 기반 | 충전 상태 추정 | 정확함 | 복잡, SOC 오차 누적 |
| 하이브리드 | 전압 + SOC | 균형잡힘 | 구현 복잡 |

## 방식 1: 델타 전압 기반 (가장 흔함)

**원리**: 가장 낮은 셀 대비 일정 전압 이상 높은 셀을 방전.

```c
#define BALANCE_START_DELTA_MV   30   // 30mV 이상 차이나면 시작
#define BALANCE_STOP_DELTA_MV    10   // 10mV 이내면 중지

typedef struct {
    float voltages[6];
    uint8_t balance_mask;
    float min_voltage;
    float max_voltage;
} cell_data_t;

void delta_voltage_balance(cell_data_t *data) {
    // 최소/최대 전압 찾기
    data->min_voltage = data->voltages[0];
    data->max_voltage = data->voltages[0];
    
    for (int i = 1; i < 6; i++) {
        if (data->voltages[i] < data->min_voltage)
            data->min_voltage = data->voltages[i];
        if (data->voltages[i] > data->max_voltage)
            data->max_voltage = data->voltages[i];
    }
    
    // 밸런싱 필요 여부
    float spread = data->max_voltage - data->min_voltage;
    if (spread < BALANCE_START_DELTA_MV) {
        data->balance_mask = 0;  // 밸런싱 불필요
        return;
    }
    
    // 높은 셀 선택
    data->balance_mask = 0;
    for (int i = 0; i < 6; i++) {
        float delta = data->voltages[i] - data->min_voltage;
        if (delta > BALANCE_STOP_DELTA_MV) {
            data->balance_mask |= (1 << i);
        }
    }
}
```

### 문제점: 충전 중 전압 변동

충전 중에는 전압이 계속 올라간다. 측정 시점에 따라 밸런싱 대상이 바뀜.

```
시점 1: Cell 1=4.10V, Cell 2=4.08V → Cell 1 밸런싱
시점 2: Cell 1=4.12V, Cell 2=4.11V → Cell 1 밸런싱
시점 3: Cell 1=4.13V, Cell 2=4.14V → Cell 2 밸런싱 ← 뒤바뀜!
```

**해결**: 이동 평균 또는 휴지 전압(OCV) 기반 판단.

## 방식 2: 절대 전압 기반

**원리**: 특정 전압 이상인 셀만 방전.

```c
#define BALANCE_VOLTAGE_THRESHOLD_MV  4100  // 4.1V 이상이면 밸런싱

void absolute_voltage_balance(cell_data_t *data) {
    data->balance_mask = 0;
    
    for (int i = 0; i < 6; i++) {
        if (data->voltages[i] > BALANCE_VOLTAGE_THRESHOLD_MV) {
            data->balance_mask |= (1 << i);
        }
    }
}
```

### 문제점

- 모든 셀이 4.1V 이상이면 전부 방전 → 의미 없음
- 셀 간 편차는 줄지 않음

**결론**: 단독 사용 비추천. 델타 방식과 조합.

## 방식 3: SOC 기반

**원리**: 각 셀의 충전 상태(SOC)를 추정하고, SOC가 높은 셀을 방전.

```c
typedef struct {
    float voltage;
    float soc;          // 0.0 ~ 100.0 %
    float capacity_mAh;
    int32_t coulomb_uAh; // 쿨롱 카운팅 누적
} cell_state_t;

// OCV-SOC 테이블 (리튬이온 예시)
const float OCV_TABLE[] = {3.0, 3.3, 3.5, 3.6, 3.7, 3.8, 3.9, 4.0, 4.1, 4.2};
const float SOC_TABLE[] = {0,   10,  20,  30,  40,  50,  60,  70,  85,  100};

float voltage_to_soc(float voltage) {
    // 선형 보간
    for (int i = 0; i < 9; i++) {
        if (voltage >= OCV_TABLE[i] && voltage < OCV_TABLE[i+1]) {
            float ratio = (voltage - OCV_TABLE[i]) / 
                         (OCV_TABLE[i+1] - OCV_TABLE[i]);
            return SOC_TABLE[i] + ratio * (SOC_TABLE[i+1] - SOC_TABLE[i]);
        }
    }
    return (voltage < 3.0f) ? 0.0f : 100.0f;
}

void soc_based_balance(cell_state_t *cells, uint8_t *balance_mask) {
    // 최소 SOC 찾기
    float min_soc = cells[0].soc;
    for (int i = 1; i < 6; i++) {
        if (cells[i].soc < min_soc)
            min_soc = cells[i].soc;
    }
    
    // SOC 차이 기반 밸런싱
    *balance_mask = 0;
    for (int i = 0; i < 6; i++) {
        if (cells[i].soc - min_soc > 5.0f) {  // 5% 이상 차이
            *balance_mask |= (1 << i);
        }
    }
}
```

### 문제점: SOC 추정 오차

- 쿨롱 카운팅: 전류 센서 오차 누적
- OCV 기반: 부하 중에는 부정확

**현실**: 정확한 SOC는 칼만 필터 등 복잡한 알고리즘 필요.

## 방식 4: 하이브리드 (실전 추천)

델타 전압 + 조건부 적용.

```c
typedef struct {
    bool charging;           // 충전 중?
    bool balancing_enabled;  // 밸런싱 활성화?
    float cell_voltages[6];
    float pack_current;      // 양수=충전, 음수=방전
    float temperature;
} bms_state_t;

#define BALANCE_MIN_VOLTAGE_MV    3500   // 3.5V 이하면 밸런싱 안 함
#define BALANCE_MAX_TEMP_C        45     // 45°C 이상이면 중지
#define BALANCE_MIN_CURRENT_MA    100    // 100mA 이상 충전 중일 때만
#define BALANCE_DELTA_MV          20     // 20mV 편차

uint8_t hybrid_balance(bms_state_t *state) {
    uint8_t mask = 0;
    
    // 조건 1: 충전 중이어야 함
    if (state->pack_current < BALANCE_MIN_CURRENT_MA) {
        return 0;
    }
    
    // 조건 2: 온도 범위
    if (state->temperature > BALANCE_MAX_TEMP_C) {
        return 0;
    }
    
    // 최소 전압 찾기
    float min_v = state->cell_voltages[0];
    for (int i = 1; i < 6; i++) {
        if (state->cell_voltages[i] < min_v)
            min_v = state->cell_voltages[i];
    }
    
    // 조건 3: 최소 전압 이상
    if (min_v < BALANCE_MIN_VOLTAGE_MV) {
        return 0;
    }
    
    // 델타 기반 선택
    for (int i = 0; i < 6; i++) {
        if (state->cell_voltages[i] - min_v > BALANCE_DELTA_MV) {
            mask |= (1 << i);
        }
    }
    
    return mask;
}
```

## 삽질 1: 밸런싱 떨림 (Oscillation)

경계값 근처에서 밸런싱이 켜졌다 꺼졌다 반복.

```
Cycle 1: delta=21mV → ON
Cycle 2: delta=19mV → OFF
Cycle 3: delta=21mV → ON
...
```

### 해결: 히스테리시스

```c
#define BALANCE_START_DELTA_MV   30   // 시작: 30mV
#define BALANCE_STOP_DELTA_MV    10   // 중지: 10mV

typedef struct {
    uint8_t balance_mask;
    bool is_balancing[6];  // 현재 밸런싱 중인지 상태 유지
} balance_state_t;

void balance_with_hysteresis(cell_data_t *data, balance_state_t *state) {
    float min_v = find_min_voltage(data->voltages, 6);
    
    state->balance_mask = 0;
    for (int i = 0; i < 6; i++) {
        float delta = data->voltages[i] - min_v;
        
        if (state->is_balancing[i]) {
            // 이미 밸런싱 중 → 낮은 임계값으로 판단
            if (delta > BALANCE_STOP_DELTA_MV) {
                state->balance_mask |= (1 << i);
            } else {
                state->is_balancing[i] = false;  // 중지
            }
        } else {
            // 밸런싱 안 함 → 높은 임계값으로 판단
            if (delta > BALANCE_START_DELTA_MV) {
                state->balance_mask |= (1 << i);
                state->is_balancing[i] = true;   // 시작
            }
        }
    }
}
```

## 삽질 2: 측정과 밸런싱 타이밍

밸런싱 중 측정하면 값이 틀어진다. ([#10](/posts/bms/ad7280a-bms-dev-10/) 참고)

### 해결: 시분할 스케줄링

```c
typedef enum {
    STATE_MEASURE,
    STATE_BALANCE,
    STATE_IDLE
} schedule_state_t;

void balance_scheduler(void) {
    static schedule_state_t state = STATE_MEASURE;
    static uint32_t last_tick = 0;
    uint32_t now = HAL_GetTick();
    
    switch (state) {
    case STATE_MEASURE:
        // 밸런싱 OFF, 안정화 대기
        ad7280a_set_balance(0, 0x00);
        if (now - last_tick > 50) {  // 50ms 대기
            ad7280a_read_all_cells(voltages);
            state = STATE_BALANCE;
            last_tick = now;
        }
        break;
        
    case STATE_BALANCE:
        // 밸런싱 마스크 계산 및 적용
        uint8_t mask = hybrid_balance(&bms_state);
        ad7280a_set_balance(0, mask);
        if (now - last_tick > 1000) {  // 1초 밸런싱
            state = STATE_MEASURE;
            last_tick = now;
        }
        break;
    }
}
```

## 삽질 3: 다중 디바이스 밸런싱

8개 데이지체인이면 48셀. 전체를 고려해야 한다.

```c
void multi_device_balance(float voltages[8][6]) {
    // 48셀 중 최소 전압 찾기
    float global_min = voltages[0][0];
    for (int dev = 0; dev < 8; dev++) {
        for (int cell = 0; cell < 6; cell++) {
            if (voltages[dev][cell] < global_min)
                global_min = voltages[dev][cell];
        }
    }
    
    // 각 디바이스별 밸런싱 마스크
    for (int dev = 0; dev < 8; dev++) {
        uint8_t mask = 0;
        for (int cell = 0; cell < 6; cell++) {
            if (voltages[dev][cell] - global_min > BALANCE_DELTA_MV) {
                mask |= (1 << cell);
            }
        }
        ad7280a_set_balance(dev, mask);
    }
}
```

## 밸런싱 효과 측정

실제 테스트 결과 (6셀, 68Ω 저항, 충전 중):

```
=== Before Balancing ===
Cell 1: 4180mV
Cell 2: 4150mV
Cell 3: 4200mV  ← max
Cell 4: 4120mV  ← min
Cell 5: 4170mV
Cell 6: 4160mV
Spread: 80mV

=== After 2 Hours ===
Cell 1: 4155mV
Cell 2: 4150mV
Cell 3: 4158mV
Cell 4: 4148mV
Cell 5: 4152mV
Cell 6: 4153mV
Spread: 10mV ✓
```

## 삽질 포인트 정리

1. **델타 전압이 가장 실용적** - SOC는 오차 누적 문제
2. **히스테리시스 필수** - 떨림 방지
3. **측정/밸런싱 분리** - 시분할 스케줄링
4. **조건 체크** - 충전 중, 온도, 최소 전압
5. **다중 디바이스** - 전체 셀 기준으로 판단

다음 글에서는 밸런싱 중 발열 관리를 다룬다.

---

## 참고 자료

- [AD7280A Datasheet - Cell Balancing](https://www.analog.com/media/en/technical-documentation/data-sheets/ad7280a.pdf)
- [Battery Cell Balancing: What to Balance and How](https://www.ti.com/lit/an/slua452a/slua452a.pdf)
