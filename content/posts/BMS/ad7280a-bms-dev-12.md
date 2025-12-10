---
title: "AD7280A BMS 개발 삽질기 #12 - 밸런싱 중 발열 관리"
date: 2024-12-09
draft: false
tags: ["BMS", "AD7280A", "임베디드", "배터리", "삽질", "밸런싱", "발열"]
categories: ["BMS 개발"]
summary: "밸런싱 저항이 뜨겁다. 만졌다가 데일 뻔했다. 열 관리가 필수다."
---

## 지난 글 요약

[지난 글](/posts/bms/ad7280a-bms-dev-11/)에서 밸런싱 알고리즘을 구현했다. 잘 동작하는데... 저항이 너무 뜨겁다!

## 발열의 원인

패시브 밸런싱 = **에너지를 열로 버리는 것**.

```
P = V² / R = I² × R

예: 4.2V 셀, 68Ω 저항
P = 4.2² / 68 = 0.26W

6셀 동시 밸런싱:
P_total = 0.26W × 6 = 1.56W
```

1.56W가 작은 PCB에서 발생하면 **상당히 뜨겁다**.

## 온도 상승 계산

### 저항 자체 온도

```
ΔT = P × Rth

0805 칩저항 열저항: ~100°C/W (대략)
ΔT = 0.26W × 100 = 26°C

주변 온도 25°C일 때:
저항 온도 = 25 + 26 = 51°C
```

**6개 밀집 배치하면?** 상호 열간섭으로 더 올라간다.

### 실측 결과

EVAL 보드에서 열화상 카메라로 측정:

| 조건 | 저항 온도 | PCB 온도 |
|------|-----------|----------|
| 밸런싱 OFF | 28°C | 27°C |
| 1셀 ON (10분) | 48°C | 32°C |
| 3셀 ON (10분) | 62°C | 41°C |
| 6셀 ON (10분) | 78°C | 52°C |

**6셀 전부 켜면 78°C!** 손 못 댄다.

## 문제 1: 배터리 과열

BMS 보드는 배터리 팩 **안에** 들어간다. 밸런싱 열이 배터리 셀에 전달되면 위험.

```
리튬이온 권장 온도: 0~45°C (충전 시)
밸런싱으로 PCB가 52°C면? → 셀도 영향받음
```

## 해결책 1: 듀티 제어

연속으로 밸런싱하지 말고, **ON/OFF 반복**.

```c
#define BALANCE_ON_TIME_MS    5000   // 5초 ON
#define BALANCE_OFF_TIME_MS   5000   // 5초 OFF (냉각)

typedef struct {
    uint8_t balance_mask;
    bool is_on;
    uint32_t last_toggle;
} duty_balance_t;

void duty_balance_control(duty_balance_t *ctx) {
    uint32_t now = HAL_GetTick();
    uint32_t elapsed = now - ctx->last_toggle;
    
    if (ctx->is_on) {
        // ON 상태 → OFF 시간 체크
        if (elapsed >= BALANCE_ON_TIME_MS) {
            ad7280a_set_balance(0, 0x00);  // OFF
            ctx->is_on = false;
            ctx->last_toggle = now;
        }
    } else {
        // OFF 상태 → ON 시간 체크
        if (elapsed >= BALANCE_OFF_TIME_MS) {
            ad7280a_set_balance(0, ctx->balance_mask);  // ON
            ctx->is_on = true;
            ctx->last_toggle = now;
        }
    }
}
```

**듀티비 50%**: 발열 절반, 밸런싱 시간 2배.

### 가변 듀티

온도에 따라 듀티비 조절:

```c
uint8_t calculate_duty_percent(float temp_c) {
    if (temp_c < 35.0f) return 100;  // 전력 밸런싱
    if (temp_c < 40.0f) return 75;
    if (temp_c < 45.0f) return 50;
    if (temp_c < 50.0f) return 25;
    return 0;  // 과열 → 중지
}
```

## 해결책 2: 인접 셀 회피

인접한 셀을 동시에 밸런싱하면 열이 집중된다.

```
나쁜 예:
Cell 1: ON  ← 열
Cell 2: ON  ← 열 (인접)
Cell 3: ON  ← 열 (인접)
→ 세 저항이 붙어서 과열

좋은 예 (체커보드):
Phase 1: Cell 1, 3, 5 ON (홀수)
Phase 2: Cell 2, 4, 6 ON (짝수)
→ 열 분산
```

```c
void checkerboard_balance(uint8_t full_mask) {
    static bool phase = false;
    uint8_t applied_mask;
    
    if (phase) {
        applied_mask = full_mask & 0x15;  // 0b010101 = Cell 1,3,5
    } else {
        applied_mask = full_mask & 0x2A;  // 0b101010 = Cell 2,4,6
    }
    
    ad7280a_set_balance(0, applied_mask);
    phase = !phase;
}
```

## 해결책 3: 온도 기반 제어

AD7280A의 AUX 채널에 **서미스터 연결**해서 온도 감시.

### 서미스터 읽기

```c
// AUX1에 10K NTC 연결
// Vref = 2.5V, 직렬 10K 저항

float read_thermistor_temp(uint16_t aux_raw) {
    // ADC 값 → 전압
    float v_aux = aux_raw * 0.976e-3f + 1.0f;  // AD7280A 공식
    
    // 전압 → 저항
    float r_therm = 10000.0f * v_aux / (2.5f - v_aux);
    
    // 저항 → 온도 (Steinhart-Hart 간략화)
    // B = 3950 (일반적인 NTC)
    float temp_k = 1.0f / (1.0f/298.15f + log(r_therm/10000.0f)/3950.0f);
    
    return temp_k - 273.15f;  // °C
}
```

### 온도 기반 밸런싱 제어

```c
#define TEMP_WARNING_C    45.0f
#define TEMP_CRITICAL_C   55.0f
#define TEMP_RESUME_C     40.0f

typedef enum {
    THERMAL_OK,
    THERMAL_WARNING,
    THERMAL_CRITICAL
} thermal_state_t;

thermal_state_t thermal_state = THERMAL_OK;

void thermal_balance_control(float pcb_temp, uint8_t *balance_mask) {
    switch (thermal_state) {
    case THERMAL_OK:
        if (pcb_temp > TEMP_CRITICAL_C) {
            thermal_state = THERMAL_CRITICAL;
            *balance_mask = 0;  // 즉시 중지
            printf("THERMAL CRITICAL: %.1f°C\n", pcb_temp);
        } else if (pcb_temp > TEMP_WARNING_C) {
            thermal_state = THERMAL_WARNING;
            // 듀티 감소
        }
        break;
        
    case THERMAL_WARNING:
        if (pcb_temp > TEMP_CRITICAL_C) {
            thermal_state = THERMAL_CRITICAL;
            *balance_mask = 0;
        } else if (pcb_temp < TEMP_RESUME_C) {
            thermal_state = THERMAL_OK;
        }
        break;
        
    case THERMAL_CRITICAL:
        *balance_mask = 0;  // 밸런싱 금지
        if (pcb_temp < TEMP_RESUME_C) {
            thermal_state = THERMAL_OK;
            printf("THERMAL OK: %.1f°C, resume balancing\n", pcb_temp);
        }
        break;
    }
}
```

## 해결책 4: 하드웨어 개선

### 저항값 증가

```
68Ω → 100Ω 변경 시:
P = 4.2² / 100 = 0.18W (30% 감소)

단점: 밸런싱 시간 1.5배 증가
```

### 저항 분산 배치

PCB 설계 시 밸런싱 저항을 **멀리 배치**.

```
[BAD]                    [GOOD]
R1 R2 R3 R4 R5 R6       R1    R3    R5
(한 줄 밀집)                R2    R4    R6
                        (분산 배치)
```

### 방열판 / 열전도 패드

고출력 밸런싱 시 저항에 작은 방열판 부착.

## 종합 열 관리 코드

```c
typedef struct {
    // 설정
    uint8_t duty_percent;
    uint16_t on_time_ms;
    uint16_t off_time_ms;
    
    // 상태
    uint8_t target_mask;
    uint8_t active_mask;
    bool is_on_phase;
    uint32_t phase_start;
    thermal_state_t thermal;
} thermal_balance_ctx_t;

void thermal_balance_update(thermal_balance_ctx_t *ctx, float pcb_temp) {
    uint32_t now = HAL_GetTick();
    
    // 1. 온도 체크
    thermal_balance_control(pcb_temp, &ctx->target_mask);
    
    if (ctx->target_mask == 0) {
        ad7280a_set_balance(0, 0x00);
        return;
    }
    
    // 2. 듀티 계산
    ctx->duty_percent = calculate_duty_percent(pcb_temp);
    ctx->on_time_ms = 100 * ctx->duty_percent;      // 0~10초
    ctx->off_time_ms = 10000 - ctx->on_time_ms;
    
    // 3. 듀티 사이클 제어
    uint32_t elapsed = now - ctx->phase_start;
    
    if (ctx->is_on_phase) {
        if (elapsed >= ctx->on_time_ms) {
            // 체커보드로 OFF
            ad7280a_set_balance(0, 0x00);
            ctx->is_on_phase = false;
            ctx->phase_start = now;
        }
    } else {
        if (elapsed >= ctx->off_time_ms) {
            // 체커보드로 ON (홀수/짝수 번갈아)
            static bool odd_phase = true;
            ctx->active_mask = ctx->target_mask & (odd_phase ? 0x15 : 0x2A);
            ad7280a_set_balance(0, ctx->active_mask);
            ctx->is_on_phase = true;
            ctx->phase_start = now;
            odd_phase = !odd_phase;
        }
    }
}
```

## 실측 결과 (열 관리 적용 후)

| 조건 | 저항 온도 | PCB 온도 |
|------|-----------|----------|
| 듀티 100%, 6셀 | 78°C | 52°C |
| 듀티 50%, 6셀 | 58°C | 42°C |
| 듀티 50%, 체커보드 | 52°C | 38°C |
| 온도 제어 (목표 40°C) | 48°C | 39°C |

**체커보드 + 듀티 50%**: 26°C 감소!

## 삽질 포인트 정리

1. **발열량 계산** - P = V²/R, 예상보다 뜨겁다
2. **듀티 제어** - 50% 듀티로 발열 절반
3. **체커보드 패턴** - 인접 셀 회피로 열 분산
4. **온도 감시** - 서미스터로 실시간 모니터링
5. **히스테리시스** - 떨림 방지 (시작/중지 온도 분리)
6. **하드웨어** - 저항값, 배치, 방열판

이제 밸런싱 파트는 끝! 다음은 펌웨어 아키텍처로 넘어간다.

---

## 참고 자료

- [AD7280A Datasheet - Auxiliary ADC Inputs](https://www.analog.com/media/en/technical-documentation/data-sheets/ad7280a.pdf)
- [NTC Thermistor Calculation](https://www.tdk-electronics.tdk.com/inf/50/db/ntc/NTC_Mini_sensors_S861.pdf)
