---
title: "AD7280A BMS 개발기 #23 - 프리차지, 돌입 전류 막기"
date: 2024-02-06
draft: false
tags: ["AD7280A", "BMS", "STM32", "프리차지", "돌입전류"]
categories: ["BMS 개발"]
series: ["AD7280A BMS 개발"]
summary: "배터리 연결하면 '펑!' 인버터 입력 캡에 돌입 전류. 프리차지로 막자."
---

배터리를 인버터에 처음 연결할 때 문제가 생겼다.

릴레이 붙이는 순간 스파크가 튀고 릴레이 접점이 녹았다.

---

## 돌입 전류

인버터 입력단에 큰 커패시터가 있다. 보통 수천 uF.

배터리 연결 순간 커패시터가 빈 상태니까 순간적으로 큰 전류가 흐른다.

```
I = C × dV/dt

C = 2000uF, V = 72V, dt = 1ms라면
I = 2000e-6 × 72 / 0.001 = 144A
```

144A가 순간적으로 흐른다. 릴레이 정격 초과.

---

## 프리차지 원리

저항을 통해 먼저 충전한 다음 메인 릴레이를 붙인다.

```
Battery+ ──[프리차지 릴레이]──[저항]──┬── Load+
              │                      │
           [메인 릴레이]─────────────┘
```

시퀀스:
1. 프리차지 릴레이 ON
2. 저항 통해 커패시터 충전 (느리게)
3. 전압이 90% 이상 도달하면
4. 메인 릴레이 ON
5. 프리차지 릴레이 OFF

---

## 저항 선정

```
충전 시정수: τ = R × C
2000uF, 충전 시간 500ms 원하면:
R = τ / C = 0.5 / 2000e-6 = 250Ω

전력:
P = V² / R = 72² / 250 = 20W (피크)
```

250Ω, 25W 저항 사용. 또는 여러 개 직렬.

---

## 상태 머신

```c
typedef enum {
    PC_IDLE,
    PC_PRECHARGE,
    PC_WAIT,
    PC_MAIN_ON,
    PC_COMPLETE,
    PC_FAULT
} PrechargeState_t;

void Precharge_Task(void) {
    static uint32_t start_time = 0;
    
    switch (g_precharge.state) {
        case PC_IDLE:
            // 시작 조건 대기
            break;
            
        case PC_PRECHARGE:
            RELAY_Precharge(ON);
            RELAY_Main(OFF);
            start_time = HAL_GetTick();
            g_precharge.state = PC_WAIT;
            break;
            
        case PC_WAIT:
            // 전압 모니터링
            if (g_bms.load_voltage > g_bms.pack_voltage * 0.9f) {
                g_precharge.state = PC_MAIN_ON;
            }
            // 타임아웃
            if (HAL_GetTick() - start_time > 2000) {
                g_precharge.state = PC_FAULT;
            }
            break;
            
        case PC_MAIN_ON:
            RELAY_Main(ON);
            HAL_Delay(50);  // 릴레이 안정화
            RELAY_Precharge(OFF);
            g_precharge.state = PC_COMPLETE;
            break;
            
        case PC_COMPLETE:
            // 정상 동작
            break;
            
        case PC_FAULT:
            RELAY_Precharge(OFF);
            RELAY_Main(OFF);
            BMS_SetFault(FAULT_PRECHARGE);
            break;
    }
}
```

---

## 부하 전압 측정

프리차지 완료 판단을 위해 부하측 전압 측정 필요.

```c
// 분압 회로로 ADC 측정
// 72V → 3.3V: 100k / 4.7k 분압

uint16_t ADC_ReadLoadVoltage(void) {
    uint16_t raw = HAL_ADC_GetValue(&hadc1);
    
    // raw → mV
    float v_adc = raw * 3300.0f / 4096;
    float v_load = v_adc * (100 + 4.7) / 4.7;
    
    return (uint16_t)v_load;
}
```

---

## 타임아웃 처리

프리차지가 안 되는 경우:
- 부하측 단락
- 저항 단선
- 릴레이 고장

2초 타임아웃 걸어서 폴트 처리.

---

## 정리

- 프리차지: 저항으로 천천히 충전
- 90% 도달 후 메인 릴레이
- 타임아웃으로 이상 감지
- 부하 전압 측정 필요

스파크 없이 부드럽게 연결된다.

---

다음은 절연 모니터링.

[#24 - 절연 모니터링](/posts/bms/ad7280a-bms-dev-24/)
