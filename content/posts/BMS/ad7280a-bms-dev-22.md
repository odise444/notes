---
title: "AD7280A BMS 개발기 #22 - 칼만 필터로 SOC 정밀화"
date: 2024-02-05
draft: false
tags: ["AD7280A", "BMS", "STM32", "칼만필터", "SOC"]
categories: ["BMS 개발"]
series: ["AD7280A BMS 개발"]
summary: "쿨롱 카운팅은 오차가 쌓인다. 칼만 필터로 보정해보자."
---

쿨롱 카운팅은 시간 지나면 오차가 누적된다. 전류 센서 오차, 샘플링 오차.

칼만 필터로 OCV 측정값이랑 융합하면 정확도가 올라간다.

---

## 칼만 필터 기초

예측 → 보정 반복.

```
예측: 이전 상태 + 입력으로 다음 상태 예측
보정: 측정값으로 예측 보정
```

SOC 추정에 적용:
- 예측: 쿨롱 카운팅 (전류 적분)
- 보정: OCV 측정

---

## 단순화된 구현

EKF(Extended Kalman Filter)가 정석이지만 복잡하다.

단순화 버전:

```c
typedef struct {
    float soc;          // 상태 (SOC)
    float P;            // 오차 공분산
    float Q;            // 프로세스 노이즈
    float R;            // 측정 노이즈
} KalmanFilter_t;

KalmanFilter_t kf = {
    .soc = 50.0f,
    .P = 1.0f,
    .Q = 0.001f,   // 쿨롱 카운팅 신뢰도
    .R = 10.0f     // OCV 측정 신뢰도
};
```

---

## 예측 단계

쿨롱 카운팅으로 예측:

```c
void KF_Predict(float current_a, float dt) {
    // 상태 예측: SOC -= I * dt / Capacity
    float dsoc = current_a * dt / (g_bms.capacity_ah * 3600) * 100;
    kf.soc -= dsoc;
    
    // 오차 공분산 증가
    kf.P += kf.Q;
    
    // 범위 제한
    if (kf.soc > 100) kf.soc = 100;
    if (kf.soc < 0) kf.soc = 0;
}
```

---

## 보정 단계

OCV 측정값으로 보정:

```c
void KF_Update(float measured_soc) {
    // 칼만 게인
    float K = kf.P / (kf.P + kf.R);
    
    // 상태 보정
    kf.soc += K * (measured_soc - kf.soc);
    
    // 오차 공분산 감소
    kf.P *= (1 - K);
}
```

---

## 전체 흐름

```c
void SOC_KalmanUpdate(void) {
    float current_a = g_bms.pack_current_ma / 1000.0f;
    float dt = 0.1f;  // 100ms
    
    // 예측 (항상)
    KF_Predict(current_a, dt);
    
    // 보정 (휴지 시에만)
    if (fabsf(current_a) < 0.5f && g_bms.rest_time > 60) {
        float soc_ocv = OCV_LookupSOC(g_bms.avg_cell_mv);
        KF_Update(soc_ocv);
    }
    
    g_bms.soc = (uint8_t)kf.soc;
}
```

---

## 파라미터 튜닝

Q와 R 비율이 중요하다.

```
Q 크면: 쿨롱 카운팅 안 믿음, OCV에 빨리 수렴
R 크면: OCV 안 믿음, 쿨롱 카운팅 위주
```

LFP처럼 OCV가 평평하면 R을 크게:

```c
// NMC
kf.Q = 0.001f;
kf.R = 5.0f;

// LFP  
kf.Q = 0.0001f;  
kf.R = 50.0f;    // OCV 신뢰도 낮춤
```

---

## 테스트 결과

![칼만 필터 결과](/imgs/ad7280a-bms-dev-22-1.png)

```
단순 쿨롱 카운팅: 오차 5~10%
칼만 필터 적용: 오차 2~3%
```

특히 장시간 사용 후 오차 누적이 줄어든다.

---

## 정리

- 예측: 쿨롱 카운팅
- 보정: OCV 측정 (휴지 시)
- Q/R 비율로 각 측정 신뢰도 조절
- LFP는 OCV 신뢰도 낮게

구현은 단순한데 효과는 좋다.

---

다음은 프리차지 회로.

[#23 - 프리차지](/posts/bms/ad7280a-bms-dev-23/)
