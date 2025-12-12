---
title: "AD7280A BMS 개발 삽질기 #15 - 알람 상태 읽기"
date: 2024-12-11
draft: false
tags: ["AD7280A", "BMS", "STM32", "배터리", "알람", "레지스터"]
categories: ["BMS 개발"]
summary: "ALERT 떴다! 근데 어떤 셀이 문제야? 알람 레지스터를 해부해보자."
---

## 지난 글 요약

[지난 글](/posts/bms/ad7280a-bms-dev-14/)에서 ALERT 핀 인터럽트를 설정했다. 이제 알람이 발생하면 **정확히 어떤 셀**이 문제인지 파악해야 한다.

## 알람 레지스터 구조

AD7280A는 알람 정보를 **4개 레지스터**에 저장:

| 레지스터 | 주소 | 내용 |
|----------|------|------|
| Alert Register A | 0x0D | Cell 3~6 OV/UV |
| Alert Register B | 0x0E | Cell 1~2, AUX OV/UV |
| Alert Register C | 0x0F | 자가진단 결과 |
| Alert Register D | 0x10 | 추가 상태 |

## Alert Register A (0x0D)

Cell 3~6의 과전압/저전압 상태:

```
┌─────────────────────────────────────────────────────┐
│  Bit 7    Bit 6    Bit 5    Bit 4                  │
│  Cell6_OV Cell5_OV Cell4_OV Cell3_OV               │
├─────────────────────────────────────────────────────┤
│  Bit 3    Bit 2    Bit 1    Bit 0                  │
│  Cell6_UV Cell5_UV Cell4_UV Cell3_UV               │
└─────────────────────────────────────────────────────┘
```

예시:
```
Alert_A = 0x24 = 0b00100100
               = Cell5_OV + Cell4_UV
→ Cell 5 과전압, Cell 4 저전압
```

## Alert Register B (0x0E)

Cell 1~2와 AUX 채널:

```
┌─────────────────────────────────────────────────────┐
│  Bit 7    Bit 6    Bit 5    Bit 4                  │
│  AUX6_OV  AUX5_OV  AUX4_OV  AUX3_OV                │
├─────────────────────────────────────────────────────┤
│  Bit 3    Bit 2    Bit 1    Bit 0                  │
│  Cell2_OV Cell1_OV Cell2_UV Cell1_UV               │
└─────────────────────────────────────────────────────┘
```

**주의**: Cell 1,2는 OV/UV 비트 위치가 다름!

## Alert Register C (0x0F)

자가진단 결과:

```
┌─────────────────────────────────────────────────────┐
│  Bit 7    Bit 6    Bit 5    Bit 4                  │
│  Reserved Reserved Reserved Reserved               │
├─────────────────────────────────────────────────────┤
│  Bit 3    Bit 2    Bit 1    Bit 0                  │
│  AUX_SELF CELL_SELF CRC_ERR  CONV_ERR              │
└─────────────────────────────────────────────────────┘
```

| 비트 | 이름 | 의미 |
|------|------|------|
| 3 | AUX_SELF | AUX 자가진단 실패 |
| 2 | CELL_SELF | Cell 자가진단 실패 |
| 1 | CRC_ERR | CRC 오류 감지 |
| 0 | CONV_ERR | 변환 오류 |

## 통합 알람 읽기 함수

```c
typedef struct {
    // Cell 알람 (비트맵: bit0=Cell1, bit5=Cell6)
    uint8_t cell_ov;        // 과전압
    uint8_t cell_uv;        // 저전압
    
    // AUX 알람 (비트맵: bit0=AUX3, bit3=AUX6)
    uint8_t aux_ov;
    uint8_t aux_uv;
    
    // 자가진단
    bool aux_self_fail;
    bool cell_self_fail;
    bool crc_error;
    bool conv_error;
    
    // 원본 레지스터 값
    uint8_t raw_alert_a;
    uint8_t raw_alert_b;
    uint8_t raw_alert_c;
} ad7280a_full_alert_t;

/**
 * @brief 전체 알람 상태 읽기
 */
HAL_StatusTypeDef ad7280a_read_full_alert(
    ad7280a_handle_t *handle,
    uint8_t device_addr,
    ad7280a_full_alert_t *alert)
{
    HAL_StatusTypeDef ret;
    
    memset(alert, 0, sizeof(ad7280a_full_alert_t));
    
    // Alert Register A (0x0D) - Cell 3~6
    ret = ad7280a_read_register(handle, device_addr, 0x0D, &alert->raw_alert_a);
    if (ret != HAL_OK) return ret;
    
    // Alert Register B (0x0E) - Cell 1~2, AUX
    ret = ad7280a_read_register(handle, device_addr, 0x0E, &alert->raw_alert_b);
    if (ret != HAL_OK) return ret;
    
    // Alert Register C (0x0F) - 자가진단
    ret = ad7280a_read_register(handle, device_addr, 0x0F, &alert->raw_alert_c);
    if (ret != HAL_OK) return ret;
    
    // Cell OV 비트맵 재구성
    alert->cell_ov = 0;
    if (alert->raw_alert_b & 0x04) alert->cell_ov |= (1 << 0);  // Cell 1
    if (alert->raw_alert_b & 0x08) alert->cell_ov |= (1 << 1);  // Cell 2
    if (alert->raw_alert_a & 0x10) alert->cell_ov |= (1 << 2);  // Cell 3
    if (alert->raw_alert_a & 0x20) alert->cell_ov |= (1 << 3);  // Cell 4
    if (alert->raw_alert_a & 0x40) alert->cell_ov |= (1 << 4);  // Cell 5
    if (alert->raw_alert_a & 0x80) alert->cell_ov |= (1 << 5);  // Cell 6
    
    // Cell UV 비트맵 재구성
    alert->cell_uv = 0;
    if (alert->raw_alert_b & 0x01) alert->cell_uv |= (1 << 0);  // Cell 1
    if (alert->raw_alert_b & 0x02) alert->cell_uv |= (1 << 1);  // Cell 2
    if (alert->raw_alert_a & 0x01) alert->cell_uv |= (1 << 2);  // Cell 3
    if (alert->raw_alert_a & 0x02) alert->cell_uv |= (1 << 3);  // Cell 4
    if (alert->raw_alert_a & 0x04) alert->cell_uv |= (1 << 4);  // Cell 5
    if (alert->raw_alert_a & 0x08) alert->cell_uv |= (1 << 5);  // Cell 6
    
    // AUX OV (AUX3~6만 알람 지원)
    alert->aux_ov = (alert->raw_alert_b >> 4) & 0x0F;
    
    // 자가진단 결과
    alert->conv_error = (alert->raw_alert_c & 0x01) != 0;
    alert->crc_error = (alert->raw_alert_c & 0x02) != 0;
    alert->cell_self_fail = (alert->raw_alert_c & 0x04) != 0;
    alert->aux_self_fail = (alert->raw_alert_c & 0x08) != 0;
    
    return HAL_OK;
}
```

## 알람 상태 분석

```c
/**
 * @brief 알람 상태 분석 및 출력
 */
void ad7280a_analyze_alert(uint8_t device, const ad7280a_full_alert_t *alert)
{
    printf("\n=== Device %d Alert Analysis ===\n", device);
    printf("Raw: A=0x%02X B=0x%02X C=0x%02X\n",
           alert->raw_alert_a, alert->raw_alert_b, alert->raw_alert_c);
    
    // 과전압
    if (alert->cell_ov) {
        printf("🔴 OVERVOLTAGE: ");
        for (int i = 0; i < 6; i++) {
            if (alert->cell_ov & (1 << i)) {
                printf("Cell%d ", i + 1);
            }
        }
        printf("\n");
    }
    
    // 저전압
    if (alert->cell_uv) {
        printf("🟡 UNDERVOLTAGE: ");
        for (int i = 0; i < 6; i++) {
            if (alert->cell_uv & (1 << i)) {
                printf("Cell%d ", i + 1);
            }
        }
        printf("\n");
    }
    
    // AUX (온도) 알람
    if (alert->aux_ov) {
        printf("🟠 AUX OVERVOLTAGE (HIGH TEMP): ");
        for (int i = 0; i < 4; i++) {
            if (alert->aux_ov & (1 << i)) {
                printf("AUX%d ", i + 3);  // AUX3~6
            }
        }
        printf("\n");
    }
    
    // 자가진단 오류
    if (alert->conv_error) {
        printf("⚠️ Conversion Error\n");
    }
    if (alert->crc_error) {
        printf("⚠️ CRC Error\n");
    }
    if (alert->cell_self_fail) {
        printf("⚠️ Cell Self-Test Failed\n");
    }
    if (alert->aux_self_fail) {
        printf("⚠️ AUX Self-Test Failed\n");
    }
    
    // 정상
    if (!alert->cell_ov && !alert->cell_uv && !alert->aux_ov &&
        !alert->conv_error && !alert->crc_error &&
        !alert->cell_self_fail && !alert->aux_self_fail) {
        printf("✅ All Clear\n");
    }
    
    printf("================================\n");
}
```

출력 예시:
```
=== Device 0 Alert Analysis ===
Raw: A=0x24 B=0x00 C=0x00
🔴 OVERVOLTAGE: Cell5 
🟡 UNDERVOLTAGE: Cell4 
================================
```

## 다중 디바이스 스캔

데이지체인 전체 스캔:

```c
typedef struct {
    uint8_t device_count;
    ad7280a_full_alert_t alerts[8];  // 최대 8개 디바이스
    
    // 요약
    bool any_ov;
    bool any_uv;
    bool any_aux;
    bool any_fault;
    
    uint8_t first_ov_device;
    uint8_t first_ov_cell;
    uint8_t first_uv_device;
    uint8_t first_uv_cell;
} bms_alert_scan_t;

/**
 * @brief 전체 시스템 알람 스캔
 */
HAL_StatusTypeDef bms_scan_all_alerts(
    ad7280a_handle_t *handle,
    uint8_t num_devices,
    bms_alert_scan_t *scan)
{
    HAL_StatusTypeDef ret;
    
    memset(scan, 0, sizeof(bms_alert_scan_t));
    scan->device_count = num_devices;
    scan->first_ov_device = 0xFF;
    scan->first_uv_device = 0xFF;
    
    for (uint8_t dev = 0; dev < num_devices; dev++) {
        ret = ad7280a_read_full_alert(handle, dev, &scan->alerts[dev]);
        if (ret != HAL_OK) {
            printf("Failed to read alert from device %d\n", dev);
            continue;
        }
        
        ad7280a_full_alert_t *a = &scan->alerts[dev];
        
        // 과전압 체크
        if (a->cell_ov) {
            scan->any_ov = true;
            if (scan->first_ov_device == 0xFF) {
                scan->first_ov_device = dev;
                for (int i = 0; i < 6; i++) {
                    if (a->cell_ov & (1 << i)) {
                        scan->first_ov_cell = i;
                        break;
                    }
                }
            }
        }
        
        // 저전압 체크
        if (a->cell_uv) {
            scan->any_uv = true;
            if (scan->first_uv_device == 0xFF) {
                scan->first_uv_device = dev;
                for (int i = 0; i < 6; i++) {
                    if (a->cell_uv & (1 << i)) {
                        scan->first_uv_cell = i;
                        break;
                    }
                }
            }
        }
        
        // AUX 체크
        if (a->aux_ov) {
            scan->any_aux = true;
        }
        
        // 폴트 체크
        if (a->conv_error || a->crc_error || 
            a->cell_self_fail || a->aux_self_fail) {
            scan->any_fault = true;
        }
    }
    
    return HAL_OK;
}

/**
 * @brief 스캔 결과 출력
 */
void bms_print_scan_result(const bms_alert_scan_t *scan)
{
    printf("\n======== BMS Alert Scan ========\n");
    printf("Devices: %d\n", scan->device_count);
    
    if (scan->any_ov) {
        printf("🔴 OV: First at Dev%d Cell%d\n",
               scan->first_ov_device, scan->first_ov_cell + 1);
    }
    if (scan->any_uv) {
        printf("🟡 UV: First at Dev%d Cell%d\n",
               scan->first_uv_device, scan->first_uv_cell + 1);
    }
    if (scan->any_aux) {
        printf("🟠 AUX Alarm (Temperature)\n");
    }
    if (scan->any_fault) {
        printf("⚠️ System Fault Detected\n");
    }
    
    if (!scan->any_ov && !scan->any_uv && !scan->any_aux && !scan->any_fault) {
        printf("✅ All Systems Normal\n");
    }
    printf("================================\n\n");
}
```

## 알람과 전압 동시 읽기

알람만으론 부족. **실제 전압**도 함께 봐야 판단 가능:

```c
typedef struct {
    ad7280a_full_alert_t alert;
    float cell_voltages[6];
    float aux_voltages[6];
    float total_voltage;
    float min_voltage;
    float max_voltage;
    uint8_t min_cell;
    uint8_t max_cell;
} device_status_t;

/**
 * @brief 디바이스 상태 전체 읽기
 */
HAL_StatusTypeDef bms_read_device_status(
    ad7280a_handle_t *handle,
    uint8_t device,
    device_status_t *status)
{
    HAL_StatusTypeDef ret;
    
    // 알람 읽기
    ret = ad7280a_read_full_alert(handle, device, &status->alert);
    if (ret != HAL_OK) return ret;
    
    // 셀 전압 읽기
    ret = ad7280a_read_cell_voltages(handle, device, status->cell_voltages);
    if (ret != HAL_OK) return ret;
    
    // AUX 전압 읽기
    ret = ad7280a_read_aux_voltages(handle, device, status->aux_voltages);
    if (ret != HAL_OK) return ret;
    
    // 통계 계산
    status->total_voltage = 0;
    status->min_voltage = 999.0f;
    status->max_voltage = 0;
    
    for (int i = 0; i < 6; i++) {
        float v = status->cell_voltages[i];
        status->total_voltage += v;
        
        if (v < status->min_voltage) {
            status->min_voltage = v;
            status->min_cell = i;
        }
        if (v > status->max_voltage) {
            status->max_voltage = v;
            status->max_cell = i;
        }
    }
    
    return HAL_OK;
}

/**
 * @brief 상태 출력
 */
void bms_print_device_status(uint8_t device, const device_status_t *s)
{
    printf("\n--- Device %d Status ---\n", device);
    
    // 전압 테이블
    printf("Cell Voltages:\n");
    for (int i = 0; i < 6; i++) {
        char flag = ' ';
        if (s->alert.cell_ov & (1 << i)) flag = '▲';
        if (s->alert.cell_uv & (1 << i)) flag = '▼';
        
        printf("  Cell %d: %6.3fV %c\n", i + 1, s->cell_voltages[i], flag);
    }
    
    printf("Total: %.2fV  Min: %.3fV(C%d)  Max: %.3fV(C%d)  Δ: %.0fmV\n",
           s->total_voltage,
           s->min_voltage, s->min_cell + 1,
           s->max_voltage, s->max_cell + 1,
           (s->max_voltage - s->min_voltage) * 1000);
    
    printf("------------------------\n");
}
```

출력 예시:
```
--- Device 0 Status ---
Cell Voltages:
  Cell 1: 3.312V  
  Cell 2: 3.308V  
  Cell 3: 3.315V  
  Cell 4: 2.489V ▼
  Cell 5: 3.682V ▲
  Cell 6: 3.301V  
Total: 19.41V  Min: 2.489V(C4)  Max: 3.682V(C5)  Δ: 1193mV
------------------------
```

## 알람 클리어 (Read-to-Clear)

AD7280A 알람은 **읽으면 자동 클리어**:

```c
/**
 * @brief 알람 클리어 (읽기로 클리어)
 */
HAL_StatusTypeDef ad7280a_clear_all_alerts(ad7280a_handle_t *handle)
{
    uint8_t dummy;
    
    for (uint8_t dev = 0; dev < handle->num_devices; dev++) {
        // 모든 Alert 레지스터 읽기 → 클리어
        ad7280a_read_register(handle, dev, 0x0D, &dummy);
        ad7280a_read_register(handle, dev, 0x0E, &dummy);
        ad7280a_read_register(handle, dev, 0x0F, &dummy);
    }
    
    printf("All alerts cleared\n");
    return HAL_OK;
}
```

**주의**: 클리어 후에도 조건이 계속되면 다시 알람 발생!

## 삽질: 비트 매핑 혼동

Cell 1~2와 Cell 3~6의 비트 위치가 다름:

```
Alert A: Cell3~6 (순서대로)
Alert B: Cell1~2 (OV/UV 위치 다름)

처음에 모든 셀이 같은 패턴이라고 생각 → 버그!
```

**해결**: 비트맵 재구성 함수로 통일

## 삽질: 읽기 순서

Alert 읽기 전에 Conversion 해야 최신 상태:

```c
// 잘못된 순서
ad7280a_read_full_alert(...);  // 이전 상태!

// 올바른 순서
ad7280a_start_conversion(...);
HAL_Delay(1);
ad7280a_read_full_alert(...);  // 최신 상태
```

## 정리

| 레지스터 | 내용 |
|----------|------|
| 0x0D (Alert A) | Cell 3~6 OV/UV |
| 0x0E (Alert B) | Cell 1~2, AUX OV/UV |
| 0x0F (Alert C) | 자가진단 결과 |

| 항목 | 방법 |
|------|------|
| 알람 읽기 | 3개 레지스터 순차 읽기 |
| 비트맵 변환 | Cell 1~2와 3~6 별도 처리 |
| 클리어 | 읽기로 자동 클리어 |
| 전체 스캔 | 모든 디바이스 순회 |

**다음 글에서**: 보호 로직 통합 - 알람, 밸런싱, FET 제어 통합.

---

## 시리즈 목차

**Part 5: 알람 & 보호**
- [#13 - 과전압/저전압 알람 설정](/posts/bms/ad7280a-bms-dev-13/)
- [#14 - Alert 핀 인터럽트 처리](/posts/bms/ad7280a-bms-dev-14/)
- **#15 - 알람 상태 읽기** ← 현재 글
- #16 - 보호 로직 통합

---

## 참고 자료

- [AD7280A Datasheet - Alert Registers](https://www.analog.com/media/en/technical-documentation/data-sheets/AD7280A.pdf)
