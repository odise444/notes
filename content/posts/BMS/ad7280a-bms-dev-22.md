---
title: "AD7280A BMS 개발기 #22 - 칼만 필터"
date: 2024-12-13
draft: false
tags: ["AD7280A", "BMS", "STM32", "칼만필터", "SOC"]
categories: ["BMS 개발"]
series: ["AD7280A BMS 개발"]
summary: "쿨롱 카운팅 오차를 줄이는 칼만 필터. 수식이 좀 복잡하다."
---

쿨롱 카운팅은 오차가 누적된다. 칼만 필터로 이걸 줄일 수 있다.

칼만 필터는 예측값과 측정값을 적절히 섞어서 추정치를 개선하는 알고리즘이다.

---

1D 칼만 필터 (SOC 추정용):

![칼만 필터 구조](/imgs/ad7280a-bms-dev-22-1.png)

```c
typedef struct {
    float x;    // 상태 (SOC)
    float p;    // 오차 공분산
    float q;    // 프로세스 노이즈
    float r;    // 측정 노이즈
} KalmanFilter_t;

void Kalman_Init(KalmanFilter_t *kf) {
    kf->x = 50.0f;   // 초기 SOC
    kf->p = 10.0f;   // 초기 불확실성
    kf->q = 0.01f;   // 프로세스 노이즈 (작을수록 예측 신뢰)
    kf->r = 1.0f;    // 측정 노이즈 (작을수록 측정 신뢰)
}

float Kalman_Update(KalmanFilter_t *kf, float soc_coulomb, float soc_ocv) {
    // 예측 단계
    // x_pred = x (SOC는 자체적으로 안 변함)
    // p_pred = p + q
    kf->p = kf->p + kf->q;
    
    // 측정 단계
    float k = kf->p / (kf->p + kf->r);  // 칼만 게인
    kf->x = kf->x + k * (soc_ocv - kf->x);  // 상태 업데이트
    kf->p = (1 - k) * kf->p;  // 공분산 업데이트
    
    return kf->x;
}
```

---

Q, R 튜닝이 중요하다.

- Q 크면: 예측 불신, 측정에 민감
- R 크면: 측정 불신, 예측에 의존

LFP는 OCV 커브가 평평해서 R을 크게 해야 한다.

```c
// LFP용 튜닝
kf->q = 0.001f;  // 쿨롱 카운팅 신뢰
kf->r = 10.0f;   // OCV 불신 (평평해서)
```

---

칼만 필터 쓰니까 SOC 정확도가 ±5%에서 ±2%로 좋아졌다.

---

다음 글에서 프리차지 회로를 다룬다.

[#23 - 프리차지 회로](/posts/bms/ad7280a-bms-dev-23/)
