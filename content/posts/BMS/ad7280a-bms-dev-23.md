---
title: "AD7280A BMS 개발 삽질기 #23 - 프리차지 회로"
date: 2024-12-13
draft: false
tags: ["AD7280A", "BMS", "STM32", "프리차지", "돌입전류", "인버터"]
categories: ["BMS 개발"]
series: ["AD7280A BMS 개발"]
summary: "배터리 연결 순간 수백 A? 프리차지 회로로 돌입 전류를 제어하고 커패시터를 안전하게 충전한다."
---

## 지난 글 요약

[지난 글](/posts/bms/ad7280a-bms-dev-22/)에서 칼만 필터를 적용했다. SOC 오차 ±2%. 이제 **하드웨어 관련** 고급 기능인 프리차지를 다루자.

## 돌입 전류 문제

### 인버터의 DC 링크 커패시터

```
배터리 ─────────────────────────┬───────────────
         72V                    │
                          ┌─────┴─────┐
                          │           │
                          │  DC Link  │ 4700μF
                          │    Cap    │
                          │           │
                          └─────┬─────┘
                                │
────────────────────────────────┴───────────────
                                GND
```

### 연결 순간

커패시터가 방전된 상태에서 배터리 연결 시:

```
I = V / ESR

V = 72V
ESR = 10mΩ (배선 + 접촉 저항)
I = 72V / 0.01Ω = 7200A !!!
```

**문제점**:
- 릴레이/접촉기 손상
- 배터리 셀 스트레스
- 커넥터 아크 발생
- 퓨즈 용단

## 프리차지 회로 원리

```
        Main Contactor
배터리 ─────┤  K1  ├───────────────────┬─────── Load+
  +         (NC)                       │
                                 ┌─────┴─────┐
        Precharge Path           │           │
            ┌───┐               │  DC Link  │
        ────┤ R ├──┤  K2  ├─────│    Cap    │
            └───┘   Precharge   │           │
            50Ω     Relay       └─────┬─────┘
                                      │
배터리 ───────────────────────────────┴─────── Load-
  -
```

### 동작 순서

1. **초기**: K1 OFF, K2 OFF, 커패시터 방전
2. **프리차지 시작**: K2 ON (저항 경유 충전)
3. **프리차지 완료**: 커패시터 전압 ≈ 배터리 전압
4. **메인 접촉**: K1 ON (메인 전류 경로)
5. **프리차지 종료**: K2 OFF (저항 열 방지)

## 프리차지 저항 설계

### 충전 시간 계산

RC 시정수:
```
τ = R × C

R = 50Ω
C = 4700μF = 0.0047F
τ = 50 × 0.0047 = 0.235초

99% 충전: 5τ = 1.17초
```

### 최대 전류

```
I_max = V / R = 72V / 50Ω = 1.44A
```

안전한 수준!

### 저항 발열

```
P_peak = I² × R = 1.44² × 50 = 104W (순간)
E_total = 0.5 × C × V² = 0.5 × 0.0047 × 72² = 12.2J
```

저항 정격: 10W 이상 (순간 발열 견딤)

## 하드웨어 인터페이스

### GPIO 할당

```c
// precharge.h

#define PRECHARGE_RELAY_PORT    GPIOB
#define PRECHARGE_RELAY_PIN     GPIO_PIN_10
#define MAIN_CONTACTOR_PORT     GPIOB
#define MAIN_CONTACTOR_PIN      GPIO_PIN_11
#define VOLTAGE_SENSE_ADC       ADC_CHANNEL_8   // DC Link 전압
```

### 릴레이 드라이버

```c
// 릴레이 제어 (Active High + FET 드라이버)
void Precharge_RelayOn(void) {
    HAL_GPIO_WritePin(PRECHARGE_RELAY_PORT, PRECHARGE_RELAY_PIN, GPIO_PIN_SET);
}

void Precharge_RelayOff(void) {
    HAL_GPIO_WritePin(PRECHARGE_RELAY_PORT, PRECHARGE_RELAY_PIN, GPIO_PIN_RESET);
}

void MainContactor_On(void) {
    HAL_GPIO_WritePin(MAIN_CONTACTOR_PORT, MAIN_CONTACTOR_PIN, GPIO_PIN_SET);
}

void MainContactor_Off(void) {
    HAL_GPIO_WritePin(MAIN_CONTACTOR_PORT, MAIN_CONTACTOR_PIN, GPIO_PIN_RESET);
}
```

## 프리차지 상태 머신

```c
// precharge.c

typedef enum {
    PRECHARGE_IDLE = 0,
    PRECHARGE_START,
    PRECHARGE_CHARGING,
    PRECHARGE_VERIFY,
    PRECHARGE_COMPLETE,
    PRECHARGE_FAULT
} precharge_state_t;

typedef struct {
    precharge_state_t state;
    uint32_t start_time;
    uint32_t timeout_ms;
    float battery_voltage;
    float dclink_voltage;
    float target_ratio;     // 목표 전압 비율 (0.9 = 90%)
} precharge_t;

static precharge_t g_precharge;

#define PRECHARGE_TIMEOUT_MS    5000    // 5초 타임아웃
#define PRECHARGE_TARGET_RATIO  0.90f   // 90% 도달 시 완료
```

### 상태 머신 구현

```c
void Precharge_Init(void) {
    g_precharge.state = PRECHARGE_IDLE;
    g_precharge.timeout_ms = PRECHARGE_TIMEOUT_MS;
    g_precharge.target_ratio = PRECHARGE_TARGET_RATIO;
    
    // 릴레이 초기 상태
    Precharge_RelayOff();
    MainContactor_Off();
}

void Precharge_Start(void) {
    if (g_precharge.state != PRECHARGE_IDLE) {
        return;  // 이미 진행 중
    }
    
    g_precharge.state = PRECHARGE_START;
}

void Precharge_StateMachine(void) {
    // 전압 측정
    g_precharge.battery_voltage = BMS_GetPackVoltage() / 1000.0f;  // V
    g_precharge.dclink_voltage = Precharge_GetDCLinkVoltage();
    
    switch (g_precharge.state) {
    case PRECHARGE_IDLE:
        // 대기
        break;
        
    case PRECHARGE_START:
        // 프리차지 릴레이 ON
        Precharge_RelayOn();
        g_precharge.start_time = HAL_GetTick();
        g_precharge.state = PRECHARGE_CHARGING;
        break;
        
    case PRECHARGE_CHARGING:
        {
            // 타임아웃 체크
            uint32_t elapsed = HAL_GetTick() - g_precharge.start_time;
            if (elapsed > g_precharge.timeout_ms) {
                g_precharge.state = PRECHARGE_FAULT;
                break;
            }
            
            // 목표 전압 도달 체크
            float ratio = g_precharge.dclink_voltage / g_precharge.battery_voltage;
            if (ratio >= g_precharge.target_ratio) {
                g_precharge.state = PRECHARGE_VERIFY;
            }
        }
        break;
        
    case PRECHARGE_VERIFY:
        // 잠시 대기 후 최종 확인
        HAL_Delay(100);
        
        {
            float ratio = g_precharge.dclink_voltage / g_precharge.battery_voltage;
            if (ratio >= g_precharge.target_ratio) {
                // 메인 접촉기 ON
                MainContactor_On();
                HAL_Delay(50);  // 접촉 안정화
                
                // 프리차지 릴레이 OFF
                Precharge_RelayOff();
                
                g_precharge.state = PRECHARGE_COMPLETE;
            } else {
                // 전압 떨어짐 → 부하 문제
                g_precharge.state = PRECHARGE_FAULT;
            }
        }
        break;
        
    case PRECHARGE_COMPLETE:
        // 완료 상태 유지
        break;
        
    case PRECHARGE_FAULT:
        // 모든 릴레이 OFF
        Precharge_RelayOff();
        MainContactor_Off();
        // 에러 처리
        break;
    }
}
```

### DC 링크 전압 측정

```c
float Precharge_GetDCLinkVoltage(void) {
    // ADC 읽기
    uint16_t adc_value = ADC_Read(VOLTAGE_SENSE_ADC);
    
    // 전압 분배기: 100k / 3.3k
    // V_dc = V_adc × (100k + 3.3k) / 3.3k
    float v_adc = (adc_value / 4096.0f) * 3.3f;
    float v_dc = v_adc * (100.0f + 3.3f) / 3.3f;
    
    return v_dc;
}
```

## 프리차지 시퀀스 다이어그램

```
시간 →
────────────────────────────────────────────────────►

K1 (Main)     ─────────────────────────┬────────────
              OFF                      │ON

K2 (Precharge) ────────┬───────────────┴────────────
              OFF      │ON             OFF

DC Link 전압  ─────────/─────────────────────────────
              0V      /           72V (90%)

상태          IDLE → START → CHARGING → VERIFY → COMPLETE
              
시간          0      0.1     1.5       1.6      1.7 (초)
```

## 폴트 처리

### 타임아웃 폴트

```c
void Precharge_HandleFault(void) {
    switch (g_precharge.fault_code) {
    case FAULT_PRECHARGE_TIMEOUT:
        // 원인: 커패시터 용량 초과, 저항 단선, 부하 단락
        // 조치: 릴레이 OFF, 에러 로그
        break;
        
    case FAULT_PRECHARGE_DROP:
        // 원인: 부하측 단락, 커패시터 불량
        // 조치: 즉시 차단
        break;
        
    case FAULT_DCLINK_OVERVOLTAGE:
        // 원인: 회생 에너지, 센서 오류
        // 조치: 방전 저항 활성화
        break;
    }
    
    Precharge_RelayOff();
    MainContactor_Off();
    
    Log_Event(EVT_PRECHARGE_FAULT, g_precharge.fault_code, 
              (uint16_t)(g_precharge.dclink_voltage * 10), 0);
}
```

### 비정상 전압 감지

```c
void Precharge_CheckAnomalies(void) {
    // DC 링크 전압이 배터리보다 높음 (회생?)
    if (g_precharge.dclink_voltage > g_precharge.battery_voltage * 1.1f) {
        g_precharge.fault_code = FAULT_DCLINK_OVERVOLTAGE;
        g_precharge.state = PRECHARGE_FAULT;
    }
    
    // 충전 중 전압 감소 (단락?)
    static float last_voltage = 0;
    if (g_precharge.state == PRECHARGE_CHARGING) {
        if (g_precharge.dclink_voltage < last_voltage - 1.0f) {
            g_precharge.fault_code = FAULT_PRECHARGE_DROP;
            g_precharge.state = PRECHARGE_FAULT;
        }
    }
    last_voltage = g_precharge.dclink_voltage;
}
```

## 전류 제한 프리차지

저항 대신 전류 제어:

```c
// 고급 방식: PWM 제어 프리차지
typedef struct {
    float target_current;   // 목표 전류
    float actual_current;   // 실제 전류
    float pwm_duty;         // PWM 듀티
} current_limited_precharge_t;

void Precharge_CurrentLimited_Update(void) {
    // PI 제어
    float error = g_clp.target_current - g_clp.actual_current;
    
    static float integral = 0;
    integral += error * 0.01f;
    
    g_clp.pwm_duty = 0.5f * error + 0.1f * integral;
    
    // 범위 제한
    if (g_clp.pwm_duty > 1.0f) g_clp.pwm_duty = 1.0f;
    if (g_clp.pwm_duty < 0.0f) g_clp.pwm_duty = 0.0f;
    
    PWM_SetDuty(PRECHARGE_PWM_CH, g_clp.pwm_duty);
}
```

## BMS 통합

```c
// bms_control.c

void BMS_PowerOn(void) {
    // 1. 배터리 상태 확인
    if (!BMS_IsSystemReady()) {
        return;
    }
    
    // 2. 프리차지 시작
    Precharge_Start();
    
    // 3. 완료 대기
    while (g_precharge.state == PRECHARGE_CHARGING ||
           g_precharge.state == PRECHARGE_VERIFY) {
        Precharge_StateMachine();
        HAL_Delay(10);
    }
    
    // 4. 결과 확인
    if (g_precharge.state == PRECHARGE_COMPLETE) {
        g_bms.state = BMS_STATE_READY;
        LED_On(LED_POWER);
    } else {
        g_bms.state = BMS_STATE_FAULT;
        BMS_SetFault(FAULT_PRECHARGE);
    }
}

void BMS_PowerOff(void) {
    // 메인 접촉기 OFF
    MainContactor_Off();
    
    // 프리차지 릴레이도 확인
    Precharge_RelayOff();
    
    g_precharge.state = PRECHARGE_IDLE;
    g_bms.state = BMS_STATE_IDLE;
}
```

## 삽질: 릴레이 순서

메인 ON 전에 프리차지 OFF하면 아크 발생:

```c
// 잘못된 순서
Precharge_RelayOff();  // OFF 먼저
MainContactor_On();    // 아크! 전류 경로 없음

// 올바른 순서
MainContactor_On();    // ON 먼저
HAL_Delay(50);         // 안정화
Precharge_RelayOff();  // 그 다음 OFF
```

## 삽질: 전압 분배기 오차

고전압 분배기 정밀도 중요:

```c
// 1% 저항 오차 → 전압 오차
// 100k ±1% = 99k ~ 101k
// 측정 오차: 약 ±0.7V @ 72V

// 보정 적용
#define DCLINK_VOLTAGE_CAL    1.02f  // 교정 계수
float v_dc = v_raw * DCLINK_VOLTAGE_CAL;
```

## 정리

| 항목 | 값 |
|------|-----|
| 프리차지 저항 | 50Ω / 10W |
| DC 링크 커패시터 | 4700μF |
| 충전 시정수 | 0.235초 |
| 목표 전압 비율 | 90% |
| 타임아웃 | 5초 |
| 최대 전류 | 1.44A |

**시퀀스**:
1. K2 ON (프리차지 시작)
2. DC 링크 충전 대기
3. 90% 도달 확인
4. K1 ON (메인 접촉)
5. K2 OFF (프리차지 종료)

**다음 글에서**: 절연 모니터링 - 고전압 시스템의 안전.

---

## 시리즈 네비게이션

**Part 7: 고급 기능편**
- [#21 - SOH 추정](/posts/bms/ad7280a-bms-dev-21/)
- [#22 - 칼만 필터 적용](/posts/bms/ad7280a-bms-dev-22/)
- **#23 - 프리차지 회로** ← 현재 글
- #24 - 절연 모니터링

---

## 참고 자료

- [EV Precharge Circuit Design](https://www.ti.com/lit/an/slva729/slva729.pdf)
- [Contactor Selection for EVs](https://www.te.com/usa-en/products/relays-contactors-switches/contactors/ev-contactors.html)
- [DC Link Capacitor Selection](https://www.electronics-tutorials.ws/capacitor/cap_7.html)
