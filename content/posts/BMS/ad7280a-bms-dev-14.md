---
title: "AD7280A BMS 개발 삽질기 #14 - Alert 핀 인터럽트 처리"
date: 2024-12-11
draft: false
tags: ["AD7280A", "BMS", "STM32", "배터리", "인터럽트", "ALERT", "EXTI"]
categories: ["BMS 개발"]
summary: "폴링으로 알람 체크? 느려터졌다. ALERT 핀으로 즉시 감지하자."
---

## 지난 글 요약

[지난 글](/posts/bms/ad7280a-bms-dev-13/)에서 과전압/저전압 임계값을 설정했다. 이제 알람이 발생하면 **즉시** 감지해야 한다.

## 폴링 vs 인터럽트

### 폴링 방식 (느림)

```c
while (1) {
    ad7280a_read_alert_status(&status);  // 100ms마다
    if (status.cell_ov || status.cell_uv) {
        handle_alarm();
    }
    HAL_Delay(100);
}
```

**문제**: 최악의 경우 100ms 동안 과전압 방치!

### 인터럽트 방식 (즉시)

```c
void EXTI9_5_IRQHandler(void) {
    if (alert_pin_triggered) {
        handle_alarm();  // 즉시 처리!
    }
}
```

**장점**: μs 단위 응답. 배터리 보호에 필수.

## AD7280A ALERT 핀

### 데이지체인에서 ALERT 동작

```
┌─────────┐    ┌─────────┐    ┌─────────┐
│ AD7280A │    │ AD7280A │    │ AD7280A │
│  Dev 0  │    │  Dev 1  │    │  Dev 2  │
│         │    │         │    │         │
│  ALERT ─┼────┼─ ALERT ─┼────┼─ ALERT ─┼───→ MCU GPIO
└─────────┘    └─────────┘    └─────────┘
                    
        Wired-OR (Open Drain)
```

**어느 디바이스든** 알람 발생 → ALERT 핀 LOW

### ALERT 핀 특성

| 항목 | 값 |
|------|-----|
| 타입 | Open Drain |
| Active Level | Low (기본) |
| 풀업 필요 | 예 (10KΩ 권장) |
| 다중 디바이스 | Wired-OR |

## 하드웨어 연결

```
AD7280A ALERT ────┬──── 10KΩ ──── VDD (3.3V)
                  │
                  └──── STM32 PA8 (EXTI8)
```

**주의**: Open Drain이므로 **반드시 풀업** 필요!

## STM32 EXTI 설정

### CubeMX 설정

```
PA8:
  - GPIO mode: External Interrupt Mode with Falling edge trigger
  - GPIO Pull-up/Pull-down: Pull-up
  - User Label: AD7280A_ALERT
```

### 코드 설정 (HAL)

```c
// GPIO 초기화
void MX_GPIO_Init(void)
{
    GPIO_InitTypeDef GPIO_InitStruct = {0};
    
    __HAL_RCC_GPIOA_CLK_ENABLE();
    
    // ALERT 핀 (PA8) - 외부 인터럽트
    GPIO_InitStruct.Pin = GPIO_PIN_8;
    GPIO_InitStruct.Mode = GPIO_MODE_IT_FALLING;  // 하강 에지
    GPIO_InitStruct.Pull = GPIO_PULLUP;
    HAL_GPIO_Init(GPIOA, &GPIO_InitStruct);
    
    // NVIC 설정
    HAL_NVIC_SetPriority(EXTI9_5_IRQn, 1, 0);  // 높은 우선순위!
    HAL_NVIC_EnableIRQ(EXTI9_5_IRQn);
}
```

### 인터럽트 핸들러

```c
// stm32f1xx_it.c
void EXTI9_5_IRQHandler(void)
{
    HAL_GPIO_EXTI_IRQHandler(GPIO_PIN_8);
}

// 콜백 함수
void HAL_GPIO_EXTI_Callback(uint16_t GPIO_Pin)
{
    if (GPIO_Pin == GPIO_PIN_8) {
        // ALERT 발생!
        g_alert_flag = true;
    }
}
```

## 알람 처리 구조

### 플래그 기반 처리

인터럽트에서 직접 처리하면 위험. **플래그만 설정**하고 메인 루프에서 처리.

```c
// 전역 변수
volatile bool g_alert_flag = false;
volatile uint32_t g_alert_timestamp = 0;

// 인터럽트 콜백
void HAL_GPIO_EXTI_Callback(uint16_t GPIO_Pin)
{
    if (GPIO_Pin == AD7280A_ALERT_PIN) {
        g_alert_flag = true;
        g_alert_timestamp = HAL_GetTick();
    }
}

// 메인 루프
void main_loop(void)
{
    while (1) {
        if (g_alert_flag) {
            g_alert_flag = false;
            bms_handle_alert();
        }
        
        // 다른 작업...
    }
}
```

### 즉시 차단이 필요한 경우

**과전압/저전압은 즉시 FET 차단**이 필요할 수 있다.

```c
void HAL_GPIO_EXTI_Callback(uint16_t GPIO_Pin)
{
    if (GPIO_Pin == AD7280A_ALERT_PIN) {
        // 즉시 안전 조치 (인터럽트 내에서)
        CHARGER_FET_OFF();
        DISCHARGE_FET_OFF();
        
        // 상세 처리는 메인 루프에서
        g_alert_flag = true;
    }
}
```

## 알람 처리 함수

```c
typedef enum {
    ALERT_NONE = 0,
    ALERT_CELL_OV,
    ALERT_CELL_UV,
    ALERT_AUX_OV,
    ALERT_AUX_UV,
    ALERT_COMM_ERROR
} alert_type_t;

typedef struct {
    alert_type_t type;
    uint8_t device;
    uint8_t cell;
    float voltage;
    uint32_t timestamp;
} alert_event_t;

#define ALERT_HISTORY_SIZE  16
alert_event_t g_alert_history[ALERT_HISTORY_SIZE];
uint8_t g_alert_history_idx = 0;

/**
 * @brief 알람 이벤트 기록
 */
void bms_log_alert(alert_type_t type, uint8_t dev, uint8_t cell, float voltage)
{
    alert_event_t *event = &g_alert_history[g_alert_history_idx];
    
    event->type = type;
    event->device = dev;
    event->cell = cell;
    event->voltage = voltage;
    event->timestamp = HAL_GetTick();
    
    g_alert_history_idx = (g_alert_history_idx + 1) % ALERT_HISTORY_SIZE;
    
    // 디버그 출력
    const char *type_str[] = {
        "NONE", "CELL_OV", "CELL_UV", "AUX_OV", "AUX_UV", "COMM_ERR"
    };
    printf("🚨 ALERT [%lu] %s: Dev%d Cell%d %.3fV\n",
           event->timestamp, type_str[type], dev, cell + 1, voltage);
}

/**
 * @brief ALERT 핀 트리거 시 호출
 */
void bms_handle_alert(void)
{
    ad7280a_alert_status_t status;
    float cell_voltages[6];
    
    printf("\n=== ALERT TRIGGERED ===\n");
    
    // 모든 디바이스 스캔
    for (uint8_t dev = 0; dev < g_num_devices; dev++) {
        // 알람 상태 읽기
        ad7280a_read_alert_status(&g_ad7280a, dev, &status);
        
        // 전압 읽기 (상세 정보용)
        ad7280a_read_cell_voltages(&g_ad7280a, dev, cell_voltages);
        
        // 과전압 체크
        for (int i = 0; i < 6; i++) {
            if (status.cell_ov & (1 << i)) {
                bms_log_alert(ALERT_CELL_OV, dev, i, cell_voltages[i]);
                bms_action_overvoltage(dev, i);
            }
        }
        
        // 저전압 체크
        for (int i = 0; i < 6; i++) {
            if (status.cell_uv & (1 << i)) {
                bms_log_alert(ALERT_CELL_UV, dev, i, cell_voltages[i]);
                bms_action_undervoltage(dev, i);
            }
        }
    }
    
    printf("=== ALERT HANDLED ===\n\n");
}
```

## 보호 동작

```c
/**
 * @brief 과전압 시 동작
 */
void bms_action_overvoltage(uint8_t device, uint8_t cell)
{
    // 1. 충전 중단
    charger_disable();
    g_bms_state.charging_allowed = false;
    
    // 2. 해당 셀 밸런싱 시작
    ad7280a_set_cell_balance(&g_ad7280a, device, cell, true);
    
    // 3. 경고등
    LED_SetState(LED_FAULT, LED_BLINK_FAST);
    
    // 4. CAN 알람 전송
    can_send_alarm(ALARM_OVERVOLTAGE, device, cell);
}

/**
 * @brief 저전압 시 동작
 */
void bms_action_undervoltage(uint8_t device, uint8_t cell)
{
    // 1. 방전 중단
    load_disconnect();
    g_bms_state.discharging_allowed = false;
    
    // 2. 모든 밸런싱 중지
    ad7280a_disable_all_balance(&g_ad7280a);
    
    // 3. 경고등
    LED_SetState(LED_FAULT, LED_BLINK_FAST);
    
    // 4. CAN 알람 전송
    can_send_alarm(ALARM_UNDERVOLTAGE, device, cell);
}
```

## 알람 클리어

알람 조건이 해소되면 클리어해야 한다.

```c
/**
 * @brief 알람 상태 클리어
 */
HAL_StatusTypeDef ad7280a_clear_alerts(ad7280a_handle_t *handle)
{
    // Alert Register 읽기 → 자동 클리어
    uint8_t dummy;
    
    for (uint8_t dev = 0; dev < handle->num_devices; dev++) {
        ad7280a_read_register(handle, dev, REG_ALERT_A, &dummy);
        ad7280a_read_register(handle, dev, REG_ALERT_B, &dummy);
    }
    
    return HAL_OK;
}

/**
 * @brief ALERT 핀 상태 확인
 */
bool ad7280a_is_alert_active(void)
{
    // Active Low이므로 LOW면 알람 활성
    return (HAL_GPIO_ReadPin(ALERT_GPIO_PORT, ALERT_GPIO_PIN) == GPIO_PIN_RESET);
}

/**
 * @brief 알람 복구 시도
 */
void bms_try_recover(void)
{
    // 알람 클리어
    ad7280a_clear_alerts(&g_ad7280a);
    
    // 잠시 대기
    HAL_Delay(100);
    
    // ALERT 핀 확인
    if (!ad7280a_is_alert_active()) {
        printf("✅ Alert cleared, system recovering...\n");
        
        // 상태 재확인 후 복구
        if (bms_check_all_cells_safe()) {
            g_bms_state.charging_allowed = true;
            g_bms_state.discharging_allowed = true;
            LED_SetState(LED_FAULT, LED_OFF);
        }
    } else {
        printf("⚠️ Alert still active, check battery!\n");
    }
}
```

## 디바운싱

노이즈로 인한 오동작 방지:

```c
#define ALERT_DEBOUNCE_MS  10

volatile uint32_t g_last_alert_time = 0;

void HAL_GPIO_EXTI_Callback(uint16_t GPIO_Pin)
{
    if (GPIO_Pin == AD7280A_ALERT_PIN) {
        uint32_t now = HAL_GetTick();
        
        // 디바운싱: 10ms 이내 재트리거 무시
        if (now - g_last_alert_time > ALERT_DEBOUNCE_MS) {
            g_alert_flag = true;
            g_last_alert_time = now;
        }
    }
}
```

## 삽질: 풀업 저항 없음

처음에 풀업 없이 테스트 → ALERT 핀이 플로팅 → 랜덤 인터럽트 폭탄

```
증상: 알람 없는데 계속 인터럽트 발생
원인: Open Drain + 풀업 없음 = 플로팅
해결: 10KΩ 풀업 추가
```

## 삽질: 인터럽트 우선순위

SPI 통신 중 ALERT 인터럽트 → SPI 깨짐

```c
// 잘못된 설정
HAL_NVIC_SetPriority(EXTI9_5_IRQn, 0, 0);  // 최고 우선순위
HAL_NVIC_SetPriority(SPI1_IRQn, 1, 0);

// 올바른 설정
HAL_NVIC_SetPriority(SPI1_IRQn, 0, 0);     // SPI가 더 높음
HAL_NVIC_SetPriority(EXTI9_5_IRQn, 1, 0);  // ALERT는 그 다음
```

## 정리

| 항목 | 설정 |
|------|------|
| ALERT 핀 | Open Drain, Active Low |
| 풀업 | 10KΩ to VDD |
| EXTI | Falling Edge |
| 우선순위 | SPI보다 낮게 |
| 디바운싱 | 10ms |
| 처리 방식 | 플래그 → 메인 루프 |

**다음 글에서**: 알람 상태 읽기 - 어떤 셀이 문제인지 정확히 파악하기.

---

## 시리즈 목차

**Part 5: 알람 & 보호**
- [#13 - 과전압/저전압 알람 설정](/posts/bms/ad7280a-bms-dev-13/)
- **#14 - Alert 핀 인터럽트 처리** ← 현재 글
- #15 - 알람 상태 읽기
- #16 - 보호 로직 통합

---

## 참고 자료

- [AD7280A Datasheet - ALERT Pin](https://www.analog.com/media/en/technical-documentation/data-sheets/AD7280A.pdf)
- [STM32 EXTI Configuration](https://www.st.com/resource/en/reference_manual/rm0008.pdf)
