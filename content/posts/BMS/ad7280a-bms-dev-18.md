---
title: "AD7280A BMS 개발기 #18 - SOC 추정, 배터리 몇 % 남았나"
date: 2024-02-01
draft: false
tags: ["AD7280A", "BMS", "STM32", "SOC", "쿨롱카운팅"]
categories: ["BMS 개발"]
series: ["AD7280A BMS 개발"]
summary: "배터리 잔량 표시가 필요하다. 생각보다 어렵다."
---

"배터리 몇 % 남았어요?"

간단한 질문 같은데 답하기가 어렵다.

---

## SOC란

State of Charge. 충전 상태. 0%~100%.

```
SOC = 현재 잔량 / 최대 용량 × 100
```

문제는 "현재 잔량"을 직접 측정할 방법이 없다는 거다.

---

## 방법 1: OCV 기반

Open Circuit Voltage. 무부하 전압으로 SOC 추정.

배터리는 SOC에 따라 전압이 다르다. 테이블 만들어놓고 역으로 찾으면 된다.

![OCV 커브](/imgs/ad7280a-bms-dev-18-2.png)

```c
// OCV → SOC 룩업 테이블
const uint16_t ocv_table[101] = {
    // SOC 0% ~ 100%
    2500, 2850, 2920, 2960, 2990, ...  // mV
};

uint8_t OCV_LookupSOC(uint16_t voltage_mv) {
    for (int soc = 100; soc >= 0; soc--) {
        if (voltage_mv >= ocv_table[soc]) {
            return soc;
        }
    }
    return 0;
}
```

문제: 전류 흐르면 전압이 달라진다. 무부하 상태에서만 정확.

---

## 방법 2: 쿨롱 카운팅

전류 적분해서 용량 계산.

```
Q = ∫ I dt
SOC = SOC_init - Q / Capacity
```

전류 센서로 전류 측정하고, 시간으로 적분:

```c
void SOC_CoulombCounting(void) {
    static uint32_t last_time = 0;
    uint32_t now = HAL_GetTick();
    float dt = (now - last_time) / 1000.0f;  // 초
    last_time = now;
    
    float current_a = g_bms.pack_current_ma / 1000.0f;
    float dq_ah = current_a * dt / 3600.0f;  // Ah
    
    g_bms.soc_coulomb -= dq_ah / g_bms.capacity_ah * 100.0f;
    
    // 범위 제한
    if (g_bms.soc_coulomb > 100) g_bms.soc_coulomb = 100;
    if (g_bms.soc_coulomb < 0) g_bms.soc_coulomb = 0;
}
```

문제: 오차가 누적된다. 전류 센서 오차, 샘플링 오차가 쌓임.

---

## 방법 3: 하이브리드

둘을 섞는다.

- 평소: 쿨롱 카운팅
- 휴지 시: OCV로 보정

```c
void SOC_UpdateHybrid(void) {
    // 쿨롱 카운팅
    SOC_CoulombCounting();
    
    // 휴지 상태면 OCV 보정
    if (fabsf(g_bms.pack_current_ma) < 500) {  // 0.5A 미만
        g_bms.rest_time++;
        
        if (g_bms.rest_time > 60) {  // 1분 이상 휴지
            uint8_t soc_ocv = OCV_LookupSOC(g_bms.avg_cell_mv);
            
            // 가중 평균
            g_bms.soc = g_bms.soc_coulomb * 0.8f + soc_ocv * 0.2f;
        }
    } else {
        g_bms.rest_time = 0;
    }
}
```

---

## LFP 문제

LFP는 OCV 커브가 평평하다.

![LFP OCV](/imgs/ad7280a-bms-dev-18-3.png)

20%~80% 구간에서 전압이 거의 같다. OCV로 SOC 추정이 안 된다.

```
3.28V → SOC 30%? 50%? 70%?
```

LFP는 쿨롱 카운팅에 더 의존해야 한다.

---

## 완충/완방 리셋

누적 오차 해결책: 끝점에서 리셋.

```c
void SOC_CheckEndpoints(void) {
    // 완충 감지
    if (g_bms.max_cell_mv > 3600 && 
        fabsf(g_bms.pack_current_ma) < 200) {
        g_bms.soc = 100;
        g_bms.soc_coulomb = 100;
    }
    
    // 완방 감지
    if (g_bms.min_cell_mv < 2600) {
        g_bms.soc = 0;
        g_bms.soc_coulomb = 0;
    }
}
```

완충/완방 될 때마다 오차 리셋.

---

## 정리

- OCV: 무부하에서 정확, 부하 시 부정확
- 쿨롱 카운팅: 연속 추정, 오차 누적
- 하이브리드: 둘 조합
- LFP: OCV 안 먹힘, 쿨롱 카운팅 의존
- 끝점 리셋으로 오차 보정

---

다음은 데이터 로깅.

[#19 - 데이터 로깅](/posts/bms/ad7280a-bms-dev-19/)
