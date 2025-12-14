---
title: "AD7280A BMS 개발 삽질기 #22 - 칼만 필터 적용"
date: 2024-12-13
draft: false
tags: ["AD7280A", "BMS", "STM32", "칼만필터", "SOC", "센서퓨전"]
categories: ["BMS 개발"]
series: ["AD7280A BMS 개발"]
summary: "쿨롱 카운팅의 누적 오차를 잡아라! 칼만 필터로 SOC 추정 정밀도를 높인다."
---

## 지난 글 요약

[지난 글](/posts/bms/ad7280a-bms-dev-21/)에서 SOH 추정을 다뤘다. 용량과 저항 기반. 이제 **SOC 정밀도**를 높이는 칼만 필터를 적용하자.

## 왜 칼만 필터인가?

### 기존 방식의 문제

```
쿨롱 카운팅:
- 실시간 추적 ✅
- 누적 오차 ❌ (전류 센서 오프셋, 노이즈)

OCV 테이블:
- 절대값 정확 ✅
- 휴식 필요 ❌
- LFP 평탄 구간 ❌
```

### 칼만 필터의 역할

```
┌───────────────────────────────────────────────────────┐
│                   Kalman Filter                       │
│                                                       │
│  ┌──────────────┐                                     │
│  │ 쿨롱 카운팅   │──────┐                              │
│  │ (Prediction) │      │    ┌──────────────┐          │
│  └──────────────┘      ├───►│   Optimal    │───► SOC  │
│                        │    │   Estimate   │          │
│  ┌──────────────┐      │    └──────────────┘          │
│  │  OCV 측정    │──────┘                              │
│  │ (Correction) │                                     │
│  └──────────────┘                                     │
│                                                       │
└───────────────────────────────────────────────────────┘
```

**핵심**: 두 정보의 **불확실성**을 고려해서 최적 결합!

## 칼만 필터 기초

### 1차원 칼만 필터 개념

```
상태: x (SOC)
측정: z (OCV로부터 추정한 SOC)
입력: u (전류)

예측 단계 (Predict):
  x̂⁻ = x̂ + (I × Δt / Q)     // 쿨롱 카운팅
  P⁻ = P + Q_process         // 불확실성 증가

보정 단계 (Update):
  K = P⁻ / (P⁻ + R_measure)  // 칼만 이득
  x̂ = x̂⁻ + K × (z - x̂⁻)    // 최적 추정
  P = (1 - K) × P⁻           // 불확실성 감소
```

### 변수 설명

| 변수 | 의미 | 예시 |
|------|------|------|
| x̂ | SOC 추정값 | 75% |
| P | 추정 불확실성 (분산) | 0.01 |
| Q | 프로세스 노이즈 | 0.001 |
| R | 측정 노이즈 | 0.1 |
| K | 칼만 이득 | 0~1 |

## 구현

### 데이터 구조

```c
// kalman_soc.c

typedef struct {
    // 상태
    float soc;              // SOC 추정값 (0~1)
    float P;                // 추정 분산
    
    // 파라미터
    float Q_process;        // 프로세스 노이즈 분산
    float R_measure;        // 측정 노이즈 분산
    float capacity_ah;      // 배터리 용량
    
    // OCV 관련
    float ocv_soc;          // OCV로 추정한 SOC
    bool ocv_valid;         // OCV 측정 유효
    
    // 디버그
    float K;                // 마지막 칼만 이득
} kalman_soc_t;

static kalman_soc_t g_kalman;
```

### 초기화

```c
void Kalman_SOC_Init(float initial_soc, float capacity_ah) {
    g_kalman.soc = initial_soc;
    g_kalman.P = 0.1f;          // 초기 불확실성 (높게)
    
    g_kalman.Q_process = 0.0001f;  // 프로세스 노이즈 (작게)
    g_kalman.R_measure = 0.05f;    // 측정 노이즈 (중간)
    
    g_kalman.capacity_ah = capacity_ah;
    g_kalman.ocv_valid = false;
}
```

### 예측 단계 (Predict)

```c
void Kalman_SOC_Predict(float current_a, float dt_hours) {
    // 1. 상태 예측 (쿨롱 카운팅)
    // 방전: + 전류, 충전: - 전류
    float delta_soc = (current_a * dt_hours) / g_kalman.capacity_ah;
    g_kalman.soc -= delta_soc;
    
    // 범위 제한
    if (g_kalman.soc > 1.0f) g_kalman.soc = 1.0f;
    if (g_kalman.soc < 0.0f) g_kalman.soc = 0.0f;
    
    // 2. 불확실성 예측
    // 전류가 클수록 노이즈 증가
    float current_factor = fabsf(current_a) / 10.0f;  // 10A 기준
    float Q_adjusted = g_kalman.Q_process * (1.0f + current_factor);
    
    g_kalman.P += Q_adjusted;
}
```

### 보정 단계 (Update)

```c
void Kalman_SOC_Update(float measured_soc) {
    // OCV로부터 측정한 SOC
    g_kalman.ocv_soc = measured_soc;
    g_kalman.ocv_valid = true;
    
    // 1. 칼만 이득 계산
    g_kalman.K = g_kalman.P / (g_kalman.P + g_kalman.R_measure);
    
    // 2. 상태 업데이트
    float innovation = measured_soc - g_kalman.soc;
    g_kalman.soc += g_kalman.K * innovation;
    
    // 범위 제한
    if (g_kalman.soc > 1.0f) g_kalman.soc = 1.0f;
    if (g_kalman.soc < 0.0f) g_kalman.soc = 0.0f;
    
    // 3. 불확실성 업데이트
    g_kalman.P = (1.0f - g_kalman.K) * g_kalman.P;
}
```

### 메인 루프 통합

```c
void BMS_MainLoop_100ms(void) {
    static uint32_t rest_counter = 0;
    
    float current_a = BMS_GetPackCurrent();
    float cell_voltage = BMS_GetMinCellVoltage() / 1000.0f;  // V
    
    // 1. 예측 (항상)
    Kalman_SOC_Predict(current_a, 100.0f / 3600000.0f);  // 100ms
    
    // 2. 휴식 상태 감지
    if (fabsf(current_a) < 0.5f) {
        rest_counter++;
    } else {
        rest_counter = 0;
    }
    
    // 3. 충분한 휴식 후 OCV 보정
    if (rest_counter >= 300) {  // 30초 휴식
        float ocv_soc = SOC_OCV_GetSOC(cell_voltage);
        Kalman_SOC_Update(ocv_soc);
        
        rest_counter = 0;  // 리셋 (너무 자주 보정 방지)
    }
    
    // 4. 결과 사용
    g_bms.soc = g_kalman.soc * 100.0f;  // 0~100%
}
```

## 노이즈 파라미터 튜닝

### Q (프로세스 노이즈)

```c
// Q가 크면: 측정값 신뢰 ↑, 쿨롱 카운팅 신뢰 ↓
// Q가 작으면: 쿨롱 카운팅 신뢰 ↑, 측정값 신뢰 ↓

// 적응형 Q: 전류 크기에 따라 조절
float Kalman_GetAdaptiveQ(float current_a) {
    float base_Q = 0.0001f;
    
    // 대전류 시 쿨롱 카운팅 오차 증가
    if (fabsf(current_a) > 20.0f) {
        return base_Q * 2.0f;
    } else if (fabsf(current_a) > 10.0f) {
        return base_Q * 1.5f;
    }
    
    return base_Q;
}
```

### R (측정 노이즈)

```c
// R이 크면: 측정값 신뢰 ↓
// R이 작으면: 측정값 신뢰 ↑

// LFP 특성: SOC에 따라 OCV 신뢰도 다름
float Kalman_GetAdaptiveR(float soc) {
    // LFP 평탄 구간 (20~80%): OCV 불확실성 높음
    if (soc > 0.2f && soc < 0.8f) {
        return 0.1f;   // 낮은 신뢰
    }
    // 양 끝 구간: OCV 확실
    return 0.01f;      // 높은 신뢰
}
```

## 확장 칼만 필터 (EKF)

비선형 시스템용 (더 정확):

```c
// 상태 벡터: [SOC, R_internal]
// 비선형 측정 방정식: V = OCV(SOC) - I × R

typedef struct {
    float x[2];         // [SOC, R_internal]
    float P[2][2];      // 공분산 행렬
    float Q[2][2];      // 프로세스 노이즈
    float R;            // 측정 노이즈
} ekf_soc_t;

void EKF_Predict(ekf_soc_t *ekf, float current_a, float dt) {
    // 상태 전이
    ekf->x[0] -= (current_a * dt) / CAPACITY_AH;  // SOC
    // R_internal은 변화 없음 (느린 변화 가정)
    
    // 공분산 전파 (자코비안 사용)
    // 간략화: P = F * P * F' + Q
    ekf->P[0][0] += ekf->Q[0][0];
    ekf->P[1][1] += ekf->Q[1][1];
}

void EKF_Update(ekf_soc_t *ekf, float voltage, float current) {
    // 예측 전압
    float ocv = OCV_Lookup(ekf->x[0]);
    float v_pred = ocv - current * ekf->x[1];
    
    // 혁신 (Innovation)
    float y = voltage - v_pred;
    
    // 측정 자코비안: H = [dV/dSOC, dV/dR] = [dOCV/dSOC, -I]
    float dOCV_dSOC = OCV_Derivative(ekf->x[0]);
    float H[2] = {dOCV_dSOC, -current};
    
    // 칼만 이득 계산 (2×1)
    float S = H[0] * ekf->P[0][0] * H[0] + 
              H[1] * ekf->P[1][1] * H[1] + ekf->R;
    
    float K[2];
    K[0] = ekf->P[0][0] * H[0] / S;
    K[1] = ekf->P[1][1] * H[1] / S;
    
    // 상태 업데이트
    ekf->x[0] += K[0] * y;
    ekf->x[1] += K[1] * y;
    
    // 공분산 업데이트
    ekf->P[0][0] *= (1.0f - K[0] * H[0]);
    ekf->P[1][1] *= (1.0f - K[1] * H[1]);
}
```

## 엔드포인트 강제 보정

칼만 필터도 엔드포인트에서 리셋:

```c
void Kalman_EndpointCorrection(float voltage, float current) {
    // 충전 완료 (CV 종료 전류)
    if (voltage >= 3.60f && current < -0.1f && current > -0.5f) {
        g_kalman.soc = 1.0f;      // 100%
        g_kalman.P = 0.001f;       // 높은 확신
    }
    
    // 방전 완료 (저전압 컷오프)
    if (voltage <= 2.50f) {
        g_kalman.soc = 0.0f;      // 0%
        g_kalman.P = 0.001f;       // 높은 확신
    }
}
```

## 칼만 필터 효과

```
시뮬레이션 결과:
────────────────────────────────────────
시간 경과 →

실제 SOC:     ████████████████████████─────
쿨롱만:       ████████████████████─────────  (누적 오차)
OCV만:        ██████████████████████████████ (노이즈)
칼만:         ████████████████████████───── (최적)

최대 오차:
- 쿨롱만: ±5%
- OCV만: ±10% (평탄 구간)
- 칼만: ±2%
```

## 삽질: 초기값 문제

초기 SOC를 모르면 수렴까지 오래 걸림:

```c
// 잘못된 초기화
Kalman_SOC_Init(0.5f, 100.0f);  // 임의의 50%

// 올바른 초기화: OCV로 초기값 설정
void Kalman_SOC_InitFromOCV(float cell_voltage, float capacity_ah) {
    float initial_soc = SOC_OCV_GetSOC(cell_voltage);
    Kalman_SOC_Init(initial_soc, capacity_ah);
    
    g_kalman.P = 0.05f;  // OCV 측정의 불확실성
}
```

## 삽질: Q/R 비율

Q/R 비율이 중요:

```c
// Q >> R: 측정값만 따라감 (노이즈 심함)
// Q << R: 예측값만 따라감 (측정 무시)

// 적절한 비율 찾기: 실험적 튜닝 필요!
// 시작점: Q/R ≈ 0.001 ~ 0.01
```

## 정리

| 항목 | 값/설명 |
|------|---------|
| 알고리즘 | 1D 칼만 필터 (또는 EKF) |
| 상태 | SOC (0~1) |
| 예측 | 쿨롱 카운팅 |
| 보정 | OCV 테이블 (휴식 시) |
| Q 기본값 | 0.0001 |
| R 기본값 | 0.05 (LFP 평탄 구간) |
| 개선 효과 | 오차 ±5% → ±2% |

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

- [Kalman Filter Tutorial](https://www.kalmanfilter.net/)
- [Battery SOC Estimation with Kalman Filter](https://www.mdpi.com/1996-1073/12/8/1447)
- [Extended Kalman Filter for BMS](https://ieeexplore.ieee.org/document/8400534)
