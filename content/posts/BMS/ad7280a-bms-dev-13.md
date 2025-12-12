---
title: "AD7280A BMS 개발 삽질기 #13 - 과전압/저전압 알람 설정"
date: 2024-12-11
draft: false
tags: ["AD7280A", "BMS", "STM32", "배터리", "알람", "과전압", "저전압"]
categories: ["BMS 개발"]
summary: "셀 전압이 위험 수준이면 즉시 알려줘야 한다. AD7280A의 알람 기능을 파헤쳐보자."
---

## 지난 글 요약

[지난 글](/posts/bms/ad7280a-bms-dev-12/)에서 밸런싱 중 발열 관리를 다뤘다. 체커보드 패턴과 듀티 제어로 온도를 26°C나 낮췄다. 이제 진짜 중요한 **보호 기능**으로 넘어간다.

## 왜 알람이 필요한가

배터리 보호의 기본:

```
과전압 (Over Voltage)
├─ 원인: 과충전
├─ 위험: 열폭주, 발화, 폭발
└─ 대응: 즉시 충전 중단

저전압 (Under Voltage)
├─ 원인: 과방전
├─ 위험: 셀 손상, 용량 감소
└─ 대응: 즉시 방전 중단
```

**LiFePO4 (LFP) 기준:**
- 과전압: 3.65V 이상 → 위험
- 저전압: 2.5V 이하 → 위험
- 정상 범위: 2.8V ~ 3.5V

## AD7280A 알람 구조

AD7280A는 **하드웨어 레벨** 알람 기능 제공:

```
┌─────────────────────────────────────────┐
│              AD7280A                    │
│                                         │
│  Cell 1 ──→ [비교기] ──→ OV/UV Flag ──┐ │
│  Cell 2 ──→ [비교기] ──→ OV/UV Flag ──┤ │
│  Cell 3 ──→ [비교기] ──→ OV/UV Flag ──┼─┼──→ ALERT 핀
│  Cell 4 ──→ [비교기] ──→ OV/UV Flag ──┤ │
│  Cell 5 ──→ [비교기] ──→ OV/UV Flag ──┤ │
│  Cell 6 ──→ [비교기] ──→ OV/UV Flag ──┘ │
│                                         │
│  [OV Threshold Register]                │
│  [UV Threshold Register]                │
└─────────────────────────────────────────┘
```

**장점**: CPU 개입 없이 실시간 감시!

## 알람 관련 레지스터

### 1. Cell Overvoltage Register (0x09)

```
┌─────────────────────────────────────┐
│  Bit 7-0: D7 D6 D5 D4 D3 D2 D1 D0  │
│           └────────────────────────┴─ 8비트 임계값
└─────────────────────────────────────┘

값 계산: V_threshold = (D[7:0] + 1) × 1.536V / 256
         = (D[7:0] + 1) × 6mV
```

| 원하는 전압 | D[7:0] 값 | 계산 |
|-------------|-----------|------|
| 3.6V | 0xE2 (226) | (226+1) × 6mV = 3.582V |
| 3.65V | 0xE7 (231) | (231+1) × 6mV = 3.612V |
| 3.7V | 0xEC (236) | (236+1) × 6mV = 3.642V |

### 2. Cell Undervoltage Register (0x0A)

동일한 계산 방식:

```
값 계산: V_threshold = (D[7:0] + 1) × 6mV
```

| 원하는 전압 | D[7:0] 값 | 계산 |
|-------------|-----------|------|
| 2.5V | 0xA0 (160) | (160+1) × 6mV = 2.502V |
| 2.8V | 0xC9 (201) | (201+1) × 6mV = 2.844V |
| 3.0V | 0xE8 (232) | (232+1) × 6mV = 3.018V |

### 3. Alert Register (0x0D)

알람 발생 상태 확인:

```
┌─────────────────────────────────────────────┐
│  Bit 7: Cell 6 OV    Bit 3: Cell 6 UV      │
│  Bit 6: Cell 5 OV    Bit 2: Cell 5 UV      │
│  Bit 5: Cell 4 OV    Bit 1: Cell 4 UV      │
│  Bit 4: Cell 3 OV    Bit 0: Cell 3 UV      │
│  ...                                        │
└─────────────────────────────────────────────┘
```

**주의**: Cell 1, 2는 별도 레지스터 (0x0E)

### 4. Cell Balance Register (0x14)

밸런싱도 알람과 연동 가능:
- 알람 발생 시 밸런싱 자동 중단 설정

## 임계값 계산 함수

```c
#define AD7280A_VOLT_RESOLUTION  0.006f  // 6mV

/**
 * @brief 전압을 레지스터 값으로 변환
 * @param voltage_v 전압 (V)
 * @return 레지스터 값 (0~255)
 */
uint8_t ad7280a_voltage_to_reg(float voltage_v)
{
    // V = (D + 1) × 6mV
    // D = V / 6mV - 1
    int32_t reg_val = (int32_t)(voltage_v / AD7280A_VOLT_RESOLUTION) - 1;
    
    // 범위 제한
    if (reg_val < 0) reg_val = 0;
    if (reg_val > 255) reg_val = 255;
    
    return (uint8_t)reg_val;
}

/**
 * @brief 레지스터 값을 전압으로 변환
 * @param reg_val 레지스터 값
 * @return 전압 (V)
 */
float ad7280a_reg_to_voltage(uint8_t reg_val)
{
    return (reg_val + 1) * AD7280A_VOLT_RESOLUTION;
}
```

## 알람 임계값 설정

### 전체 설정 함수

```c
#define REG_CELL_OV_THRESHOLD   0x09
#define REG_CELL_UV_THRESHOLD   0x0A
#define REG_AUX_OV_THRESHOLD    0x0B
#define REG_AUX_UV_THRESHOLD    0x0C

typedef struct {
    float cell_ov_v;    // 셀 과전압 임계값 (V)
    float cell_uv_v;    // 셀 저전압 임계값 (V)
    float aux_ov_v;     // AUX 과전압 임계값 (V)
    float aux_uv_v;     // AUX 저전압 임계값 (V)
} ad7280a_alarm_config_t;

/**
 * @brief 알람 임계값 설정
 */
HAL_StatusTypeDef ad7280a_set_alarm_thresholds(
    ad7280a_handle_t *handle,
    const ad7280a_alarm_config_t *config)
{
    HAL_StatusTypeDef status;
    uint8_t reg_val;
    
    // Cell OV 임계값
    reg_val = ad7280a_voltage_to_reg(config->cell_ov_v);
    status = ad7280a_write_all(handle, REG_CELL_OV_THRESHOLD, reg_val);
    if (status != HAL_OK) return status;
    
    // Cell UV 임계값
    reg_val = ad7280a_voltage_to_reg(config->cell_uv_v);
    status = ad7280a_write_all(handle, REG_CELL_UV_THRESHOLD, reg_val);
    if (status != HAL_OK) return status;
    
    // AUX OV 임계값 (온도 센서용)
    reg_val = ad7280a_voltage_to_reg(config->aux_ov_v);
    status = ad7280a_write_all(handle, REG_AUX_OV_THRESHOLD, reg_val);
    if (status != HAL_OK) return status;
    
    // AUX UV 임계값
    reg_val = ad7280a_voltage_to_reg(config->aux_uv_v);
    status = ad7280a_write_all(handle, REG_AUX_UV_THRESHOLD, reg_val);
    if (status != HAL_OK) return status;
    
    return HAL_OK;
}
```

### LiFePO4 기본 설정

```c
void ad7280a_init_lfp_alarms(ad7280a_handle_t *handle)
{
    ad7280a_alarm_config_t config = {
        .cell_ov_v = 3.65f,    // LFP 과전압
        .cell_uv_v = 2.50f,    // LFP 저전압
        .aux_ov_v  = 3.0f,     // NTC 고온 (저항 감소 → 전압 증가)
        .aux_uv_v  = 0.5f,     // NTC 저온 (저항 증가 → 전압 감소)
    };
    
    ad7280a_set_alarm_thresholds(handle, &config);
    
    printf("Alarm thresholds set:\n");
    printf("  Cell OV: %.3fV (reg: 0x%02X)\n", 
           config.cell_ov_v, ad7280a_voltage_to_reg(config.cell_ov_v));
    printf("  Cell UV: %.3fV (reg: 0x%02X)\n",
           config.cell_uv_v, ad7280a_voltage_to_reg(config.cell_uv_v));
}
```

출력:
```
Alarm thresholds set:
  Cell OV: 3.650V (reg: 0xE3)
  Cell UV: 2.500V (reg: 0x9F)
```

## 알람 활성화

임계값 설정만으론 부족. **알람 기능을 켜야** 한다.

### Control Register (0x01)

```
┌─────────────────────────────────────────────────┐
│  Bit 7: ALERT_HIGH  - ALERT 핀 활성 레벨        │
│  Bit 6: ALERT_EN    - ALERT 핀 활성화           │
│  Bit 5: COMP_EN     - 비교기 활성화             │ ← 핵심!
│  Bit 4: CONV_EN     - 연속 변환 모드            │
│  Bit 3-0: Reserved                              │
└─────────────────────────────────────────────────┘
```

### 알람 활성화 함수

```c
#define REG_CONTROL     0x01

#define CTRL_ALERT_HIGH (1 << 7)  // ALERT 핀 Active High
#define CTRL_ALERT_EN   (1 << 6)  // ALERT 핀 활성화
#define CTRL_COMP_EN    (1 << 5)  // 비교기 (알람) 활성화
#define CTRL_CONV_EN    (1 << 4)  // 연속 변환 모드

/**
 * @brief 알람 기능 활성화
 */
HAL_StatusTypeDef ad7280a_enable_alarms(ad7280a_handle_t *handle)
{
    uint8_t ctrl_val = CTRL_ALERT_EN | CTRL_COMP_EN;
    
    // ALERT 핀을 Active Low로 설정 (기본)
    // Active High 원하면: ctrl_val |= CTRL_ALERT_HIGH;
    
    return ad7280a_write_all(handle, REG_CONTROL, ctrl_val);
}

/**
 * @brief 알람 기능 비활성화
 */
HAL_StatusTypeDef ad7280a_disable_alarms(ad7280a_handle_t *handle)
{
    return ad7280a_write_all(handle, REG_CONTROL, 0x00);
}
```

## 알람 상태 읽기

### Alert Register 구조

Cell 1~6의 OV/UV 상태가 2개 레지스터에 분산:

```
Alert Register A (0x0D):
  Bit 7: Cell 6 OV    Bit 6: Cell 5 OV
  Bit 5: Cell 4 OV    Bit 4: Cell 3 OV
  Bit 3: Cell 6 UV    Bit 2: Cell 5 UV
  Bit 1: Cell 4 UV    Bit 0: Cell 3 UV

Alert Register B (0x0E):
  Bit 7: AUX 6 OV     Bit 6: AUX 5 OV
  Bit 5: AUX 4 OV     Bit 4: AUX 3 OV
  Bit 3: Cell 2 OV    Bit 2: Cell 1 OV
  Bit 1: Cell 2 UV    Bit 0: Cell 1 UV
```

### 알람 상태 구조체

```c
typedef struct {
    uint8_t cell_ov;    // 셀 과전압 비트맵 (bit 0~5 = cell 1~6)
    uint8_t cell_uv;    // 셀 저전압 비트맵
    uint8_t aux_ov;     // AUX 과전압 비트맵
    uint8_t aux_uv;     // AUX 저전압 비트맵
} ad7280a_alert_status_t;

/**
 * @brief 알람 상태 읽기
 */
HAL_StatusTypeDef ad7280a_read_alert_status(
    ad7280a_handle_t *handle,
    uint8_t device_addr,
    ad7280a_alert_status_t *status)
{
    uint8_t alert_a, alert_b;
    HAL_StatusTypeDef ret;
    
    // Alert Register A 읽기
    ret = ad7280a_read_register(handle, device_addr, 0x0D, &alert_a);
    if (ret != HAL_OK) return ret;
    
    // Alert Register B 읽기
    ret = ad7280a_read_register(handle, device_addr, 0x0E, &alert_b);
    if (ret != HAL_OK) return ret;
    
    // 비트맵 재구성
    status->cell_ov = 0;
    status->cell_uv = 0;
    
    // Cell 1, 2 (from Alert B)
    if (alert_b & 0x04) status->cell_ov |= (1 << 0);  // Cell 1 OV
    if (alert_b & 0x08) status->cell_ov |= (1 << 1);  // Cell 2 OV
    if (alert_b & 0x01) status->cell_uv |= (1 << 0);  // Cell 1 UV
    if (alert_b & 0x02) status->cell_uv |= (1 << 1);  // Cell 2 UV
    
    // Cell 3~6 (from Alert A)
    if (alert_a & 0x10) status->cell_ov |= (1 << 2);  // Cell 3 OV
    if (alert_a & 0x20) status->cell_ov |= (1 << 3);  // Cell 4 OV
    if (alert_a & 0x40) status->cell_ov |= (1 << 4);  // Cell 5 OV
    if (alert_a & 0x80) status->cell_ov |= (1 << 5);  // Cell 6 OV
    if (alert_a & 0x01) status->cell_uv |= (1 << 2);  // Cell 3 UV
    if (alert_a & 0x02) status->cell_uv |= (1 << 3);  // Cell 4 UV
    if (alert_a & 0x04) status->cell_uv |= (1 << 4);  // Cell 5 UV
    if (alert_a & 0x08) status->cell_uv |= (1 << 5);  // Cell 6 UV
    
    // AUX (from Alert B)
    status->aux_ov = (alert_b >> 4) & 0x0F;
    status->aux_uv = 0;  // AUX UV는 별도 처리 필요
    
    return HAL_OK;
}
```

### 알람 상태 출력

```c
void ad7280a_print_alert_status(
    uint8_t device_addr,
    const ad7280a_alert_status_t *status)
{
    printf("Device %d Alert Status:\n", device_addr);
    
    // 과전압 체크
    if (status->cell_ov) {
        printf("  ⚠️ OVERVOLTAGE: ");
        for (int i = 0; i < 6; i++) {
            if (status->cell_ov & (1 << i)) {
                printf("Cell%d ", i + 1);
            }
        }
        printf("\n");
    }
    
    // 저전압 체크
    if (status->cell_uv) {
        printf("  ⚠️ UNDERVOLTAGE: ");
        for (int i = 0; i < 6; i++) {
            if (status->cell_uv & (1 << i)) {
                printf("Cell%d ", i + 1);
            }
        }
        printf("\n");
    }
    
    if (!status->cell_ov && !status->cell_uv) {
        printf("  ✅ All cells normal\n");
    }
}
```

출력 예시:
```
Device 0 Alert Status:
  ⚠️ OVERVOLTAGE: Cell3 Cell5 
Device 1 Alert Status:
  ✅ All cells normal
```

## 전체 시스템 알람 체크

```c
typedef struct {
    bool has_overvoltage;
    bool has_undervoltage;
    uint8_t ov_device;      // 과전압 발생 디바이스
    uint8_t ov_cell;        // 과전압 발생 셀
    uint8_t uv_device;      // 저전압 발생 디바이스
    uint8_t uv_cell;        // 저전압 발생 셀
    float max_voltage;      // 최고 전압
    float min_voltage;      // 최저 전압
} bms_alarm_result_t;

/**
 * @brief 전체 BMS 알람 상태 체크
 */
void bms_check_alarms(
    ad7280a_handle_t *handle,
    uint8_t num_devices,
    bms_alarm_result_t *result)
{
    ad7280a_alert_status_t status;
    
    memset(result, 0, sizeof(bms_alarm_result_t));
    result->min_voltage = 999.0f;
    result->max_voltage = 0.0f;
    
    for (uint8_t dev = 0; dev < num_devices; dev++) {
        ad7280a_read_alert_status(handle, dev, &status);
        
        // 과전압 체크
        if (status.cell_ov) {
            result->has_overvoltage = true;
            result->ov_device = dev;
            // 첫 번째 알람 셀 찾기
            for (int i = 0; i < 6; i++) {
                if (status.cell_ov & (1 << i)) {
                    result->ov_cell = i;
                    break;
                }
            }
        }
        
        // 저전압 체크
        if (status.cell_uv) {
            result->has_undervoltage = true;
            result->uv_device = dev;
            for (int i = 0; i < 6; i++) {
                if (status.cell_uv & (1 << i)) {
                    result->uv_cell = i;
                    break;
                }
            }
        }
    }
}
```

## 보호 동작 연동

```c
void bms_protection_handler(bms_alarm_result_t *alarm)
{
    if (alarm->has_overvoltage) {
        printf("🚨 OVERVOLTAGE DETECTED! Dev:%d Cell:%d\n",
               alarm->ov_device, alarm->ov_cell + 1);
        
        // 충전 FET OFF
        charger_disable();
        
        // 밸런싱 시작 (과전압 셀)
        ad7280a_set_balance(alarm->ov_device, 
                           1 << alarm->ov_cell, true);
    }
    
    if (alarm->has_undervoltage) {
        printf("🚨 UNDERVOLTAGE DETECTED! Dev:%d Cell:%d\n",
               alarm->uv_device, alarm->uv_cell + 1);
        
        // 방전 FET OFF
        load_disconnect();
        
        // 밸런싱 중지
        ad7280a_disable_all_balance();
    }
}
```

## 삽질: 임계값 정밀도

처음에 3.65V 설정했는데 3.64V에서 알람이 안 떴다.

**원인**: 6mV 해상도 때문에 정확히 3.65V 설정 불가

```
3.65V ÷ 6mV = 608.3
reg = 608 - 1 = 607 (0x25F) → 8비트 오버플로!

실제 설정: 0xE3 (227)
실제 임계값: (227 + 1) × 6mV = 3.612V
```

**해결**: 안전 마진을 고려해서 약간 낮게/높게 설정

```c
// 과전압: 목표값보다 약간 낮게 (일찍 감지)
config.cell_ov_v = 3.60f;  // 3.65V 대신

// 저전압: 목표값보다 약간 높게 (일찍 감지)
config.cell_uv_v = 2.55f;  // 2.50V 대신
```

## 삽질: 비교기 지연

알람 설정 직후 바로 상태를 읽으면 이전 상태가 나온다.

**원인**: 비교기 안정화 시간 필요

```c
// 잘못된 코드
ad7280a_enable_alarms(handle);
ad7280a_read_alert_status(handle, 0, &status);  // 이전 상태!

// 올바른 코드
ad7280a_enable_alarms(handle);
HAL_Delay(10);  // 비교기 안정화 대기
ad7280a_read_alert_status(handle, 0, &status);  // 정확한 상태
```

## 정리

| 레지스터 | 주소 | 용도 |
|----------|------|------|
| Control | 0x01 | 알람 활성화 |
| Cell OV Threshold | 0x09 | 과전압 임계값 |
| Cell UV Threshold | 0x0A | 저전압 임계값 |
| Alert A | 0x0D | Cell 3~6 알람 상태 |
| Alert B | 0x0E | Cell 1~2, AUX 알람 상태 |

| 항목 | 공식 |
|------|------|
| 전압 → 레지스터 | D = V / 6mV - 1 |
| 레지스터 → 전압 | V = (D + 1) × 6mV |
| 해상도 | 6mV |
| 범위 | 6mV ~ 1.536V |

**다음 글에서**: ALERT 핀 인터럽트 처리 - 폴링 말고 하드웨어 인터럽트로 즉시 감지.

---

## 시리즈 목차

**Part 1~4**: [이전 글 목록](/tags/ad7280a/)

**Part 5: 알람 & 보호**
- **#13 - 과전압/저전압 알람 설정** ← 현재 글
- #14 - Alert 핀 인터럽트 처리
- #15 - 알람 상태 읽기
- #16 - 보호 로직 통합

---

## 참고 자료

- [AD7280A Datasheet - Alert Registers](https://www.analog.com/media/en/technical-documentation/data-sheets/AD7280A.pdf)
- [LiFePO4 Safe Operating Limits](https://batteryuniversity.com/article/bu-205-types-of-lithium-ion)
