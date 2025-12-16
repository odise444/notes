---
title: "AD7280A BMS 개발기 #23 - 프리차지 회로"
date: 2024-12-13
draft: false
tags: ["AD7280A", "BMS", "STM32", "프리차지", "돌입전류"]
categories: ["BMS 개발"]
series: ["AD7280A BMS 개발"]
summary: "인버터 연결하면 엄청난 돌입 전류가 흐른다. 프리차지로 방지한다."
---

72V 배터리를 인버터에 연결하면 인버터 내부 커패시터로 순간적으로 큰 전류가 흐른다. 수백 암페어.

이러면 커넥터가 녹거나 릴레이 접점이 손상된다. 프리차지(Pre-charge)로 천천히 충전해야 한다.

---

회로 구성:

```
배터리+ ──┬── [메인 릴레이] ──┬── 출력+
          │                   │
          └── [프리차지 릴레이] ─┤
                    │          │
                  [50Ω]        │
                    │          │
                    └──────────┘
```

1. 프리차지 릴레이 ON (50Ω 통해 충전)
2. 출력 전압이 배터리 전압의 90%가 될 때까지 대기
3. 메인 릴레이 ON
4. 프리차지 릴레이 OFF

---

```c
typedef enum {
    PRECHARGE_IDLE,
    PRECHARGE_ACTIVE,
    PRECHARGE_COMPLETE,
    PRECHARGE_FAIL,
} PrechargeState_t;

void Precharge_Update(void) {
    switch (precharge_state) {
    case PRECHARGE_IDLE:
        // 시작 명령 대기
        break;
        
    case PRECHARGE_ACTIVE:
        // 프리차지 릴레이 ON
        HAL_GPIO_WritePin(PRECHARGE_RELAY, GPIO_PIN_SET);
        
        // 타임아웃 체크 (5초)
        if (HAL_GetTick() - precharge_start > 5000) {
            precharge_state = PRECHARGE_FAIL;
            break;
        }
        
        // 전압 체크 (90% 도달?)
        if (output_voltage > pack_voltage * 0.9f) {
            precharge_state = PRECHARGE_COMPLETE;
        }
        break;
        
    case PRECHARGE_COMPLETE:
        // 메인 릴레이 ON
        HAL_GPIO_WritePin(MAIN_RELAY, GPIO_PIN_SET);
        HAL_Delay(50);
        // 프리차지 릴레이 OFF
        HAL_GPIO_WritePin(PRECHARGE_RELAY, GPIO_PIN_RESET);
        precharge_state = PRECHARGE_IDLE;
        break;
        
    case PRECHARGE_FAIL:
        // 프리차지 릴레이 OFF
        HAL_GPIO_WritePin(PRECHARGE_RELAY, GPIO_PIN_RESET);
        // 에러 처리
        SetFault(FAULT_PRECHARGE);
        break;
    }
}
```

---

프리차지 저항 선정:

```
시간 상수 τ = R × C
90% 충전 시간 ≈ 2.3 × τ

인버터 커패시터 C = 1000µF라면
50Ω × 1000µF = 50ms
2.3 × 50ms = 115ms

실제로는 마진 줘서 1~2초 대기
```

저항이 너무 작으면 돌입 전류가 크고, 너무 크면 충전 시간이 오래 걸린다. 50~100Ω이 적당.

---

다음 글에서 절연 모니터링을 다룬다.

[#24 - 절연 모니터링](/posts/bms/ad7280a-bms-dev-24/)
