---
title: "AD7280A BMS 개발 삽질기 #22 - 칼만 필터 적용"
date: 2024-12-14
draft: false
tags: ["AD7280A", "BMS", "STM32", "칼만필터", "SOC", "상태추정"]
categories: ["BMS 개발"]
series: ["AD7280A BMS 개발"]
summary: "쿨롱 카운팅의 누적 오차를 잡자! 칼만 필터로 SOC 추정 정밀도를 높인다."
---

## 지난 글 요약

[지난 글](/posts/bms/ad7280a-bms-dev-21/)에서 SOH 추정을 다뤘다. 용량 기반, 저항 기반. 이제 **SOC 추정의 정밀도**를 높여보자. 칼만 필터로!

## 왜 칼만 필터인가?

### 쿨롱 카운팅의 문제

```
시간 →
─────────────────────────────────────────►

실제 SOC:    100% ──────────────────────► 0%
추정 SOC:    100% ────────────────►┐
                                   │ 누적 오차!
                                   ▼
                              (실제보다 높게 추정)
```

**문제점**:
- 전류 센서 오차가 시간에 따라 누적
- 초기값 오차 계속 유지
- 자가 방전 미반영

### OCV 테이블의 문제

```
LFP 특성:
│        ┌─────────────────────┐
│       /                       \
│      /    평탄 구간            \
│     /   (SOC 구분 불가)         \
└────/─────────────────────────────\───► SOC
```

**문제점**:
- 20~80%에서 전압 변화 미미
- 부하 중에는 사용 불가
- 온도 영향 받음

### 칼만 필터의 해결책

```
┌─────────────────────────────────────────┐
│         Kalman Filter                   │
│                                         │
│  쿨롱 카운팅 ──────┐                    │
│  (동적 추적)       │   최적 조합        │
│                   ├─────────────► SOC   │
│  OCV 테이블 ──────┘                     │
│  (절대 기준)                            │
│                                         │
└─────────────────────────────────────────┘
```

## 칼만 필터 기초

### 상태 방정식

```
상태 변수:  x = [SOC]

상태 전이:  SOC(k+1) = SOC(k) - (I * Δt) / Q_total + w

측정:       V_terminal = OCV(SOC) - I * R_internal + v

w = 프로세스 노이즈 (전류 센서 오차 등)
v = 측정 노이즈 (전압 센서 오차)
```

### 알고리즘 단계

```
1. 예측 (Predict)
   ┌────────────────────────────────┐
   │ x̂⁻ = A * x̂ + B * u            │  상태 예측
   │ P⁻ = A * P * Aᵀ + Q           │  공분산 예측
   └────────────────────────────────┘

2. 갱신 (Update)
   ┌────────────────────────────────┐
   │ K = P⁻ * Hᵀ / (H * P⁻ * Hᵀ + R)│  칼만 게인
   │ x̂ = x̂⁻ + K * (z - H * x̂⁻)     │  상태 갱신
   │ P = (I - K * H) * P⁻          │  공분산 갱신
   └────────────────────────────────┘
```

## 단순화된 구현

### 1차원 칼만 필터 (SOC만)

```c
// kalman_soc.c

typedef struct {
    // 상태
    float soc;              // 추정 SOC (0~1)
    float P;                // 오차 공분산
    
    // 파라미터
    float Q;                // 프로세스 노이즈
    float R;                // 측정 노이즈
    float capacity_ah;      // 배터리 용량
    float internal_r_ohm;   // 내부 저항
    
    // OCV 테이블
    const float *ocv_table;
    int ocv_table_size;
} kalman_soc_t;

static kalman_soc_t g_kalman;

void Kalman_Init(float initial_soc, float capacity_ah, 
                 float internal_r, const float *ocv_table, int table_size) {
    g_kalman.soc = initial_soc;
    g_kalman.P = 0.1f;           // 초기 불확실성
    g_kalman.Q = 0.0001f;        // 프로세스 노이즈 (작음)
    g_kalman.R = 0.01f;          // 측정 노이즈
    g_kalman.capacity_ah = capacity_ah;
    g_kalman.internal_r_ohm = internal_r;
    g_kalman.ocv_table = ocv_table;
    g_kalman.ocv_table_size = table_size;
}
```

### 예측 단계

```c
void Kalman_Predict(float current_a, float dt_s) {
    // 상태 전이: SOC(k+1) = SOC(k) - I*dt/Q
    float delta_soc = (current_a * dt_s) / (g_kalman.capacity_ah * 3600.0f);
    
    // 상태 예측
    g_kalman.soc -= delta_soc;
    
    // 범위 제한
    if (g_kalman.soc > 1.0f) g_kalman.soc = 1.0f;
    if (g_kalman.soc < 0.0f) g_kalman.soc = 0.0f;
    
    // 공분산 예측: P = P + Q
    g_kalman.P += g_kalman.Q;
}
```

### OCV 룩업

```c
// OCV 테이블에서 전압 조회 (선형 보간)
float Kalman_GetOCV(float soc) {
    // SOC를 인덱스로 변환
    float idx_f = soc * (g_kalman.ocv_table_size - 1);
    int idx_low = (int)idx_f;
    int idx_high = idx_low + 1;
    
    if (idx_high >= g_kalman.ocv_table_size) {
        return g_kalman.ocv_table[g_kalman.ocv_table_size - 1];
    }
    if (idx_low < 0) {
        return g_kalman.ocv_table[0];
    }
    
    // 선형 보간
    float frac = idx_f - idx_low;
    return g_kalman.ocv_table[idx_low] * (1.0f - frac) + 
           g_kalman.ocv_table[idx_high] * frac;
}

// OCV의 SOC에 대한 기울기 (dOCV/dSOC)
float Kalman_GetOCVSlope(float soc) {
    float delta = 0.01f;  // 1%
    float ocv_high = Kalman_GetOCV(soc + delta);
    float ocv_low = Kalman_GetOCV(soc - delta);
    
    return (ocv_high - ocv_low) / (2.0f * delta);
}
```

### 갱신 단계

```c
void Kalman_Update(float measured_voltage, float current_a) {
    // 예측 전압: V = OCV(SOC) - I * R
    float predicted_ocv = Kalman_GetOCV(g_kalman.soc);
    float predicted_voltage = predicted_ocv - current_a * g_kalman.internal_r_ohm;
    
    // 측정 오차 (Innovation)
    float y = measured_voltage - predicted_voltage;
    
    // 측정 행렬의 기울기: H = dV/dSOC = dOCV/dSOC
    float H = Kalman_GetOCVSlope(g_kalman.soc);
    
    // 칼만 게인: K = P * H / (H * P * H + R)
    float S = H * g_kalman.P * H + g_kalman.R;
    float K = g_kalman.P * H / S;
    
    // 상태 갱신: x = x + K * y
    g_kalman.soc += K * y;
    
    // 범위 제한
    if (g_kalman.soc > 1.0f) g_kalman.soc = 1.0f;
    if (g_kalman.soc < 0.0f) g_kalman.soc = 0.0f;
    
    // 공분산 갱신: P = (1 - K * H) * P
    g_kalman.P = (1.0f - K * H) * g_kalman.P;
}
```

### 메인 루프 통합

```c
// 100ms 주기로 호출
void Kalman_Process(float current_a, float voltage_v) {
    static uint32_t last_time = 0;
    uint32_t now = HAL_GetTick();
    float dt = (now - last_time) / 1000.0f;  // 초 단위
    last_time = now;
    
    // 예측 (쿨롱 카운팅)
    Kalman_Predict(current_a, dt);
    
    // 갱신 (전압 측정)
    // 휴식 상태이거나 전류가 작을 때만 갱신
    if (fabsf(current_a) < 1.0f) {
        Kalman_Update(voltage_v, current_a);
    }
    
    // SOC 출력 (0~100%)
    g_bms.soc = g_kalman.soc * 100.0f;
}

float Kalman_GetSOC(void) {
    return g_kalman.soc * 100.0f;  // %
}
```

## 확장 칼만 필터 (EKF)

### 2차원 상태 (SOC + 내부 저항)

```c
typedef struct {
    float x[2];     // [SOC, R_internal]
    float P[2][2];  // 2x2 공분산 행렬
    float Q[2][2];  // 프로세스 노이즈
    float R;        // 측정 노이즈
} ekf_soc_t;

void EKF_Predict(ekf_soc_t *ekf, float current_a, float dt_s) {
    // 상태 전이 (비선형)
    // SOC(k+1) = SOC(k) - I*dt/Q
    // R(k+1) = R(k) (저항은 변하지 않는다고 가정)
    
    ekf->x[0] -= (current_a * dt_s) / (CAPACITY_AH * 3600.0f);
    // ekf->x[1] 유지
    
    // 야코비안 A = ∂f/∂x
    float A[2][2] = {{1, 0}, {0, 1}};
    
    // P = A * P * Aᵀ + Q
    // (행렬 연산 생략)
}

void EKF_Update(ekf_soc_t *ekf, float voltage_v, float current_a) {
    // 예측 전압: V = OCV(SOC) - I * R
    float soc = ekf->x[0];
    float r = ekf->x[1];
    
    float ocv = GetOCV(soc);
    float v_pred = ocv - current_a * r;
    
    // 야코비안 H = ∂h/∂x = [dOCV/dSOC, -I]
    float H[2] = {GetOCVSlope(soc), -current_a};
    
    // 칼만 게인 계산...
    // 상태 갱신...
    // (생략)
}
```

## LFP용 OCV 테이블

```c
// LiFePO4 OCV-SOC 테이블 (25°C)
// 0%, 10%, 20%, ... 100%
const float LFP_OCV_TABLE[11] = {
    2.50f,   // 0%   - Empty
    3.00f,   // 10%
    3.20f,   // 20%
    3.25f,   // 30%
    3.27f,   // 40%
    3.28f,   // 50%  - 평탄 구간 시작
    3.29f,   // 60%
    3.30f,   // 70%
    3.32f,   // 80%  - 평탄 구간 끝
    3.40f,   // 90%
    3.65f,   // 100% - Full
};
```

## 파라미터 튜닝

### Q (프로세스 노이즈)

```c
// Q가 작으면: 쿨롱 카운팅 신뢰
// Q가 크면: OCV 측정 신뢰

// 전류 센서 오차 1% → Q ≈ 0.0001
g_kalman.Q = 0.0001f;
```

### R (측정 노이즈)

```c
// R이 작으면: 전압 측정 신뢰
// R이 크면: 예측 신뢰

// 전압 측정 오차 10mV, OCV 기울기 ~0.1V → R ≈ 0.01
g_kalman.R = 0.01f;
```

### 실험적 튜닝

```c
void Kalman_AutoTune(void) {
    // 수렴 속도 관찰
    // - 너무 빠르면: Q 감소 또는 R 증가
    // - 너무 느리면: Q 증가 또는 R 감소
    
    // 안정성 관찰
    // - 발산하면: Q 감소
    // - 노이즈가 심하면: R 증가
    
    // LFP 평탄 구간에서는 R 증가 (OCV 신뢰도 낮음)
    if (g_kalman.soc > 0.2f && g_kalman.soc < 0.8f) {
        g_kalman.R = 0.1f;  // 10배 증가
    } else {
        g_kalman.R = 0.01f;
    }
}
```

## 성능 비교

```c
// 테스트 결과 (시뮬레이션)

// 1. 쿨롱 카운팅만
// - 초기: 정확
// - 1시간 후: 오차 2%
// - 10시간 후: 오차 20%

// 2. OCV만 (휴식 시)
// - 정확도: ±5% (평탄 구간에서 ±15%)
// - 사용 불가: 부하 중

// 3. 칼만 필터
// - 초기: 정확
// - 10시간 후: 오차 2~3%
// - 평탄 구간: 오차 5% (쿨롱 카운팅으로 보완)
```

## 삽질: 수치 안정성

P가 음수가 되면 발산:

```c
// 잘못된 코드
g_kalman.P = (1.0f - K * H) * g_kalman.P;  // 부동소수점 오차 누적

// 안정적인 코드 (Joseph form)
float temp = 1.0f - K * H;
g_kalman.P = temp * g_kalman.P * temp + K * g_kalman.R * K;

// 최소값 보장
if (g_kalman.P < 0.0001f) g_kalman.P = 0.0001f;
```

## 삽질: OCV 갱신 조건

부하 중에는 OCV 측정 불가:

```c
// 잘못된 코드
Kalman_Update(voltage, current);  // 항상 갱신 → 오차 증가

// 올바른 코드
// 휴식 상태에서만 OCV로 보정
if (fabsf(current_a) < 0.5f && rest_time_ms > 30000) {
    Kalman_Update(voltage, current);
}

// 또는 부하 보정
float ocv_estimated = voltage + current * R_internal;
// 하지만 부하 중에는 정확도 낮음
```

## 정리

| 항목 | 쿨롱 카운팅 | OCV | 칼만 필터 |
|------|-------------|-----|-----------|
| 실시간 | ✅ | ❌ | ✅ |
| 누적 오차 | ❌ | ✅ | ✅ |
| 평탄 구간 | ✅ | ❌ | ✅ |
| 구현 복잡도 | 낮음 | 낮음 | 중간 |

**칼만 필터 핵심**:
- 예측: 쿨롱 카운팅
- 갱신: OCV 측정 (조건부)
- 결과: 두 방법의 장점 결합

**다음 글에서**: 프리차지 회로 - 돌입 전류 방지.

---

## 시리즈 네비게이션

**Part 7: 고급 기능편**
- [#21 - SOH 추정](/posts/bms/ad7280a-bms-dev-21/)
- **#22 - 칼만 필터 적용** ← 현재 글
- #23 - 프리차지 회로
- #24 - 절연 모니터링

---

## 참고 자료

- [Kalman Filter for SOC Estimation](https://ieeexplore.ieee.org/document/8469055)
- [Extended Kalman Filter Tutorial](https://www.kalmanfilter.net/default.aspx)
- [BMS Kalman Filter Implementation](https://github.com/battery-system/kalman-soc)
