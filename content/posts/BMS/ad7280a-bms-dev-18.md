---
title: "AD7280A BMS 개발 삽질기 #18 - SOC 추정 기초"
date: 2024-12-13
draft: false
tags: ["AD7280A", "BMS", "STM32", "SOC", "쿨롱카운팅", "OCV"]
categories: ["BMS 개발"]
series: ["AD7280A BMS 개발"]
summary: "배터리 잔량은 어떻게 알까? 쿨롱 카운팅과 OCV 테이블의 장단점."
---

## 지난 글 요약

[지난 글](/posts/bms/ad7280a-bms-dev-17/)에서 CAN 통신 프로토콜을 설계했다. 이제 BMS의 핵심 기능 중 하나인 **SOC(State of Charge)** 추정을 다룬다.

## SOC란?

**SOC** = 현재 잔량 / 전체 용량 × 100%

```
┌────────────────────────────────────┐
│████████████████░░░░░░░░░░░░░░░░░░░░│ 50% SOC
└────────────────────────────────────┘
      현재 잔량        사용 가능
```

**왜 중요한가?**
- 사용자에게 잔량 표시
- 충전 종료 판단
- 방전 종료 판단 (과방전 보호)
- 밸런싱 결정

## SOC 추정 방법

| 방법 | 장점 | 단점 |
|------|------|------|
| 쿨롱 카운팅 | 실시간 추적 | 누적 오차 |
| OCV 테이블 | 절대값 보정 | 휴식 필요 |
| 칼만 필터 | 정확도 높음 | 구현 복잡 |
| 임피던스 기반 | 동적 추정 | 하드웨어 필요 |

**실용적 접근**: 쿨롱 카운팅 + OCV 보정

## 쿨롱 카운팅 (Coulomb Counting)

### 원리

전류 적분으로 용량 변화 추적:

```
SOC(t) = SOC(0) - ∫I(t)dt / Q_total × 100%
```

### 구현

```c
// soc_coulomb.c

typedef struct {
    float soc;              // 현재 SOC (0.0 ~ 100.0)
    float capacity_ah;      // 총 용량 (Ah)
    float accumulated_ah;   // 누적 방전량 (Ah)
    uint32_t last_update;   // 마지막 업데이트 시간
} soc_coulomb_t;

static soc_coulomb_t g_soc;

void SOC_Coulomb_Init(float initial_soc, float capacity_ah) {
    g_soc.soc = initial_soc;
    g_soc.capacity_ah = capacity_ah;
    g_soc.accumulated_ah = 0;
    g_soc.last_update = HAL_GetTick();
}

void SOC_Coulomb_Update(float current_a) {
    uint32_t now = HAL_GetTick();
    float dt_hours = (now - g_soc.last_update) / 3600000.0f;  // ms → h
    
    // 전류 적분 (방전: +, 충전: -)
    float delta_ah = current_a * dt_hours;
    g_soc.accumulated_ah += delta_ah;
    
    // SOC 계산
    g_soc.soc -= (delta_ah / g_soc.capacity_ah) * 100.0f;
    
    // 범위 제한
    if (g_soc.soc > 100.0f) g_soc.soc = 100.0f;
    if (g_soc.soc < 0.0f) g_soc.soc = 0.0f;
    
    g_soc.last_update = now;
}

float SOC_Coulomb_GetSOC(void) {
    return g_soc.soc;
}
```

### 충전 효율 보정

충전 시 100% 효율이 아님:

```c
void SOC_Coulomb_Update(float current_a) {
    // ...
    
    // 충전 효율 보정 (충전 시 약 95~98%)
    if (current_a < 0) {  // 충전
        delta_ah *= 0.97f;  // 97% 효율
    }
    
    // ...
}
```

### 쿨롱 카운팅의 문제점

1. **누적 오차**: 전류 센서 오차가 누적
2. **리셋 필요**: 시작 SOC를 알아야 함
3. **자가 방전**: 장기간 미사용 시 오차

```
시간 →
실제 SOC:  100% ────────────────────► 0%
추정 SOC:  100% ──────────────►┐
                               │ 오차 누적
                               ▼
```

## OCV (Open Circuit Voltage) 테이블

### 원리

**무부하 전압**과 **SOC**의 관계:

```
OCV (V)
  │
3.6├────────────────────────────╮
   │                            │
3.4├──────────────────────╮     │
   │                      │     │
3.2├────────────────╮     │     │
   │                │     │     │
3.0├──────────╮     │     │     │
   │          │     │     │     │
2.8├────╮     │     │     │     │
   │    │     │     │     │     │
   └────┴─────┴─────┴─────┴─────┴──► SOC (%)
        0    20    40    60    80   100
```

### LiFePO4 OCV 테이블

```c
// soc_ocv.c

// LiFePO4 OCV-SOC 테이블 (휴식 상태, 25°C)
// 10% 단위
static const float LFP_OCV_TABLE[11] = {
    2.80f,   // 0%
    3.10f,   // 10%
    3.20f,   // 20%
    3.25f,   // 30%
    3.27f,   // 40%
    3.28f,   // 50%
    3.29f,   // 60%
    3.30f,   // 70%
    3.32f,   // 80%
    3.35f,   // 90%
    3.60f,   // 100%
};
```

### OCV로 SOC 추정

```c
float SOC_OCV_GetSOC(float ocv_v) {
    // 범위 체크
    if (ocv_v <= LFP_OCV_TABLE[0]) return 0.0f;
    if (ocv_v >= LFP_OCV_TABLE[10]) return 100.0f;
    
    // 선형 보간
    for (int i = 0; i < 10; i++) {
        if (ocv_v >= LFP_OCV_TABLE[i] && ocv_v < LFP_OCV_TABLE[i+1]) {
            float ratio = (ocv_v - LFP_OCV_TABLE[i]) / 
                          (LFP_OCV_TABLE[i+1] - LFP_OCV_TABLE[i]);
            return (i + ratio) * 10.0f;
        }
    }
    
    return 50.0f;  // 기본값
}
```

### OCV의 문제점

1. **휴식 필요**: 부하 상태에서는 전압 강하
2. **LFP의 평탄 구간**: 20~80%에서 전압 변화 미미
3. **온도 의존성**: 저온에서 전압 낮음

```
LFP 특성:
│
│        ┌─────────────────────┐  ← 평탄 구간
│       /                       \    (구분 어려움)
│      /                         \
│     /                           \
└────/─────────────────────────────\────► SOC
    0%                            100%
```

## 하이브리드 방식

### 전략

```
┌─────────────────────────────────────────┐
│                                         │
│  ┌─────────────┐     ┌──────────────┐  │
│  │ 쿨롱 카운팅  │────►│   현재 SOC   │  │
│  └─────────────┘     └──────────────┘  │
│        ▲                    ▲          │
│        │                    │          │
│  실시간 추적            주기적 보정     │
│        │                    │          │
│  ┌─────────────┐     ┌──────────────┐  │
│  │  전류 센서   │     │  OCV 테이블   │  │
│  └─────────────┘     └──────────────┘  │
│                                         │
└─────────────────────────────────────────┘
```

### 구현

```c
// soc_hybrid.c

typedef struct {
    float soc_coulomb;      // 쿨롱 카운팅 SOC
    float soc_ocv;          // OCV 기반 SOC
    float soc_final;        // 최종 SOC
    uint32_t rest_time_ms;  // 휴식 시간
    bool is_resting;        // 휴식 상태
    float last_current;     // 이전 전류값
} soc_hybrid_t;

static soc_hybrid_t g_soc_hybrid;

#define REST_THRESHOLD_A    0.5f    // 휴식 판정 전류 (A)
#define REST_TIME_MS        30000   // 휴식 시간 (30초)
#define OCV_WEIGHT          0.3f    // OCV 보정 가중치

void SOC_Hybrid_Update(float current_a, float cell_voltage_v) {
    // 쿨롱 카운팅 업데이트 (항상)
    SOC_Coulomb_Update(current_a);
    g_soc_hybrid.soc_coulomb = SOC_Coulomb_GetSOC();
    
    // 휴식 상태 판정
    if (fabsf(current_a) < REST_THRESHOLD_A) {
        if (!g_soc_hybrid.is_resting) {
            g_soc_hybrid.rest_time_ms = 0;
            g_soc_hybrid.is_resting = true;
        }
        g_soc_hybrid.rest_time_ms += 100;  // 100ms 주기 가정
    } else {
        g_soc_hybrid.is_resting = false;
        g_soc_hybrid.rest_time_ms = 0;
    }
    
    // OCV 보정 (충분한 휴식 후)
    if (g_soc_hybrid.rest_time_ms >= REST_TIME_MS) {
        g_soc_hybrid.soc_ocv = SOC_OCV_GetSOC(cell_voltage_v);
        
        // 가중 평균으로 보정
        g_soc_hybrid.soc_coulomb = 
            g_soc_hybrid.soc_coulomb * (1.0f - OCV_WEIGHT) +
            g_soc_hybrid.soc_ocv * OCV_WEIGHT;
        
        // 쿨롱 카운터에 반영
        SOC_Coulomb_SetSOC(g_soc_hybrid.soc_coulomb);
    }
    
    g_soc_hybrid.soc_final = g_soc_hybrid.soc_coulomb;
    g_soc_hybrid.last_current = current_a;
}
```

### 엔드포인트 보정

**Full/Empty 상태에서 강제 보정**:

```c
void SOC_EndpointCorrection(float cell_voltage_v, float current_a) {
    // 충전 완료 검출 (CV 모드에서 전류 감소)
    if (cell_voltage_v >= 3.60f && current_a < -0.1f && current_a > -0.5f) {
        // 충전 종료 → SOC = 100%
        SOC_Coulomb_SetSOC(100.0f);
    }
    
    // 방전 완료 검출 (저전압 도달)
    if (cell_voltage_v <= 2.50f) {
        // 방전 종료 → SOC = 0%
        SOC_Coulomb_SetSOC(0.0f);
    }
}
```

## 전류 센서

### 션트 저항 방식

```
    ┌─────────────────────────────────┐
    │                                 │
    │    Battery Pack                 │
    │                                 │
    └─────────────┬───────────────────┘
                  │
                  ▼
              ┌───────┐
    ─────────►│ Shunt │────────►  Load
              │ 1mΩ   │
              └───┬───┘
                  │
                  ▼
              ┌───────┐
              │  ADC  │ ← 전압 측정
              └───────┘
```

```c
// current_sense.c

#define SHUNT_RESISTANCE_MOHM   1.0f    // 1mΩ
#define ADC_VREF_MV             3300.0f
#define ADC_RESOLUTION          4096
#define CURRENT_GAIN            50.0f   // INA210 등 증폭기

float Current_GetAmps(uint16_t adc_value) {
    float voltage_mv = (adc_value / (float)ADC_RESOLUTION) * ADC_VREF_MV;
    float shunt_mv = voltage_mv / CURRENT_GAIN;
    float current_a = shunt_mv / SHUNT_RESISTANCE_MOHM;
    
    return current_a;
}
```

### 홀 센서 방식

비접촉 측정, 절연 필요 시 사용:

```c
// ACS712 예시
#define ACS712_SENSITIVITY_MV_A   66.0f   // 66mV/A (30A 모델)
#define ACS712_OFFSET_MV          2500.0f // 0A = 2.5V

float Current_Hall_GetAmps(uint16_t adc_value) {
    float voltage_mv = (adc_value / (float)ADC_RESOLUTION) * ADC_VREF_MV;
    float current_a = (voltage_mv - ACS712_OFFSET_MV) / ACS712_SENSITIVITY_MV_A;
    
    return current_a;
}
```

## 온도 보정

저온에서 용량 감소:

```c
// 온도 계수 테이블
static const float TEMP_FACTOR[] = {
    0.70f,  // -20°C
    0.80f,  // -10°C
    0.90f,  //   0°C
    0.95f,  //  10°C
    1.00f,  //  20°C (기준)
    1.00f,  //  30°C
    0.98f,  //  40°C
    0.95f,  //  50°C
};

float SOC_GetEffectiveCapacity(float base_capacity, int8_t temp_c) {
    int idx = (temp_c + 20) / 10;
    if (idx < 0) idx = 0;
    if (idx > 7) idx = 7;
    
    return base_capacity * TEMP_FACTOR[idx];
}
```

## 삽질: LFP 평탄 구간

LFP는 20~80%에서 OCV 변화가 **50mV 이내**:

```c
// 잘못된 접근: OCV만으로 SOC 추정
float soc = SOC_OCV_GetSOC(3.28f);  // 50%? 60%? 알 수 없음!

// 올바른 접근: 쿨롱 카운팅 주력, OCV는 보조
```

## 삽질: 자가 방전

장기간 미사용 후 SOC 오차:

```c
// 자가 방전 보정 (월 2~3%)
void SOC_SelfDischargeCorrection(uint32_t idle_days) {
    float loss_per_day = 0.1f;  // 0.1%/day
    float total_loss = loss_per_day * idle_days;
    
    g_soc.soc -= total_loss;
    if (g_soc.soc < 0) g_soc.soc = 0;
}
```

## 정리

| 방법 | 용도 | 정확도 |
|------|------|--------|
| 쿨롱 카운팅 | 실시간 추적 | 중 (누적 오차) |
| OCV 테이블 | 휴식 시 보정 | 높음 (조건부) |
| 엔드포인트 | Full/Empty 보정 | 높음 |
| 온도 보정 | 용량 조정 | 높음 |

**실용적 조합**:
1. 쿨롱 카운팅으로 실시간 추적
2. 휴식 시 OCV로 보정
3. 충전/방전 완료 시 엔드포인트 리셋
4. 온도에 따른 용량 조정

**다음 글에서**: 데이터 로깅 - Flash 저장과 이력 관리.

---

## 시리즈 네비게이션

**Part 6: 통신 & 진단편**
- [#17 - CAN 통신 프로토콜 설계](/posts/bms/ad7280a-bms-dev-17/)
- **#18 - SOC 추정 기초** ← 현재 글
- #19 - 데이터 로깅
- #20 - 진단 인터페이스

---

## 참고 자료

- [Battery SOC Estimation Methods](https://www.mdpi.com/2079-9292/10/22/2835)
- [LiFePO4 Characteristics](https://batteryuniversity.com/article/bu-205-types-of-lithium-ion)
- [Coulomb Counting Implementation](https://www.ti.com/lit/an/slua450/slua450.pdf)
