---
title: "AD7280A BMS 개발 삽질기 #24 - 절연 모니터링"
date: 2024-12-13
draft: false
tags: ["AD7280A", "BMS", "STM32", "절연", "안전", "IMD"]
categories: ["BMS 개발"]
series: ["AD7280A BMS 개발"]
summary: "고전압이 섀시로 새면 감전! 절연 모니터링으로 누설 전류를 감지하고 안전을 확보한다."
---

## 지난 글 요약

[지난 글](/posts/bms/ad7280a-bms-dev-23/)에서 프리차지 회로를 구현했다. 돌입 전류 방지. 이제 고전압 시스템의 **절연 안전**을 다루자.

## 왜 절연 모니터링인가?

### 고전압 시스템의 위험

```
정상 상태:
┌─────────────────────────────────────┐
│      Battery Pack (72V)            │
│  ┌─────┐  ┌─────┐  ┌─────┐        │
│  │Cell │──│Cell │──│Cell │──...   │ 절연
│  └─────┘  └─────┘  └─────┘        │
└─────────────────────────────────────┘
        ╱╱╱ (절연) ╱╱╱
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
         차체 (Chassis GND)


절연 파괴 시:
┌─────────────────────────────────────┐
│      Battery Pack (72V)            │
│  ┌─────┐  ┌─────┐  ┌─────┐        │
│  │Cell │──│Cell │──│Cell │──...   │
│  └─────┘  └──┬──┘  └─────┘        │
└──────────────│──────────────────────┘
               │ ← 절연 파괴!
               ▼   누설 전류
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
         차체 (Chassis GND)
              │
              ▼
            감전 위험!
```

### 절연 요구사항

| 규격 | 요구 저항 | 용도 |
|------|----------|------|
| UN ECE R100 | >100Ω/V | 전기차 |
| IEC 61851 | >100Ω/V | 충전기 |
| 일반 | >500Ω/V | 산업용 |

**72V 시스템**: 최소 7.2kΩ (100Ω/V 기준)

## 절연 측정 원리

### 방법 1: 저항 분압법

```
        ┌─── Vmid ───┐
        │            │
    ────┼────────────┼────
        │            │
   ┌────┴────┐  ┌────┴────┐
   │   R1    │  │   R2    │
   │  1MΩ    │  │  1MΩ    │
   └────┬────┘  └────┬────┘
        │            │
   Pack+│            │Pack-
        │            │
        └──── Riso ──┘
               │
               ▼
          Chassis GND
```

**원리**:
- R1, R2: 알려진 저항
- Vmid 측정으로 Riso 계산
- Riso가 낮으면 절연 불량

### 측정 회로

```c
// isolation_monitor.h

typedef struct {
    float v_pack_plus;      // Pack+ 전압 (vs Chassis)
    float v_pack_minus;     // Pack- 전압 (vs Chassis)
    float v_mid;            // 중간점 전압
    float r_plus;           // Pack+ 절연 저항
    float r_minus;          // Pack- 절연 저항
    float r_total;          // 총 절연 저항
    bool fault;             // 절연 불량
} iso_monitor_t;

#define ISO_DIVIDER_R1      1000000.0f  // 1MΩ
#define ISO_DIVIDER_R2      1000000.0f  // 1MΩ
#define ISO_THRESHOLD_OHM   50000.0f    // 50kΩ 경고
#define ISO_FAULT_OHM       10000.0f    // 10kΩ 폴트
```

## 절연 저항 계산

### 측정 원리

```
Pack+ ─────┬─────────────────┬───── Pack-
           │                 │
          Rp                Rn
           │                 │
           └────────┬────────┘
                    │
                   GND

Rp = Pack+ to GND 저항
Rn = Pack- to GND 저항
Riso = Rp // Rn (병렬)
```

### 구현

```c
// isolation_monitor.c

static iso_monitor_t g_iso;

void ISO_Measure(void) {
    float v_pack = BMS_GetPackVoltage() / 1000.0f;  // V
    
    // ADC로 측정 (분압 회로 경유)
    float v_plus = ISO_GetVoltagePlus();   // Pack+ vs GND
    float v_minus = ISO_GetVoltageMinus(); // Pack- vs GND
    
    g_iso.v_pack_plus = v_plus;
    g_iso.v_pack_minus = v_minus;
    
    // 절연 저항 계산
    // 분압 공식: Vmeas = Vpack × Riso / (Rdiv + Riso)
    // Riso = Rdiv × Vmeas / (Vpack - Vmeas)
    
    if (v_pack > v_plus && v_plus > 0.1f) {
        g_iso.r_plus = ISO_DIVIDER_R1 * v_plus / (v_pack - v_plus);
    } else {
        g_iso.r_plus = 10000000.0f;  // 매우 높음 (10MΩ)
    }
    
    if (v_minus > 0.1f) {
        g_iso.r_minus = ISO_DIVIDER_R2 * (v_pack - v_minus) / v_minus;
    } else {
        g_iso.r_minus = 10000000.0f;  // 매우 높음
    }
    
    // 총 절연 저항 (병렬)
    g_iso.r_total = (g_iso.r_plus * g_iso.r_minus) / 
                    (g_iso.r_plus + g_iso.r_minus);
}
```

### 전압 측정

```c
float ISO_GetVoltagePlus(void) {
    // 고전압 분압기 경유
    // 100:1 분압 → 72V → 0.72V
    uint16_t adc = ADC_Read(ADC_CH_ISO_PLUS);
    float v_adc = (adc / 4096.0f) * 3.3f;
    float v_hv = v_adc * 100.0f;  // 분압 역산
    
    return v_hv;
}

float ISO_GetVoltageMinus(void) {
    uint16_t adc = ADC_Read(ADC_CH_ISO_MINUS);
    float v_adc = (adc / 4096.0f) * 3.3f;
    float v_hv = v_adc * 100.0f;
    
    return v_hv;
}
```

## 방법 2: 펄스 주입법

더 정확한 방법:

```
        ┌─────── Pulse Generator
        │
   ┌────┴────┐
   │  SW1    │──── Pack+
   └────┬────┘
        │
        R (측정 저항)
        │
   ┌────┴────┐
   │  SW2    │──── Pack-
   └─────────┘
        │
       GND
```

### 원리

1. SW1 ON: Pack+에 펄스 인가, 응답 측정
2. SW2 ON: Pack-에 펄스 인가, 응답 측정
3. 응답으로 절연 저항 계산

### 장점

- DC 오프셋 영향 없음
- 노이즈에 강함
- 연속 모니터링 가능

```c
// 펄스 주입 방식 (간략화)
void ISO_PulseMeasure(void) {
    float r_iso;
    
    // Pack+ 펄스
    ISO_SetPulse(ISO_PULSE_PLUS);
    HAL_Delay(10);
    float v_resp_plus = ISO_GetResponse();
    
    // Pack- 펄스  
    ISO_SetPulse(ISO_PULSE_MINUS);
    HAL_Delay(10);
    float v_resp_minus = ISO_GetResponse();
    
    // 펄스 OFF
    ISO_SetPulse(ISO_PULSE_OFF);
    
    // 저항 계산 (펄스 응답 기반)
    // R_iso = V_pulse / I_response
    // ...
}
```

## 절연 상태 판정

```c
typedef enum {
    ISO_STATE_NORMAL = 0,   // 정상
    ISO_STATE_WARNING,      // 경고 (저하)
    ISO_STATE_FAULT,        // 폴트 (위험)
    ISO_STATE_MEASURING     // 측정 중
} iso_state_t;

void ISO_CheckStatus(void) {
    ISO_Measure();
    
    float r_per_volt = g_iso.r_total / (g_bms.pack_voltage_mv / 1000.0f);
    
    if (g_iso.r_total < ISO_FAULT_OHM) {
        // 심각한 절연 불량
        g_iso.state = ISO_STATE_FAULT;
        BMS_SetFault(FAULT_ISO_FAILURE);
        BMS_EmergencyShutdown();  // 즉시 차단!
        
    } else if (g_iso.r_total < ISO_THRESHOLD_OHM) {
        // 절연 저하 경고
        g_iso.state = ISO_STATE_WARNING;
        BMS_SetWarning(WARN_ISO_LOW);
        
        Log_Event(EVT_ISO_WARNING, 0,
                  (uint16_t)(g_iso.r_total / 100),  // 100Ω 단위
                  (uint16_t)(r_per_volt), 0);
                  
    } else {
        // 정상
        g_iso.state = ISO_STATE_NORMAL;
    }
}
```

## 비대칭 절연 감지

```c
void ISO_CheckAsymmetry(void) {
    // Pack+ vs Pack- 절연 불균형
    float ratio = g_iso.r_plus / g_iso.r_minus;
    
    if (ratio > 10.0f || ratio < 0.1f) {
        // 한쪽이 10배 이상 낮음 → 특정 위치 문제
        
        if (g_iso.r_plus < g_iso.r_minus) {
            // Pack+ 쪽 절연 불량
            Log_Event(EVT_ISO_ASYMMETRIC, 0, 1,  // Plus side
                      (uint16_t)(g_iso.r_plus / 100), 0);
        } else {
            // Pack- 쪽 절연 불량
            Log_Event(EVT_ISO_ASYMMETRIC, 0, 2,  // Minus side
                      (uint16_t)(g_iso.r_minus / 100), 0);
        }
    }
}
```

## 상태 머신

```c
void ISO_StateMachine(void) {
    static uint32_t measure_interval = 0;
    
    // 1초 주기 측정
    if (HAL_GetTick() - measure_interval < 1000) {
        return;
    }
    measure_interval = HAL_GetTick();
    
    switch (g_iso.state) {
    case ISO_STATE_NORMAL:
        // 정상 → 주기적 측정
        ISO_Measure();
        ISO_CheckStatus();
        break;
        
    case ISO_STATE_WARNING:
        // 경고 → 더 자주 측정
        ISO_Measure();
        ISO_CheckStatus();
        
        // 3회 연속 정상이면 복귀
        static int normal_count = 0;
        if (g_iso.r_total >= ISO_THRESHOLD_OHM) {
            normal_count++;
            if (normal_count >= 3) {
                g_iso.state = ISO_STATE_NORMAL;
                BMS_ClearWarning(WARN_ISO_LOW);
                normal_count = 0;
            }
        } else {
            normal_count = 0;
        }
        break;
        
    case ISO_STATE_FAULT:
        // 폴트 → 시스템 차단 유지
        // 수동 리셋 필요
        break;
        
    case ISO_STATE_MEASURING:
        // 측정 중 (펄스 방식)
        break;
    }
}
```

## CAN 보고

```c
void ISO_SendCANStatus(void) {
    uint8_t data[8];
    
    // 0x230: 절연 상태 메시지
    data[0] = g_iso.state;
    data[1] = (uint8_t)(g_iso.r_total / 1000);        // kΩ 단위
    data[2] = (uint8_t)(g_iso.r_total / 1000 >> 8);
    data[3] = (uint8_t)(g_iso.r_plus / 1000);
    data[4] = (uint8_t)(g_iso.r_minus / 1000);
    data[5] = 0;  // Reserved
    data[6] = 0;
    data[7] = 0;
    
    CAN_Transmit(0x230, data, 8);
}
```

## 전용 IC 사용

더 정확한 측정을 위한 전용 IC:

```c
// Bender ISOMETER 또는 유사 IC
// I2C/SPI 통신으로 절연 저항 읽기

typedef struct {
    uint32_t r_plus_ohm;
    uint32_t r_minus_ohm;
    uint32_t r_total_ohm;
    uint8_t status;
    uint8_t fault_location;
} imd_data_t;

HAL_StatusTypeDef IMD_ReadData(imd_data_t *data) {
    uint8_t buf[12];
    
    // I2C 읽기
    HAL_I2C_Mem_Read(&hi2c1, IMD_I2C_ADDR, 0x00, 
                     I2C_MEMADD_SIZE_8BIT, buf, 12, 100);
    
    data->r_plus_ohm = buf[0] | (buf[1] << 8) | 
                       (buf[2] << 16) | (buf[3] << 24);
    data->r_minus_ohm = buf[4] | (buf[5] << 8) | 
                        (buf[6] << 16) | (buf[7] << 24);
    data->r_total_ohm = buf[8] | (buf[9] << 8) | 
                        (buf[10] << 16) | (buf[11] << 24);
    
    return HAL_OK;
}
```

## 삽질: 측정 노이즈

고임피던스 측정이라 노이즈에 민감:

```c
// 잘못된 코드: 단일 측정
float r = ISO_MeasureOnce();  // 노이즈 심함

// 올바른 코드: 다중 샘플 평균
#define ISO_SAMPLES 16

float ISO_MeasureFiltered(void) {
    float sum = 0;
    
    for (int i = 0; i < ISO_SAMPLES; i++) {
        sum += ISO_MeasureOnce();
        HAL_Delay(10);
    }
    
    return sum / ISO_SAMPLES;
}
```

## 삽질: 습도 영향

습도가 높으면 절연 저항 감소:

```c
// 온습도 보상 (간략화)
float ISO_HumidityCompensate(float r_measured, float humidity_pct) {
    // 습도 60% 이상에서 보정
    if (humidity_pct > 60.0f) {
        float factor = 1.0f + (humidity_pct - 60.0f) * 0.02f;
        return r_measured * factor;
    }
    return r_measured;
}
```

## Part 7 완료!

고급 기능편 완료:

| # | 제목 | 핵심 내용 |
|---|------|----------|
| #21 | SOH 추정 | 용량, 저항 기반 |
| #22 | 칼만 필터 | SOC 정밀도 향상 |
| #23 | 프리차지 | 돌입 전류 방지 |
| #24 | 절연 모니터링 | 안전 확보 |

---

## 정리

| 항목 | 값/설명 |
|------|---------|
| 측정 방법 | 저항 분압 / 펄스 주입 |
| 경고 임계값 | 50kΩ |
| 폴트 임계값 | 10kΩ |
| 요구 저항 | >100Ω/V |
| 측정 주기 | 1초 |
| 비대칭 감지 | 10:1 비율 |

**안전 조치**:
1. 경고: 운전자 알림
2. 폴트: 즉시 차단
3. 비대칭: 위치 특정

---

## 시리즈 네비게이션

**Part 7: 고급 기능편** ✅
- [#21 - SOH 추정](/posts/bms/ad7280a-bms-dev-21/)
- [#22 - 칼만 필터 적용](/posts/bms/ad7280a-bms-dev-22/)
- [#23 - 프리차지 회로](/posts/bms/ad7280a-bms-dev-23/)
- **#24 - 절연 모니터링** ← 현재 글

**다음**: Part 8 - 실전 적용편 (72V 팩 연결, 트러블슈팅, EMC, 양산)

---

## 참고 자료

- [UN ECE R100 - Electric Vehicle Safety](https://unece.org/transport/standards/transport/vehicle-regulations-wp29)
- [Bender ISOMETER Application](https://www.bender.de/en/products/ground-fault-monitoring/isometer)
- [Isolation Monitoring for EVs](https://www.analog.com/en/technical-articles/isolation-monitoring-in-electric-vehicles.html)
