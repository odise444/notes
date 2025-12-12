---
title: "AD7280A BMS 개발 삽질기 #16 - 보호 로직 통합"
date: 2024-12-11
draft: false
tags: ["AD7280A", "BMS", "STM32", "배터리", "보호", "상태머신", "FET"]
categories: ["BMS 개발"]
summary: "알람, 밸런싱, FET 제어를 하나로. BMS의 두뇌를 만들어보자."
---

## 지난 글 요약

[지난 글](/posts/bms/ad7280a-bms-dev-15/)에서 알람 상태를 읽는 방법을 배웠다. 이제 모든 것을 **하나의 보호 로직**으로 통합하자.

## BMS 보호 시스템 구조

```
┌─────────────────────────────────────────────────────────────┐
│                     BMS 보호 시스템                         │
├─────────────────────────────────────────────────────────────┤
│                                                             │
│   ┌─────────┐    ┌─────────────┐    ┌─────────────────┐    │
│   │ AD7280A │───▶│  보호 로직   │───▶│  FET 제어       │    │
│   │ (측정)  │    │  (판단)     │    │  (실행)         │    │
│   └─────────┘    └─────────────┘    └─────────────────┘    │
│        │               │                    │               │
│        ▼               ▼                    ▼               │
│   ┌─────────┐    ┌─────────────┐    ┌─────────────────┐    │
│   │ ALERT   │    │ 상태 머신   │    │  CHG_FET        │    │
│   │ 인터럽트 │    │ (FSM)      │    │  DSG_FET        │    │
│   └─────────┘    └─────────────┘    └─────────────────┘    │
│                                                             │
└─────────────────────────────────────────────────────────────┘
```

## BMS 상태 정의

```c
typedef enum {
    BMS_STATE_INIT = 0,      // 초기화 중
    BMS_STATE_IDLE,          // 대기 (충방전 없음)
    BMS_STATE_CHARGING,      // 충전 중
    BMS_STATE_DISCHARGING,   // 방전 중
    BMS_STATE_BALANCING,     // 밸런싱 중
    BMS_STATE_FAULT_OV,      // 과전압 폴트
    BMS_STATE_FAULT_UV,      // 저전압 폴트
    BMS_STATE_FAULT_OT,      // 과온도 폴트
    BMS_STATE_FAULT_UT,      // 저온도 폴트
    BMS_STATE_FAULT_COMM,    // 통신 오류
    BMS_STATE_SHUTDOWN,      // 셧다운
} bms_state_t;

const char* bms_state_str[] = {
    "INIT", "IDLE", "CHARGING", "DISCHARGING", "BALANCING",
    "FAULT_OV", "FAULT_UV", "FAULT_OT", "FAULT_UT", "FAULT_COMM",
    "SHUTDOWN"
};
```

## 보호 임계값 설정

```c
typedef struct {
    // 전압 임계값 (V)
    float cell_ov_threshold;      // 과전압 (충전 중단)
    float cell_ov_recover;        // 과전압 복구
    float cell_uv_threshold;      // 저전압 (방전 중단)
    float cell_uv_recover;        // 저전압 복구
    
    // 온도 임계값 (°C)
    float temp_ot_charge;         // 충전 시 과온도
    float temp_ot_discharge;      // 방전 시 과온도
    float temp_ut_charge;         // 충전 시 저온도
    float temp_ut_discharge;      // 방전 시 저온도
    float temp_recover;           // 온도 복구 마진
    
    // 밸런싱 임계값 (mV)
    float balance_start_delta;    // 밸런싱 시작 편차
    float balance_stop_delta;     // 밸런싱 중단 편차
    float balance_min_voltage;    // 밸런싱 최소 전압
    
    // 타이밍 (ms)
    uint32_t fault_delay;         // 폴트 판정 지연
    uint32_t recover_delay;       // 복구 판정 지연
} bms_protection_config_t;

// LiFePO4 기본 설정
const bms_protection_config_t BMS_CONFIG_LFP = {
    .cell_ov_threshold = 3.65f,
    .cell_ov_recover = 3.50f,
    .cell_uv_threshold = 2.50f,
    .cell_uv_recover = 2.80f,
    
    .temp_ot_charge = 45.0f,
    .temp_ot_discharge = 55.0f,
    .temp_ut_charge = 0.0f,
    .temp_ut_discharge = -20.0f,
    .temp_recover = 5.0f,
    
    .balance_start_delta = 30.0f,   // 30mV
    .balance_stop_delta = 10.0f,    // 10mV
    .balance_min_voltage = 3.00f,
    
    .fault_delay = 1000,            // 1초
    .recover_delay = 5000,          // 5초
};
```

## BMS 컨텍스트 구조체

```c
#define BMS_MAX_DEVICES     8
#define BMS_MAX_CELLS       (BMS_MAX_DEVICES * 6)

typedef struct {
    // 상태
    bms_state_t state;
    bms_state_t prev_state;
    uint32_t state_enter_time;
    
    // 하드웨어
    ad7280a_handle_t *ad7280a;
    uint8_t num_devices;
    uint8_t num_cells;
    
    // 측정값
    float cell_voltages[BMS_MAX_CELLS];
    float temperatures[BMS_MAX_DEVICES];
    float total_voltage;
    float min_cell_voltage;
    float max_cell_voltage;
    float delta_voltage;
    uint8_t min_cell_index;
    uint8_t max_cell_index;
    
    // 알람
    ad7280a_full_alert_t alerts[BMS_MAX_DEVICES];
    bool alert_pending;
    
    // 밸런싱
    uint8_t balance_bitmap[BMS_MAX_DEVICES];
    bool balancing_active;
    
    // FET 상태
    bool chg_fet_on;
    bool dsg_fet_on;
    
    // 폴트 카운터 (디바운싱)
    uint32_t ov_fault_count;
    uint32_t uv_fault_count;
    uint32_t ot_fault_count;
    uint32_t comm_fault_count;
    
    // 설정
    bms_protection_config_t config;
    
    // 통계
    uint32_t total_faults;
    uint32_t last_fault_time;
    
} bms_context_t;

// 전역 BMS 컨텍스트
static bms_context_t g_bms;
```

## BMS 초기화

```c
/**
 * @brief BMS 시스템 초기화
 */
HAL_StatusTypeDef bms_init(ad7280a_handle_t *ad7280a, uint8_t num_devices)
{
    memset(&g_bms, 0, sizeof(bms_context_t));
    
    g_bms.ad7280a = ad7280a;
    g_bms.num_devices = num_devices;
    g_bms.num_cells = num_devices * 6;
    g_bms.config = BMS_CONFIG_LFP;
    g_bms.state = BMS_STATE_INIT;
    
    printf("BMS Init: %d devices, %d cells\n", num_devices, g_bms.num_cells);
    
    // AD7280A 알람 임계값 설정
    ad7280a_alarm_config_t alarm_cfg = {
        .cell_ov_v = g_bms.config.cell_ov_threshold,
        .cell_uv_v = g_bms.config.cell_uv_threshold,
        .aux_ov_v = 3.0f,  // NTC 고온
        .aux_uv_v = 0.5f,  // NTC 저온
    };
    ad7280a_set_alarm_thresholds(ad7280a, &alarm_cfg);
    
    // 알람 활성화
    ad7280a_enable_alarms(ad7280a);
    
    // FET 초기 상태 (OFF)
    bms_set_charge_fet(false);
    bms_set_discharge_fet(false);
    
    // 상태 전이
    bms_change_state(BMS_STATE_IDLE);
    
    return HAL_OK;
}
```

## 측정 및 업데이트

```c
/**
 * @brief 모든 측정값 업데이트
 */
HAL_StatusTypeDef bms_update_measurements(void)
{
    float min_v = 999.0f, max_v = 0.0f;
    float total_v = 0.0f;
    uint8_t min_idx = 0, max_idx = 0;
    
    // 모든 디바이스 셀 전압 읽기
    for (uint8_t dev = 0; dev < g_bms.num_devices; dev++) {
        float voltages[6];
        
        if (ad7280a_read_cell_voltages(g_bms.ad7280a, dev, voltages) != HAL_OK) {
            g_bms.comm_fault_count++;
            return HAL_ERROR;
        }
        
        for (int i = 0; i < 6; i++) {
            uint8_t cell_idx = dev * 6 + i;
            g_bms.cell_voltages[cell_idx] = voltages[i];
            total_v += voltages[i];
            
            if (voltages[i] < min_v) {
                min_v = voltages[i];
                min_idx = cell_idx;
            }
            if (voltages[i] > max_v) {
                max_v = voltages[i];
                max_idx = cell_idx;
            }
        }
        
        // 알람 상태 읽기
        ad7280a_read_full_alert(g_bms.ad7280a, dev, &g_bms.alerts[dev]);
        
        // 온도 읽기 (AUX 채널)
        float aux[6];
        ad7280a_read_aux_voltages(g_bms.ad7280a, dev, aux);
        g_bms.temperatures[dev] = ntc_voltage_to_temp(aux[0]);
    }
    
    g_bms.total_voltage = total_v;
    g_bms.min_cell_voltage = min_v;
    g_bms.max_cell_voltage = max_v;
    g_bms.min_cell_index = min_idx;
    g_bms.max_cell_index = max_idx;
    g_bms.delta_voltage = (max_v - min_v) * 1000;  // mV
    
    g_bms.comm_fault_count = 0;  // 통신 성공
    
    return HAL_OK;
}
```

## 상태 머신

```c
/**
 * @brief 상태 전이
 */
void bms_change_state(bms_state_t new_state)
{
    if (g_bms.state != new_state) {
        printf("BMS: %s -> %s\n", 
               bms_state_str[g_bms.state], 
               bms_state_str[new_state]);
        
        g_bms.prev_state = g_bms.state;
        g_bms.state = new_state;
        g_bms.state_enter_time = HAL_GetTick();
        
        // 상태 진입 동작
        bms_on_state_enter(new_state);
    }
}

/**
 * @brief 상태 진입 시 동작
 */
void bms_on_state_enter(bms_state_t state)
{
    switch (state) {
    case BMS_STATE_IDLE:
        LED_SetState(LED_STATUS, LED_ON);
        LED_SetState(LED_FAULT, LED_OFF);
        break;
        
    case BMS_STATE_CHARGING:
        bms_set_charge_fet(true);
        LED_SetState(LED_STATUS, LED_BLINK_SLOW);
        break;
        
    case BMS_STATE_DISCHARGING:
        bms_set_discharge_fet(true);
        LED_SetState(LED_STATUS, LED_BLINK_SLOW);
        break;
        
    case BMS_STATE_BALANCING:
        bms_update_balance_bitmap();
        bms_apply_balance();
        LED_SetState(LED_STATUS, LED_BLINK_FAST);
        break;
        
    case BMS_STATE_FAULT_OV:
    case BMS_STATE_FAULT_UV:
    case BMS_STATE_FAULT_OT:
    case BMS_STATE_FAULT_UT:
    case BMS_STATE_FAULT_COMM:
        bms_set_charge_fet(false);
        bms_set_discharge_fet(false);
        bms_stop_balance();
        LED_SetState(LED_FAULT, LED_BLINK_FAST);
        g_bms.total_faults++;
        g_bms.last_fault_time = HAL_GetTick();
        break;
        
    case BMS_STATE_SHUTDOWN:
        bms_set_charge_fet(false);
        bms_set_discharge_fet(false);
        bms_stop_balance();
        LED_SetState(LED_STATUS, LED_OFF);
        LED_SetState(LED_FAULT, LED_ON);
        break;
        
    default:
        break;
    }
}

/**
 * @brief 상태 머신 실행 (주기적 호출)
 */
void bms_run_state_machine(void)
{
    // 측정값 업데이트
    if (bms_update_measurements() != HAL_OK) {
        g_bms.comm_fault_count++;
        if (g_bms.comm_fault_count > 5) {
            bms_change_state(BMS_STATE_FAULT_COMM);
            return;
        }
    }
    
    // 상태별 처리
    switch (g_bms.state) {
    case BMS_STATE_INIT:
        bms_state_init();
        break;
    case BMS_STATE_IDLE:
        bms_state_idle();
        break;
    case BMS_STATE_CHARGING:
        bms_state_charging();
        break;
    case BMS_STATE_DISCHARGING:
        bms_state_discharging();
        break;
    case BMS_STATE_BALANCING:
        bms_state_balancing();
        break;
    case BMS_STATE_FAULT_OV:
    case BMS_STATE_FAULT_UV:
    case BMS_STATE_FAULT_OT:
    case BMS_STATE_FAULT_UT:
        bms_state_fault();
        break;
    case BMS_STATE_FAULT_COMM:
        bms_state_fault_comm();
        break;
    default:
        break;
    }
}
```

## 각 상태 처리

```c
/**
 * @brief IDLE 상태 처리
 */
void bms_state_idle(void)
{
    // 폴트 체크
    if (bms_check_overvoltage()) {
        bms_change_state(BMS_STATE_FAULT_OV);
        return;
    }
    if (bms_check_undervoltage()) {
        bms_change_state(BMS_STATE_FAULT_UV);
        return;
    }
    if (bms_check_overtemp()) {
        bms_change_state(BMS_STATE_FAULT_OT);
        return;
    }
    
    // 충전기 감지
    if (bms_is_charger_connected()) {
        bms_change_state(BMS_STATE_CHARGING);
        return;
    }
    
    // 부하 감지
    if (bms_is_load_connected()) {
        bms_change_state(BMS_STATE_DISCHARGING);
        return;
    }
    
    // 밸런싱 필요 여부
    if (bms_need_balancing()) {
        bms_change_state(BMS_STATE_BALANCING);
        return;
    }
}

/**
 * @brief 충전 상태 처리
 */
void bms_state_charging(void)
{
    // 과전압 체크 (즉시 중단)
    if (bms_check_overvoltage()) {
        bms_change_state(BMS_STATE_FAULT_OV);
        return;
    }
    
    // 과온도 체크
    if (bms_check_overtemp_charge()) {
        bms_change_state(BMS_STATE_FAULT_OT);
        return;
    }
    
    // 충전 완료 체크
    if (g_bms.min_cell_voltage >= g_bms.config.cell_ov_recover) {
        printf("Charging complete: min=%.3fV\n", g_bms.min_cell_voltage);
        bms_set_charge_fet(false);
        bms_change_state(BMS_STATE_IDLE);
        return;
    }
    
    // 충전기 분리
    if (!bms_is_charger_connected()) {
        bms_set_charge_fet(false);
        bms_change_state(BMS_STATE_IDLE);
        return;
    }
    
    // 충전 중 밸런싱 (Top Balancing)
    if (g_bms.max_cell_voltage > 3.40f && g_bms.delta_voltage > 20.0f) {
        bms_update_balance_bitmap();
        bms_apply_balance();
    }
}

/**
 * @brief 방전 상태 처리
 */
void bms_state_discharging(void)
{
    // 저전압 체크 (즉시 중단)
    if (bms_check_undervoltage()) {
        bms_change_state(BMS_STATE_FAULT_UV);
        return;
    }
    
    // 과온도 체크
    if (bms_check_overtemp_discharge()) {
        bms_change_state(BMS_STATE_FAULT_OT);
        return;
    }
    
    // 부하 분리
    if (!bms_is_load_connected()) {
        bms_set_discharge_fet(false);
        bms_change_state(BMS_STATE_IDLE);
        return;
    }
}

/**
 * @brief 밸런싱 상태 처리
 */
void bms_state_balancing(void)
{
    // 폴트 체크
    if (bms_check_any_fault()) {
        bms_stop_balance();
        return;
    }
    
    // 밸런싱 업데이트
    bms_update_balance_bitmap();
    
    // 밸런싱 완료 체크
    if (g_bms.delta_voltage < g_bms.config.balance_stop_delta) {
        printf("Balancing complete: delta=%.1fmV\n", g_bms.delta_voltage);
        bms_stop_balance();
        bms_change_state(BMS_STATE_IDLE);
        return;
    }
    
    // 외부 이벤트 체크
    if (bms_is_charger_connected()) {
        bms_change_state(BMS_STATE_CHARGING);
        return;
    }
    if (bms_is_load_connected()) {
        bms_stop_balance();
        bms_change_state(BMS_STATE_DISCHARGING);
        return;
    }
    
    bms_apply_balance();
}

/**
 * @brief 폴트 상태 처리
 */
void bms_state_fault(void)
{
    uint32_t elapsed = HAL_GetTick() - g_bms.state_enter_time;
    
    // 복구 지연 대기
    if (elapsed < g_bms.config.recover_delay) {
        return;
    }
    
    // 복구 조건 체크
    bool can_recover = false;
    
    switch (g_bms.state) {
    case BMS_STATE_FAULT_OV:
        can_recover = (g_bms.max_cell_voltage < g_bms.config.cell_ov_recover);
        break;
    case BMS_STATE_FAULT_UV:
        can_recover = (g_bms.min_cell_voltage > g_bms.config.cell_uv_recover);
        break;
    case BMS_STATE_FAULT_OT:
    case BMS_STATE_FAULT_UT:
        can_recover = bms_check_temp_recovered();
        break;
    default:
        break;
    }
    
    if (can_recover) {
        printf("Fault recovered after %lu ms\n", elapsed);
        bms_change_state(BMS_STATE_IDLE);
    }
}
```

## 보호 체크 함수

```c
/**
 * @brief 과전압 체크
 */
bool bms_check_overvoltage(void)
{
    if (g_bms.max_cell_voltage > g_bms.config.cell_ov_threshold) {
        g_bms.ov_fault_count++;
        if (g_bms.ov_fault_count >= 3) {  // 3회 연속
            printf("🔴 OV: Cell %d = %.3fV\n", 
                   g_bms.max_cell_index + 1, g_bms.max_cell_voltage);
            return true;
        }
    } else {
        g_bms.ov_fault_count = 0;
    }
    return false;
}

/**
 * @brief 저전압 체크
 */
bool bms_check_undervoltage(void)
{
    if (g_bms.min_cell_voltage < g_bms.config.cell_uv_threshold) {
        g_bms.uv_fault_count++;
        if (g_bms.uv_fault_count >= 3) {
            printf("🟡 UV: Cell %d = %.3fV\n",
                   g_bms.min_cell_index + 1, g_bms.min_cell_voltage);
            return true;
        }
    } else {
        g_bms.uv_fault_count = 0;
    }
    return false;
}

/**
 * @brief 밸런싱 필요 여부
 */
bool bms_need_balancing(void)
{
    // 조건 1: 편차가 임계값 이상
    if (g_bms.delta_voltage < g_bms.config.balance_start_delta) {
        return false;
    }
    
    // 조건 2: 최소 전압 이상
    if (g_bms.min_cell_voltage < g_bms.config.balance_min_voltage) {
        return false;
    }
    
    return true;
}
```

## FET 제어

```c
/**
 * @brief 충전 FET 제어
 */
void bms_set_charge_fet(bool on)
{
    if (on && !bms_is_charge_allowed()) {
        printf("Charge not allowed!\n");
        return;
    }
    
    HAL_GPIO_WritePin(CHG_FET_GPIO, CHG_FET_PIN, 
                      on ? GPIO_PIN_SET : GPIO_PIN_RESET);
    g_bms.chg_fet_on = on;
    printf("CHG FET: %s\n", on ? "ON" : "OFF");
}

/**
 * @brief 방전 FET 제어
 */
void bms_set_discharge_fet(bool on)
{
    if (on && !bms_is_discharge_allowed()) {
        printf("Discharge not allowed!\n");
        return;
    }
    
    HAL_GPIO_WritePin(DSG_FET_GPIO, DSG_FET_PIN,
                      on ? GPIO_PIN_SET : GPIO_PIN_RESET);
    g_bms.dsg_fet_on = on;
    printf("DSG FET: %s\n", on ? "ON" : "OFF");
}

/**
 * @brief 충전 허용 여부
 */
bool bms_is_charge_allowed(void)
{
    // 폴트 상태면 불가
    if (g_bms.state >= BMS_STATE_FAULT_OV) {
        return false;
    }
    // 과전압이면 불가
    if (g_bms.max_cell_voltage >= g_bms.config.cell_ov_threshold) {
        return false;
    }
    // 저온이면 불가
    for (int i = 0; i < g_bms.num_devices; i++) {
        if (g_bms.temperatures[i] < g_bms.config.temp_ut_charge) {
            return false;
        }
    }
    return true;
}
```

## 메인 루프 통합

```c
int main(void)
{
    HAL_Init();
    SystemClock_Config();
    MX_GPIO_Init();
    MX_SPI1_Init();
    
    // AD7280A 초기화
    ad7280a_handle_t ad7280a;
    ad7280a_init(&ad7280a, &hspi1, 4);  // 4 devices
    
    // BMS 초기화
    bms_init(&ad7280a, 4);
    
    printf("\n=== BMS Started ===\n");
    printf("Cells: %d, Config: LFP\n", g_bms.num_cells);
    printf("OV: %.2fV, UV: %.2fV\n", 
           g_bms.config.cell_ov_threshold,
           g_bms.config.cell_uv_threshold);
    
    uint32_t last_run = 0;
    uint32_t last_print = 0;
    
    while (1) {
        uint32_t now = HAL_GetTick();
        
        // ALERT 인터럽트 처리
        if (g_alert_flag) {
            g_alert_flag = false;
            printf("⚡ ALERT!\n");
            bms_handle_alert_interrupt();
        }
        
        // 100ms 주기 상태 머신
        if (now - last_run >= 100) {
            last_run = now;
            bms_run_state_machine();
        }
        
        // 1초 주기 상태 출력
        if (now - last_print >= 1000) {
            last_print = now;
            bms_print_status();
        }
        
        // 기타 태스크
        bms_process_can_messages();
    }
}
```

## 상태 출력

```c
void bms_print_status(void)
{
    printf("\n[%s] %.1fV | ", bms_state_str[g_bms.state], g_bms.total_voltage);
    printf("Min:%.3f(C%d) Max:%.3f(C%d) Δ:%.0fmV | ",
           g_bms.min_cell_voltage, g_bms.min_cell_index + 1,
           g_bms.max_cell_voltage, g_bms.max_cell_index + 1,
           g_bms.delta_voltage);
    printf("CHG:%s DSG:%s BAL:%s\n",
           g_bms.chg_fet_on ? "ON" : "OFF",
           g_bms.dsg_fet_on ? "ON" : "OFF",
           g_bms.balancing_active ? "ON" : "OFF");
}
```

출력 예시:
```
[CHARGING] 76.8V | Min:3.195(C12) Max:3.215(C5) Δ:20mV | CHG:ON DSG:OFF BAL:OFF
[CHARGING] 76.9V | Min:3.198(C12) Max:3.218(C5) Δ:20mV | CHG:ON DSG:OFF BAL:OFF
[FAULT_OV] 77.2V | Min:3.210(C12) Max:3.658(C5) Δ:448mV | CHG:OFF DSG:OFF BAL:OFF
🔴 OV: Cell 5 = 3.658V
```

## 정리

| 구성요소 | 역할 |
|----------|------|
| 상태 머신 | 전체 동작 흐름 제어 |
| 보호 체크 | OV/UV/OT/UT 판정 |
| FET 제어 | 충전/방전 차단 |
| 밸런싱 | 셀 균등화 |
| ALERT 처리 | 즉각 대응 |

**Part 5 완료!** 🎉

---

## 시리즈 목차

**Part 5: 알람 & 보호** ✅
- [#13 - 과전압/저전압 알람 설정](/posts/bms/ad7280a-bms-dev-13/)
- [#14 - Alert 핀 인터럽트 처리](/posts/bms/ad7280a-bms-dev-14/)
- [#15 - 알람 상태 읽기](/posts/bms/ad7280a-bms-dev-15/)
- **#16 - 보호 로직 통합** ← 현재 글

**다음 Part 6: 통신 & 진단**
- #17 - CAN 통신 프로토콜
- #18 - SOC 추정 기초
- #19 - 데이터 로깅
- #20 - 진단 인터페이스

---

## 참고 자료

- [AD7280A Datasheet](https://www.analog.com/media/en/technical-documentation/data-sheets/AD7280A.pdf)
- [Battery Protection IC Design Guide](https://www.ti.com/lit/an/slva654/slva654.pdf)
- [State Machine Design for Embedded Systems](https://www.embedded.com/design-patterns-for-state-machines/)
