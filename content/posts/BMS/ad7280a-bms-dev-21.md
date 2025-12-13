---
title: "AD7280A BMS 개발 삽질기 #21 - SOH 추정"
date: 2024-12-13
draft: false
tags: ["AD7280A", "BMS", "STM32", "SOH", "배터리수명", "내부저항"]
categories: ["BMS 개발"]
series: ["AD7280A BMS 개발"]
summary: "배터리가 얼마나 늙었는지 어떻게 알까? SOH 추정의 두 가지 접근법 - 용량 열화와 내부 저항."
---

## 지난 글 요약

[Part 6](/posts/bms/ad7280a-bms-dev-20/)에서 진단 인터페이스를 구현했다. CLI, CAN 진단. 이제 **고급 기능**으로 넘어가자. 첫 번째는 **SOH(State of Health)**.

## SOH란?

**SOH** = 현재 상태 / 새 배터리 상태 × 100%

```
새 배터리:     SOH 100%
┌────────────────────────────────────┐
│████████████████████████████████████│ 100Ah 용량
└────────────────────────────────────┘

3년 사용 후:   SOH 80%
┌────────────────────────────────────┐
│████████████████████████████░░░░░░░░│ 80Ah 용량
└────────────────────────────────────┘
             ↑ 20% 열화
```

## SOH 추정 방법

| 방법 | 측정 대상 | 정확도 | 복잡도 |
|------|----------|--------|--------|
| 용량 기반 | 실제 방전 용량 | 높음 | 낮음 |
| 저항 기반 | 내부 저항 증가 | 중간 | 중간 |
| EIS | 임피던스 스펙트럼 | 매우 높음 | 높음 |
| 모델 기반 | 전기화학 모델 | 높음 | 매우 높음 |

**실용적 선택**: 용량 + 저항 조합

## 방법 1: 용량 기반 SOH

### 원리

완충 → 완방 사이클에서 실제 용량 측정:

```
SOH_capacity = (실측 용량 / 정격 용량) × 100%
```

### 구현

```c
// soh_capacity.c

typedef struct {
    float nominal_capacity_ah;    // 정격 용량
    float measured_capacity_ah;   // 실측 용량
    float soh_capacity;           // 용량 기반 SOH (%)
    bool measurement_valid;
    
    // 측정 상태
    float charge_start_soc;
    float discharge_start_ah;
    float total_discharged_ah;
} soh_capacity_t;

static soh_capacity_t g_soh_cap;

void SOH_Capacity_Init(float nominal_ah) {
    g_soh_cap.nominal_capacity_ah = nominal_ah;
    g_soh_cap.measured_capacity_ah = nominal_ah;  // 초기값
    g_soh_cap.soh_capacity = 100.0f;
    g_soh_cap.measurement_valid = false;
}
```

### 용량 측정 트리거

완충 후 완방 시 측정:

```c
void SOH_Capacity_OnChargeComplete(void) {
    // 충전 완료 시점 기록
    g_soh_cap.charge_start_soc = 100.0f;
    g_soh_cap.discharge_start_ah = 0;
    g_soh_cap.total_discharged_ah = 0;
}

void SOH_Capacity_OnDischargeComplete(void) {
    // 방전 완료 → 용량 계산
    if (g_soh_cap.charge_start_soc >= 95.0f) {  // 거의 완충에서 시작
        g_soh_cap.measured_capacity_ah = g_soh_cap.total_discharged_ah;
        
        g_soh_cap.soh_capacity = 
            (g_soh_cap.measured_capacity_ah / g_soh_cap.nominal_capacity_ah) * 100.0f;
        
        // 범위 제한
        if (g_soh_cap.soh_capacity > 100.0f) g_soh_cap.soh_capacity = 100.0f;
        if (g_soh_cap.soh_capacity < 0.0f) g_soh_cap.soh_capacity = 0.0f;
        
        g_soh_cap.measurement_valid = true;
        
        // 통계에 기록
        Log_Event(EVT_SOH_MEASURED, 0, 
                  (uint16_t)(g_soh_cap.soh_capacity * 10),
                  (uint16_t)(g_soh_cap.measured_capacity_ah * 10), 0);
    }
}

void SOH_Capacity_Update(float current_a, float dt_hours) {
    if (current_a > 0) {  // 방전 중
        g_soh_cap.total_discharged_ah += current_a * dt_hours;
    }
}
```

### 용량 기반 SOH의 한계

1. **완전 사이클 필요**: 완충→완방이 드묾
2. **시간 소요**: 대용량 배터리는 수 시간
3. **사용 패턴 의존**: 부분 사이클만 사용하면 측정 불가

## 방법 2: 내부 저항 기반 SOH

### 원리

배터리 노화 → 내부 저항 증가:

```
        새 배터리              노화된 배터리
        ┌─────┐               ┌─────┐
    ─┤├─┤ R_i ├─         ─┤├─┤ R_i'├─
        └─────┘               └─────┘
        R_i = 5mΩ            R_i' = 10mΩ
                                ↑ 2배 증가
```

### DC 펄스 방식

전류 펄스를 인가하고 전압 강하 측정:

```c
// soh_resistance.c

typedef struct {
    float internal_resistance_mohm;   // 내부 저항 (mΩ)
    float initial_resistance_mohm;    // 초기 저항 (새 배터리)
    float soh_resistance;             // 저항 기반 SOH (%)
    
    // 측정용
    float v_before;
    float v_after;
    float pulse_current;
} soh_resistance_t;

static soh_resistance_t g_soh_res;

float SOH_Resistance_Measure(void) {
    // 1. 휴식 상태 전압 측정
    g_soh_res.v_before = BMS_GetPackVoltage();
    
    // 2. 부하 인가 (또는 자연 발생 펄스 이용)
    // 실제로는 방전 시작 순간의 전압 강하 이용
    HAL_Delay(100);  // 100ms 후
    
    // 3. 부하 상태 전압 측정
    g_soh_res.v_after = BMS_GetPackVoltage();
    g_soh_res.pulse_current = BMS_GetPackCurrent();
    
    // 4. 저항 계산: R = ΔV / I
    if (fabsf(g_soh_res.pulse_current) > 1.0f) {  // 최소 1A
        float delta_v = g_soh_res.v_before - g_soh_res.v_after;
        g_soh_res.internal_resistance_mohm = 
            (delta_v * 1000.0f) / g_soh_res.pulse_current;
    }
    
    return g_soh_res.internal_resistance_mohm;
}
```

### 방전 시작 순간 이용

자연스러운 방전 시작 시점 활용:

```c
void SOH_Resistance_OnLoadChange(float v_before, float v_after, 
                                  float current_before, float current_after) {
    // 전류 변화가 충분할 때만
    float delta_i = fabsf(current_after - current_before);
    
    if (delta_i > 5.0f) {  // 5A 이상 변화
        float delta_v = v_before - v_after;
        
        // 팩 저항 계산
        float pack_r = (delta_v * 1000.0f) / delta_i;  // mΩ
        
        // 셀당 저항 (24셀 직렬)
        g_soh_res.internal_resistance_mohm = pack_r / 24.0f;
        
        // SOH 계산
        SOH_Resistance_Calculate();
    }
}

void SOH_Resistance_Calculate(void) {
    // 저항 증가율로 SOH 추정
    // 일반적으로 저항 2배 = EOL (End of Life)
    
    float resistance_ratio = g_soh_res.internal_resistance_mohm / 
                             g_soh_res.initial_resistance_mohm;
    
    // 선형 모델: SOH = 100% - (ratio - 1) × 100%
    // ratio 1.0 → SOH 100%
    // ratio 2.0 → SOH 0%
    
    g_soh_res.soh_resistance = (2.0f - resistance_ratio) * 100.0f;
    
    // 범위 제한
    if (g_soh_res.soh_resistance > 100.0f) g_soh_res.soh_resistance = 100.0f;
    if (g_soh_res.soh_resistance < 0.0f) g_soh_res.soh_resistance = 0.0f;
}
```

### 온도 보정

저항은 온도에 민감:

```c
// 25°C 기준으로 보정
float SOH_Resistance_TempCompensate(float measured_r, int8_t temp_c) {
    // LiFePO4 온도 계수: 약 0.5%/°C
    float temp_factor = 1.0f + 0.005f * (temp_c - 25);
    
    return measured_r / temp_factor;
}
```

## 복합 SOH 계산

두 방법 조합:

```c
// soh_combined.c

typedef struct {
    float soh_capacity;      // 용량 기반 (%)
    float soh_resistance;    // 저항 기반 (%)
    float soh_combined;      // 최종 SOH (%)
    float confidence;        // 신뢰도 (0~1)
} soh_combined_t;

static soh_combined_t g_soh;

void SOH_UpdateCombined(void) {
    float w_cap = 0.7f;   // 용량 가중치
    float w_res = 0.3f;   // 저항 가중치
    
    // 용량 측정이 유효하면 가중치 높임
    if (g_soh_cap.measurement_valid) {
        w_cap = 0.8f;
        w_res = 0.2f;
    }
    
    // 용량 측정이 오래됐으면 저항 가중치 높임
    if (g_soh_cap.days_since_measurement > 30) {
        w_cap = 0.5f;
        w_res = 0.5f;
    }
    
    g_soh.soh_combined = 
        g_soh.soh_capacity * w_cap + 
        g_soh.soh_resistance * w_res;
    
    // 신뢰도 계산
    float diff = fabsf(g_soh.soh_capacity - g_soh.soh_resistance);
    g_soh.confidence = 1.0f - (diff / 50.0f);  // 차이 50%면 신뢰도 0
    if (g_soh.confidence < 0) g_soh.confidence = 0;
}
```

## 사이클 카운트 기반 추정

측정 없이 대략적 추정:

```c
// 사이클 수명 모델
// LiFePO4: 약 3000 사이클 @ 80% DOD
#define CYCLE_LIFE_ESTIMATE     3000

float SOH_FromCycleCount(uint16_t cycles) {
    // 선형 모델
    float degradation = (float)cycles / CYCLE_LIFE_ESTIMATE;
    float soh = (1.0f - degradation) * 100.0f;
    
    if (soh < 0) soh = 0;
    if (soh > 100) soh = 100;
    
    return soh;
}
```

## SOH 기반 동작 조정

### 충전 전류 제한

```c
float BMS_GetMaxChargeCurrent(void) {
    float base_current = 50.0f;  // 정격 1C
    
    // SOH에 따른 제한
    float soh = g_soh.soh_combined;
    
    if (soh >= 80.0f) {
        return base_current;  // 100%
    } else if (soh >= 60.0f) {
        return base_current * 0.8f;  // 80%
    } else if (soh >= 40.0f) {
        return base_current * 0.5f;  // 50%
    } else {
        return base_current * 0.3f;  // 30%
    }
}
```

### 사용 가능 용량 조정

```c
float BMS_GetUsableCapacity(void) {
    return g_bms.nominal_capacity_ah * (g_soh.soh_combined / 100.0f);
}
```

### EOL 경고

```c
void SOH_CheckWarnings(void) {
    if (g_soh.soh_combined < 80.0f && g_soh.soh_combined >= 70.0f) {
        // 주의: 배터리 교체 권장
        BMS_SetWarning(WARN_SOH_LOW);
    } else if (g_soh.soh_combined < 70.0f) {
        // 경고: 배터리 교체 필요
        BMS_SetWarning(WARN_SOH_CRITICAL);
    }
}
```

## 삽질: 저항 측정 노이즈

전압/전류 측정 노이즈로 저항 오차 큼:

```c
// 잘못된 코드: 단일 측정
float r = delta_v / delta_i;  // 노이즈 영향 큼

// 올바른 코드: 이동 평균
#define R_FILTER_SIZE   10
static float r_samples[R_FILTER_SIZE];
static int r_index = 0;

void SOH_AddResistanceSample(float r) {
    r_samples[r_index++] = r;
    if (r_index >= R_FILTER_SIZE) r_index = 0;
}

float SOH_GetFilteredResistance(void) {
    float sum = 0;
    for (int i = 0; i < R_FILTER_SIZE; i++) {
        sum += r_samples[i];
    }
    return sum / R_FILTER_SIZE;
}
```

## 삽질: 온도 미보정

25°C에서 5mΩ인 셀이 0°C에서 7mΩ 측정:

```c
// 잘못된 해석
// "저항이 40% 증가! SOH 60%?"

// 올바른 해석
float compensated = SOH_Resistance_TempCompensate(7.0f, 0);
// → 약 5.3mΩ (온도 보정 후)
// SOH 거의 100%
```

## 정리

| 방법 | 장점 | 단점 |
|------|------|------|
| 용량 기반 | 정확함 | 완전 사이클 필요 |
| 저항 기반 | 빠른 측정 | 온도 영향 |
| 사이클 카운트 | 간단함 | 부정확 |
| 복합 | 균형 | 복잡 |

**실용적 조합**:
1. 평소: 저항 + 사이클 카운트
2. 완전 사이클 시: 용량 측정 업데이트
3. 두 방법 교차 검증

**다음 글에서**: 칼만 필터 적용 - SOC 정밀도 향상.

---

## 시리즈 네비게이션

**Part 7: 고급 기능편**
- **#21 - SOH 추정** ← 현재 글
- #22 - 칼만 필터 적용
- #23 - 프리차지 회로
- #24 - 절연 모니터링

---

## 참고 자료

- [Battery SOH Estimation](https://www.mdpi.com/2079-9292/10/22/2835)
- [Internal Resistance Measurement](https://batteryuniversity.com/article/bu-902-how-to-measure-internal-resistance)
- [LiFePO4 Aging Characteristics](https://www.sciencedirect.com/science/article/pii/S0378775315004449)
