---
title: "AD7280A BMS 개발기 #18 - SOC 추정"
date: 2024-12-13
draft: false
tags: ["AD7280A", "BMS", "STM32", "SOC", "쿨롱카운팅"]
categories: ["BMS 개발"]
series: ["AD7280A BMS 개발"]
summary: "배터리 잔량 추정. 쿨롱 카운팅이랑 OCV 테이블을 섞어서 쓴다."
---

SOC(State of Charge)는 배터리 잔량이다. 이걸 정확히 아는 게 생각보다 어렵다.

---

쿨롱 카운팅이 기본이다. 전류를 적분하면 된다.

```c
// dt마다 호출
void SOC_CoulombCounting(float current_a, float dt_sec) {
    // Ah = A × sec / 3600
    float delta_ah = current_a * dt_sec / 3600.0f;
    
    accumulated_ah += delta_ah;
    soc = (capacity_ah - accumulated_ah) / capacity_ah * 100.0f;
}
```

문제는 오차가 누적된다는 거다. 센서 오차 1%가 하루 지나면 몇 %가 된다.

---

OCV(Open Circuit Voltage) 테이블로 보정한다. 전류가 안 흐를 때 전압으로 SOC를 추정할 수 있다.

![OCV 커브](/imgs/ad7280a-bms-dev-18-2.png)

```c
// LFP OCV 테이블 (전압 → SOC)
const uint8_t ocv_to_soc[] = {
    // 2.50V = 0%, 2.80V = 5%, ... , 3.65V = 100%
    0, 5, 8, 10, 12, 14, 16, 18, 20, 22,  // 2.5~2.95V
    25, 28, 32, 36, 40, 45, 50, 55, 60, 65, // 2.95~3.20V
    // ... 생략
};

uint8_t OCV_LookupSOC(uint16_t cell_mv) {
    if (cell_mv < 2500) return 0;
    if (cell_mv > 3650) return 100;
    
    int idx = (cell_mv - 2500) / 50;  // 50mV 단위
    return ocv_to_soc[idx];
}
```

---

하이브리드 방식: 평소에는 쿨롱 카운팅, 쉴 때 OCV로 보정.

```c
void SOC_Update(float current_a, float avg_cell_mv) {
    static uint32_t rest_start = 0;
    
    // 쿨롱 카운팅은 항상
    SOC_CoulombCounting(current_a, 0.1f);
    
    // 전류가 거의 없으면 (휴지 상태)
    if (fabsf(current_a) < 0.5f) {
        if (rest_start == 0) rest_start = HAL_GetTick();
        
        // 10분 이상 쉬면 OCV 보정
        if (HAL_GetTick() - rest_start > 600000) {
            uint8_t ocv_soc = OCV_LookupSOC(avg_cell_mv);
            soc = soc * 0.7f + ocv_soc * 0.3f;  // 블렌딩
            rest_start = 0;
        }
    } else {
        rest_start = 0;
    }
}
```

---

LFP는 OCV 커브가 평평해서 중간 구간에서 OCV 보정이 잘 안 먹는다. 이건 #B4 번외편에서 다룬다.

---

다음 글에서 데이터 로깅을 다룬다.

[#19 - 데이터 로깅](/posts/bms/ad7280a-bms-dev-19/)
