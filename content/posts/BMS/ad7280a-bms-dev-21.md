---
title: "AD7280A BMS 개발 삽질기 #21 - SOH 추정"
date: 2024-12-14
draft: false
tags: ["AD7280A", "BMS", "STM32", "SOH", "배터리수명", "내부저항"]
categories: ["BMS 개발"]
series: ["AD7280A BMS 개발"]
summary: "배터리 건강 상태는 어떻게 알까? 용량 열화와 내부 저항으로 SOH를 추정한다."
---

## 지난 글 요약

[Part 6](/posts/bms/ad7280a-bms-dev-20/)에서 통신과 진단을 완료했다. CAN 프로토콜, SOC, 로깅, CLI. 이제 **고급 기능**으로 넘어가자. 첫 번째는 **SOH(State of Health)**.

## SOH란?

**SOH** = 현재 최대 용량 / 초기 용량 × 100%

```
새 배터리:    [████████████████████] 100Ah = 100% SOH
3년 후:      [████████████████    ] 80Ah  = 80% SOH
수명 종료:   [████████████        ] 60Ah  = 60% SOH (교체 권장)
```

**왜 중요한가?**
- 배터리 교체 시기 판단
- 잔여 수명 예측
- 보증(Warranty) 판정
- 중고 배터리 가치 평가

## SOH 추정 방법

| 방법 | 정확도 | 복잡도 | 실시간 |
|------|--------|--------|--------|
| 용량 기반 | 높음 | 중간 | ❌ |
| 내부 저항 기반 | 중간 | 낮음 | ✅ |
| EIS (임피던스) | 매우 높음 | 높음 | ❌ |
| 머신 러닝 | 높음 | 매우 높음 | ✅ |

## 방법 1: 용량 기반 SOH

### 원리

**완전 충방전 사이클**에서 실제 용량 측정:

```
충전 시작 (0% SOC) ──────────────► 충전 완료 (100% SOC)
                    
                    ∫I(t)dt = 실제 용량 (Ah)
```

### 구현

```c
// soh_capacity.c

typedef struct {
    float initial_capacity_ah;   // 공칭 용량
    float measured_capacity_ah;  // 측정 용량
    float soh_capacity;          // 용량 기반 SOH (%)
    
    // 측정 상태
    bool measuring;
    float accumulated_ah;
    uint8_t start_soc;
} soh_capacity_t;

static soh_capacity_t g_soh_cap;

void SOH_Capacity_Init(float initial_capacity) {
    g_soh_cap.initial_capacity_ah = initial_capacity;
    g_soh_cap.measured_capacity_ah = initial_capacity;
    g_soh_cap.soh_capacity = 100.0f;
    g_soh_cap.measuring = false;
}

void SOH_Capacity_StartMeasurement(uint8_t current_soc) {
    // 낮은 SOC에서 시작해야 정확
    if (current_soc > 10) {
        return;  // 10% 이하에서만 시작
    }
    
    g_soh_cap.measuring = true;
    g_soh_cap.accumulated_ah = 0;
    g_soh_cap.start_soc = current_soc;
}

void SOH_Capacity_Update(float current_a) {
    if (!g_soh_cap.measuring) return;
    
    // 충전 전류만 적분 (음수 = 충전)
    if (current_a < 0) {
        float dt_hours = 0.1f / 3600.0f;  // 100ms 주기
        g_soh_cap.accumulated_ah += (-current_a) * dt_hours;
    }
}

void SOH_Capacity_EndMeasurement(uint8_t current_soc) {
    if (!g_soh_cap.measuring) return;
    
    // 높은 SOC에서 종료해야 정확
    if (current_soc < 95) {
        return;  // 95% 이상에서만 종료
    }
    
    g_soh_cap.measuring = false;
    
    // SOC 범위 보정
    float soc_range = (current_soc - g_soh_cap.start_soc) / 100.0f;
    
    // 실제 용량 계산
    g_soh_cap.measured_capacity_ah = g_soh_cap.accumulated_ah / soc_range;
    
    // SOH 계산
    g_soh_cap.soh_capacity = 
        (g_soh_cap.measured_capacity_ah / g_soh_cap.initial_capacity_ah) * 100.0f;
    
    // 범위 제한
    if (g_soh_cap.soh_capacity > 100.0f) g_soh_cap.soh_capacity = 100.0f;
    if (g_soh_cap.soh_capacity < 0.0f) g_soh_cap.soh_capacity = 0.0f;
}
```

### 자동 측정 트리거

```c
void SOH_Capacity_AutoTrigger(void) {
    static bool was_charging = false;
    bool is_charging = (g_bms.pack_current_ma < -100);  // 100mA 이상 충전
    
    // 충전 시작 감지
    if (is_charging && !was_charging) {
        if (g_bms.soc <= 10) {
            SOH_Capacity_StartMeasurement(g_bms.soc);
        }
    }
    
    // 충전 완료 감지
    if (!is_charging && was_charging) {
        if (g_bms.soc >= 95) {
            SOH_Capacity_EndMeasurement(g_bms.soc);
        }
    }
    
    was_charging = is_charging;
}
```

## 방법 2: 내부 저항 기반 SOH

### 원리

배터리 열화 → **내부 저항 증가**:

```
새 배터리:     Ri = 5mΩ
열화된 배터리:  Ri = 10mΩ  (2배 증가 → SOH 낮음)
```

내부 저항 측정:

```
        ΔV = V_noload - V_load
Ri = ─────────────────────────
              ΔI
```

### 구현

```c
// soh_resistance.c

typedef struct {
    float internal_resistance_mohm;  // 현재 내부 저항 (mΩ)
    float initial_resistance_mohm;   // 초기 내부 저항
    float soh_resistance;            // 저항 기반 SOH (%)
    
    // 측정 버퍼
    float voltage_samples[10];
    float current_samples[10];
    uint8_t sample_index;
    bool buffer_full;
} soh_resistance_t;

static soh_resistance_t g_soh_res;

void SOH_Resistance_Init(float initial_resistance) {
    g_soh_res.initial_resistance_mohm = initial_resistance;
    g_soh_res.internal_resistance_mohm = initial_resistance;
    g_soh_res.soh_resistance = 100.0f;
    g_soh_res.sample_index = 0;
    g_soh_res.buffer_full = false;
}

void SOH_Resistance_AddSample(float voltage_v, float current_a) {
    g_soh_res.voltage_samples[g_soh_res.sample_index] = voltage_v;
    g_soh_res.current_samples[g_soh_res.sample_index] = current_a;
    
    g_soh_res.sample_index++;
    if (g_soh_res.sample_index >= 10) {
        g_soh_res.sample_index = 0;
        g_soh_res.buffer_full = true;
    }
}

float SOH_Resistance_Calculate(void) {
    if (!g_soh_res.buffer_full) return 0;
    
    // 선형 회귀: V = OCV - I * Ri
    // 최소자승법으로 Ri 추정
    
    float sum_i = 0, sum_v = 0, sum_ii = 0, sum_iv = 0;
    int n = 10;
    
    for (int i = 0; i < n; i++) {
        float I = g_soh_res.current_samples[i];
        float V = g_soh_res.voltage_samples[i];
        
        sum_i += I;
        sum_v += V;
        sum_ii += I * I;
        sum_iv += I * V;
    }
    
    // 기울기 = -Ri
    float slope = (n * sum_iv - sum_i * sum_v) / 
                  (n * sum_ii - sum_i * sum_i);
    
    float ri_ohm = -slope;
    g_soh_res.internal_resistance_mohm = ri_ohm * 1000.0f;  // Ω → mΩ
    
    return g_soh_res.internal_resistance_mohm;
}
```

### 저항 기반 SOH 계산

```c
void SOH_Resistance_UpdateSOH(void) {
    float ri = SOH_Resistance_Calculate();
    if (ri <= 0) return;
    
    float ri_initial = g_soh_res.initial_resistance_mohm;
    float ri_eol = ri_initial * 2.0f;  // EOL = 초기의 2배
    
    // 선형 보간
    // ri_initial → 100% SOH
    // ri_eol → 0% SOH
    
    g_soh_res.soh_resistance = 
        100.0f * (ri_eol - ri) / (ri_eol - ri_initial);
    
    // 범위 제한
    if (g_soh_res.soh_resistance > 100.0f) g_soh_res.soh_resistance = 100.0f;
    if (g_soh_res.soh_resistance < 0.0f) g_soh_res.soh_resistance = 0.0f;
}
```

### 펄스 방전 기반 측정

더 정확한 측정을 위해 **펄스 부하** 사용:

```c
float SOH_Resistance_PulseMeasure(void) {
    // 1. 휴식 상태 전압 측정 (OCV)
    float v_rest = BMS_GetPackVoltage();
    float i_rest = BMS_GetPackCurrent();  // ~0A
    
    // 2. 부하 인가 (테스트 저항 또는 실제 부하)
    // 외부에서 부하 스위치 제어 필요
    
    HAL_Delay(100);  // 안정화 대기
    
    // 3. 부하 상태 전압 측정
    float v_load = BMS_GetPackVoltage();
    float i_load = BMS_GetPackCurrent();  // 예: 10A
    
    // 4. 내부 저항 계산
    float delta_v = v_rest - v_load;
    float delta_i = i_load - i_rest;
    
    if (fabsf(delta_i) < 1.0f) {
        return 0;  // 전류 변화 부족
    }
    
    float ri_ohm = delta_v / delta_i;
    
    return ri_ohm * 1000.0f;  // mΩ
}
```

## 통합 SOH 추정

```c
// soh_combined.c

typedef struct {
    float soh_capacity;      // 용량 기반 (0~100%)
    float soh_resistance;    // 저항 기반 (0~100%)
    float soh_combined;      // 통합 SOH
    float weight_capacity;   // 용량 가중치
    float weight_resistance; // 저항 가중치
} soh_combined_t;

static soh_combined_t g_soh;

void SOH_Combined_Init(void) {
    g_soh.soh_capacity = 100.0f;
    g_soh.soh_resistance = 100.0f;
    g_soh.soh_combined = 100.0f;
    
    // 가중치 설정 (용량이 더 신뢰성 높음)
    g_soh.weight_capacity = 0.7f;
    g_soh.weight_resistance = 0.3f;
}

void SOH_Combined_Update(void) {
    g_soh.soh_capacity = g_soh_cap.soh_capacity;
    g_soh.soh_resistance = g_soh_res.soh_resistance;
    
    // 가중 평균
    g_soh.soh_combined = 
        g_soh.soh_capacity * g_soh.weight_capacity +
        g_soh.soh_resistance * g_soh.weight_resistance;
    
    // 두 방법의 차이가 크면 낮은 값 사용 (보수적)
    float diff = fabsf(g_soh.soh_capacity - g_soh.soh_resistance);
    if (diff > 20.0f) {
        g_soh.soh_combined = fminf(g_soh.soh_capacity, g_soh.soh_resistance);
    }
}

float SOH_GetSOH(void) {
    return g_soh.soh_combined;
}
```

## 사이클 카운트 기반 추정

경험적 모델:

```c
// 사이클 수 기반 SOH 추정
float SOH_EstimateFromCycles(uint16_t cycle_count) {
    // LiFePO4: 약 3000 사이클에서 80% SOH
    // 선형 모델 (실제로는 비선형)
    
    float cycles_to_80pct = 3000.0f;
    float degradation_per_cycle = 20.0f / cycles_to_80pct;
    
    float soh = 100.0f - (cycle_count * degradation_per_cycle);
    
    if (soh < 0) soh = 0;
    
    return soh;
}
```

## 온도 보정

고온 운영 = 빠른 열화:

```c
// 온도에 따른 열화 가속 계수
float SOH_GetDegradationFactor(int8_t temp_c) {
    // 25°C 기준
    if (temp_c <= 25) {
        return 1.0f;
    } else if (temp_c <= 35) {
        return 1.5f;  // 10°C 상승 → 1.5배
    } else if (temp_c <= 45) {
        return 2.0f;  // 20°C 상승 → 2배
    } else {
        return 3.0f;  // 45°C 이상 → 3배
    }
}

void SOH_AccumulateDegradation(float dt_hours, int8_t temp_c) {
    static float accumulated_stress = 0;
    
    float factor = SOH_GetDegradationFactor(temp_c);
    accumulated_stress += dt_hours * factor;
    
    // 1000시간 스트레스 = 1% 열화 (예시)
    float degradation = accumulated_stress / 1000.0f;
    
    g_soh.soh_combined -= degradation;
    if (g_soh.soh_combined < 0) g_soh.soh_combined = 0;
}
```

## 삽질: 측정 조건

SOH 측정은 **조건이 중요**:

```c
// 잘못된 측정
// - 충전/방전 중 측정 → 분극 영향
// - 온도 변동 중 → OCV 변화
// - 부분 충방전 → 용량 과소평가

// 올바른 측정 조건
bool SOH_IsMeasurementValid(void) {
    // 1. 충분한 휴식 (분극 해소)
    if (g_bms.rest_time_ms < 60000) return false;  // 최소 1분
    
    // 2. 안정된 온도
    int8_t temp_diff = g_bms.max_temp - g_bms.min_temp;
    if (temp_diff > 5) return false;  // 5°C 이내
    
    // 3. 적절한 SOC 범위
    if (g_bms.soc < 20 || g_bms.soc > 80) return false;
    
    return true;
}
```

## 삽질: 내부 저항의 SOC 의존성

내부 저항은 SOC에 따라 변함:

```
Ri (mΩ)
   │
12 ├────╮                    ╭────
   │     ╲                  ╱
 8 ├──────╲────────────────╱──────
   │       ╲              ╱
 4 ├────────╲────────────╱────────
   │         ╲──────────╱
   └────┴────┴────┴────┴────┴────► SOC (%)
        0   20   40   60   80  100
```

**해결책**: 같은 SOC에서 측정 비교

```c
// SOC 50% 기준 내부 저항 보정
float SOH_NormalizeResistance(float ri_measured, uint8_t soc) {
    // SOC별 보정 계수 (실험 데이터)
    static const float soc_factor[] = {
        1.20f,  // 0%
        1.10f,  // 20%
        1.05f,  // 40%
        1.00f,  // 60% (기준)
        1.02f,  // 80%
        1.15f,  // 100%
    };
    
    int idx = soc / 20;
    if (idx > 5) idx = 5;
    
    return ri_measured / soc_factor[idx];
}
```

## 정리

| 방법 | 장점 | 단점 | 용도 |
|------|------|------|------|
| 용량 기반 | 정확함 | 완전 사이클 필요 | 정기 점검 |
| 저항 기반 | 실시간 | SOC 의존성 | 상시 모니터링 |
| 사이클 기반 | 간단함 | 부정확 | 참고용 |
| 통합 | 균형 잡힘 | 복잡 | 제품 적용 |

**다음 글에서**: 칼만 필터 적용 - SOC 추정 정밀도 향상.

---

## 시리즈 네비게이션

**Part 7: 고급 기능편**
- **#21 - SOH 추정** ← 현재 글
- #22 - 칼만 필터 적용
- #23 - 프리차지 회로
- #24 - 절연 모니터링

---

## 참고 자료

- [Battery SOH Estimation](https://www.sciencedirect.com/science/article/pii/S2352152X20313360)
- [Internal Resistance Measurement](https://batteryuniversity.com/article/bu-902-how-to-measure-internal-resistance)
- [LiFePO4 Cycle Life](https://www.epectec.com/batteries/cell-comparison.html)
