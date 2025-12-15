---
title: "AD7280A BMS 개발 삽질기 #25 - 72V 팩 첫 연결"
date: 2024-12-14
draft: false
tags: ["AD7280A", "BMS", "STM32", "72V", "LiFePO4", "실전"]
categories: ["BMS 개발"]
series: ["AD7280A BMS 개발"]
summary: "드디어 실제 72V 배터리 팩과의 첫 만남! 벤치 테스트와는 차원이 다른 긴장감. 고전압 작업의 현실."
---

## 지난 글 요약

[Part 7](/posts/bms/ad7280a-bms-dev-24/)에서 절연 모니터링까지 구현했다. 이제 **실제 배터리**와 연결할 시간이다. 시뮬레이션과 현실의 간극을 메워보자.

## 드디어 실전

### 지금까지의 테스트 환경

```
벤치 테스트:
┌─────────────────────────────────────────┐
│  전원 공급기 (3.3V × 24 = 79.2V 모사)   │
│  ┌─────┐ ┌─────┐ ┌─────┐              │
│  │ PS1 │ │ PS2 │ │ PS3 │ ...          │
│  │3.3V │ │3.3V │ │3.3V │              │
│  └──┬──┘ └──┬──┘ └──┬──┘              │
│     └───────┴───────┴── AD7280A        │
└─────────────────────────────────────────┘
장점: 안전, 전압 조절 가능
단점: 실제 배터리 특성 없음
```

### 실제 배터리 팩

```
실전:
┌─────────────────────────────────────────┐
│     24S LiFePO4 배터리 팩 (72V)         │
│  ┌─────┬─────┬─────┬─────┬─────┐       │
│  │Cell1│Cell2│Cell3│ ... │Cel24│       │
│  │3.2V │3.2V │3.2V │     │3.2V │       │
│  └──┬──┴──┬──┴──┬──┴─────┴──┬──┘       │
│     │     │     │           │          │
│     └─────┴─────┴───────────┴── BMS    │
│                                         │
│  총 전압: 76.8V (3.2V × 24)            │
│  에너지: 약 3.8kWh                      │
└─────────────────────────────────────────┘
위험: 감전, 화재, 폭발 가능
```

## 작업 전 준비

### 안전 장비

```
필수 장비:
┌────────────────────────────────────────┐
│  □ 절연 장갑 (1000V 등급)              │
│  □ 보안경                               │
│  □ 절연 매트                            │
│  □ 소화기 (ABC 또는 CO2)               │
│  □ 절연 공구 세트                       │
│  □ 멀티미터 (CAT III 600V 이상)        │
└────────────────────────────────────────┘
```

### 작업 환경

```c
// 체크리스트
typedef struct {
    bool insulation_mat;        // 절연 매트
    bool fire_extinguisher;     // 소화기 비치
    bool emergency_contact;     // 비상 연락처
    bool first_aid_kit;         // 응급 키트
    bool ventilation;           // 환기
    bool no_metal_jewelry;      // 금속 장신구 제거
} safety_checklist_t;

bool Safety_PreWorkCheck(safety_checklist_t *checklist) {
    return checklist->insulation_mat &&
           checklist->fire_extinguisher &&
           checklist->emergency_contact &&
           checklist->first_aid_kit &&
           checklist->ventilation &&
           checklist->no_metal_jewelry;
}
```

## 연결 전 점검

### 1단계: BMS 보드 단독 테스트

```c
// 배터리 연결 전, 저전압으로 최종 확인
void PreConnection_Test(void) {
    // 1. 전원 공급 (5V/3.3V만)
    printf("=== BMS Board Self-Test ===\n");
    
    // 2. SPI 통신 확인
    uint32_t dev_id = AD7280A_ReadDeviceID();
    printf("Device ID: 0x%08X\n", dev_id);
    
    // 3. 레지스터 R/W 테스트
    AD7280A_WriteRegister(0, REG_CTRL_LB, 0x10);
    uint8_t readback = AD7280A_ReadRegister(0, REG_CTRL_LB);
    printf("Register R/W: %s\n", (readback == 0x10) ? "PASS" : "FAIL");
    
    // 4. Alert 핀 상태
    printf("Alert Pin: %s\n", 
           HAL_GPIO_ReadPin(ALERT_PORT, ALERT_PIN) ? "HIGH" : "LOW");
    
    // 5. 릴레이 드라이버
    printf("Testing relays...\n");
    MainContactor_On();
    HAL_Delay(100);
    MainContactor_Off();
    printf("Relay test complete\n");
}
```

### 2단계: 배터리 팩 점검

```
배터리 측 확인:
┌────────────────────────────────────────┐
│  1. 총 전압 측정: _____ V (정상: 72~84V)│
│  2. 셀 밸런스: 편차 _____ mV           │
│  3. 외관 검사: 부풀음, 누액 없음       │
│  4. 커넥터 상태: 단자 청결             │
│  5. 퓨즈 상태: 정상                    │
└────────────────────────────────────────┘
```

### 3단계: 배선 점검

```c
// 연결 전 저항 측정으로 배선 확인
void Wiring_Check(void) {
    printf("=== Wiring Verification ===\n");
    printf("Measure with multimeter (power OFF):\n\n");
    
    printf("1. Cell tap wiring:\n");
    printf("   C0-C1: should be ~3.2V when connected\n");
    printf("   C1-C2: should be ~3.2V when connected\n");
    // ... 모든 셀 탭
    
    printf("2. Thermistor wiring:\n");
    printf("   NTC1: ~10k ohm at 25C\n");
    printf("   NTC2: ~10k ohm at 25C\n");
    
    printf("3. Current sense:\n");
    printf("   Shunt: < 1 ohm\n");
    
    printf("4. Power supply:\n");
    printf("   VDD to GND: > 1M ohm (no short)\n");
}
```

## 첫 연결 순서

### 연결 시퀀스

```
               안전한 연결 순서
    ┌─────────────────────────────────────┐
    │ 1. 메인 퓨즈/스위치 OFF 확인        │
    ├─────────────────────────────────────┤
    │ 2. GND 연결 먼저 (Pack-)            │
    ├─────────────────────────────────────┤
    │ 3. 셀 탭 연결 (낮은 전위부터)       │
    │    C0 → C1 → C2 → ... → C24        │
    ├─────────────────────────────────────┤
    │ 4. 센서 연결 (온도, 전류)           │
    ├─────────────────────────────────────┤
    │ 5. 통신 연결 (CAN, UART)            │
    ├─────────────────────────────────────┤
    │ 6. 최종 점검 후 전원 인가           │
    └─────────────────────────────────────┘
```

### 셀 탭 연결 시 주의

```
중요: 순서대로 연결!

잘못된 연결:
C0 ─────── (연결)
C12 ────── (연결)  ← 12셀 전압(38V)이 IC에 인가!
          → IC 파손

올바른 연결:
C0 ─────── (연결)
C1 ─────── (연결)  ← 1셀 전압(3.2V)만 인가
C2 ─────── (연결)
...순차적으로
```

## 전원 인가 - 긴장의 순간

### 첫 전원 ON

```c
void FirstPowerOn(void) {
    printf("\n");
    printf("╔════════════════════════════════════╗\n");
    printf("║     FIRST POWER-ON SEQUENCE        ║\n");
    printf("╚════════════════════════════════════╝\n\n");
    
    // 1. 전류 제한 전원 공급기 사용 권장
    printf("[1] Current limit set? (100mA recommended)\n");
    printf("    Press any key to continue...\n");
    getchar();
    
    // 2. 전원 인가
    printf("[2] Applying power...\n");
    HAL_Delay(1000);
    
    // 3. 전류 확인
    printf("[3] Check current draw:\n");
    printf("    Normal: 10-50mA\n");
    printf("    If > 100mA: STOP immediately!\n\n");
    
    // 4. LED 상태
    printf("[4] LED status:\n");
    printf("    Power LED: should be ON\n");
    printf("    Fault LED: should be OFF\n\n");
    
    // 5. 첫 통신
    printf("[5] Attempting first communication...\n");
    
    if (AD7280A_Init() == HAL_OK) {
        printf("    ✓ AD7280A initialization successful!\n");
    } else {
        printf("    ✗ AD7280A initialization FAILED!\n");
        printf("    → Check wiring and power\n");
        return;
    }
}
```

### 첫 셀 전압 읽기

```c
void FirstCellReading(void) {
    printf("\n=== First Cell Voltage Reading ===\n\n");
    
    // 변환 시작
    AD7280A_StartConversion(CONV_ALL_CELLS);
    HAL_Delay(50);  // 변환 완료 대기
    
    // 결과 읽기
    cell_data_t cells[24];
    AD7280A_ReadAllCells(cells);
    
    // 출력
    float total_voltage = 0;
    float min_cell = 5000.0f;
    float max_cell = 0.0f;
    
    printf("Device | Cell | Voltage (mV) | Status\n");
    printf("-------|------|--------------|--------\n");
    
    for (int dev = 0; dev < 4; dev++) {
        for (int cell = 0; cell < 6; cell++) {
            int idx = dev * 6 + cell;
            float mv = cells[idx].voltage_mv;
            
            total_voltage += mv;
            if (mv < min_cell) min_cell = mv;
            if (mv > max_cell) max_cell = mv;
            
            char *status = "OK";
            if (mv < 2500) status = "LOW!";
            if (mv > 3650) status = "HIGH!";
            if (mv < 1000 || mv > 4500) status = "ERROR!";
            
            printf("   %d   |  %d   |    %7.1f   | %s\n",
                   dev, cell, mv, status);
        }
        printf("-------|------|--------------|--------\n");
    }
    
    printf("\nSummary:\n");
    printf("  Total Voltage: %.2f V\n", total_voltage / 1000.0f);
    printf("  Min Cell: %.1f mV\n", min_cell);
    printf("  Max Cell: %.1f mV\n", max_cell);
    printf("  Imbalance: %.1f mV\n", max_cell - min_cell);
    
    // 판정
    if (max_cell - min_cell < 50) {
        printf("\n  ✓ Cell balance: GOOD\n");
    } else if (max_cell - min_cell < 100) {
        printf("\n  △ Cell balance: ACCEPTABLE (balancing needed)\n");
    } else {
        printf("\n  ✗ Cell balance: POOR (check cells)\n");
    }
}
```

## 삽질 1: 셀 탭 접촉 불량

### 증상

```
측정값:
Cell 0: 3245 mV  ✓
Cell 1: 3251 mV  ✓
Cell 2: 6498 mV  ← 이상!
Cell 3: 3240 mV  ✓
...
```

### 원인

```
정상:                    접촉 불량:
C0 ──┬── Cell0          C0 ──┬── Cell0
     │                       │
C1 ──┼── Cell1          C1 ──╳── (끊어짐)
     │                       │
C2 ──┼── Cell2          C2 ──┼── Cell1+Cell2 합산!
     │                       │
C3 ──┴── Cell3          C3 ──┴── Cell3
```

### 해결

```c
// 접촉 불량 감지
bool CheckCellTapConnection(cell_data_t *cells, int num_cells) {
    for (int i = 0; i < num_cells; i++) {
        // 비정상적으로 높은 전압 = 2셀 이상 합산
        if (cells[i].voltage_mv > 4500) {
            printf("ERROR: Cell %d shows %.1f mV\n", 
                   i, cells[i].voltage_mv);
            printf("       → Check tap connection at C%d\n", i);
            return false;
        }
        
        // 비정상적으로 낮은 전압
        if (cells[i].voltage_mv < 1000) {
            printf("ERROR: Cell %d shows %.1f mV\n",
                   i, cells[i].voltage_mv);
            printf("       → Open circuit at C%d or C%d\n", i, i+1);
            return false;
        }
    }
    return true;
}
```

## 삽질 2: 데이지체인 통신 실패

### 증상

```
Device 0: OK (6 cells read)
Device 1: OK (6 cells read)
Device 2: TIMEOUT
Device 3: TIMEOUT
```

### 원인 분석

```
             SDI      SDO
             ↓        ↑
        ┌────┴────────┴────┐
Dev 0   │    AD7280A       │  ← OK
        └────┬────────┬────┘
             ↓        ↑
        ┌────┴────────┴────┐
Dev 1   │    AD7280A       │  ← OK
        └────┬────────┬────┘
             ↓        ↑
        ╳────┴────────┴────╳  ← 연결 불량!
        ┌────┬────────┬────┐
Dev 2   │    AD7280A       │  ← 통신 안 됨
        └────┴────────┴────┘
```

### 해결

```c
// 데이지체인 진단
void DiagnoseChain(void) {
    printf("=== Daisy Chain Diagnosis ===\n");
    
    for (int dev = 0; dev < MAX_DEVICES; dev++) {
        // 개별 디바이스 핑
        HAL_StatusTypeDef status = AD7280A_PingDevice(dev);
        
        if (status == HAL_OK) {
            printf("Device %d: CONNECTED\n", dev);
        } else {
            printf("Device %d: NOT RESPONDING\n", dev);
            printf("  → Check wiring between Dev %d and Dev %d\n",
                   dev - 1, dev);
            printf("  → Verify power supply to Dev %d\n", dev);
            break;
        }
    }
}
```

## 삽질 3: 노이즈로 인한 오측정

### 증상

```
연속 측정 결과:
Time 0:  3245 mV
Time 1:  3312 mV  ← 튐
Time 2:  3248 mV
Time 3:  3189 mV  ← 튐
Time 4:  3251 mV
```

### 원인

```
노이즈 원인:
┌─────────────────────────────────────┐
│ 1. 인버터 스위칭 노이즈             │
│ 2. 모터 드라이브 PWM               │
│ 3. 접지 루프                        │
│ 4. 긴 배선 (안테나 효과)           │
└─────────────────────────────────────┘
```

### 해결

```c
// 소프트웨어 필터링
#define CELL_FILTER_SIZE    8

typedef struct {
    float samples[CELL_FILTER_SIZE];
    int index;
    bool filled;
} cell_filter_t;

static cell_filter_t g_cell_filters[24];

float Cell_GetFiltered(int cell_idx, float new_sample) {
    cell_filter_t *f = &g_cell_filters[cell_idx];
    
    // 새 샘플 저장
    f->samples[f->index] = new_sample;
    f->index = (f->index + 1) % CELL_FILTER_SIZE;
    if (f->index == 0) f->filled = true;
    
    // 중간값 필터 (노이즈에 강함)
    int count = f->filled ? CELL_FILTER_SIZE : f->index;
    
    // 복사 후 정렬
    float sorted[CELL_FILTER_SIZE];
    memcpy(sorted, f->samples, count * sizeof(float));
    
    // 간단한 버블 정렬
    for (int i = 0; i < count - 1; i++) {
        for (int j = 0; j < count - i - 1; j++) {
            if (sorted[j] > sorted[j + 1]) {
                float tmp = sorted[j];
                sorted[j] = sorted[j + 1];
                sorted[j + 1] = tmp;
            }
        }
    }
    
    // 중간값 반환
    return sorted[count / 2];
}
```

## 기능 검증 체크리스트

```c
typedef struct {
    bool cell_voltage_read;
    bool temperature_read;
    bool balancing_test;
    bool overvoltage_alert;
    bool undervoltage_alert;
    bool communication_stable;
    bool current_sense;
    bool soc_calculation;
    bool can_communication;
    bool data_logging;
} function_checklist_t;

void RunFunctionTest(function_checklist_t *result) {
    printf("\n=== Function Verification ===\n\n");
    
    // 1. 셀 전압
    printf("[1] Cell Voltage Reading... ");
    result->cell_voltage_read = TestCellVoltage();
    printf("%s\n", result->cell_voltage_read ? "PASS" : "FAIL");
    
    // 2. 온도 측정
    printf("[2] Temperature Reading... ");
    result->temperature_read = TestTemperature();
    printf("%s\n", result->temperature_read ? "PASS" : "FAIL");
    
    // 3. 밸런싱
    printf("[3] Balancing Function... ");
    result->balancing_test = TestBalancing();
    printf("%s\n", result->balancing_test ? "PASS" : "FAIL");
    
    // 4. 과전압 알람
    printf("[4] Overvoltage Alert... ");
    result->overvoltage_alert = TestOVAlert();
    printf("%s\n", result->overvoltage_alert ? "PASS" : "FAIL");
    
    // 5. 저전압 알람
    printf("[5] Undervoltage Alert... ");
    result->undervoltage_alert = TestUVAlert();
    printf("%s\n", result->undervoltage_alert ? "PASS" : "FAIL");
    
    // 6. 통신 안정성
    printf("[6] Communication Stability... ");
    result->communication_stable = TestCommStability();
    printf("%s\n", result->communication_stable ? "PASS" : "FAIL");
    
    // 7. 전류 센싱
    printf("[7] Current Sensing... ");
    result->current_sense = TestCurrentSense();
    printf("%s\n", result->current_sense ? "PASS" : "FAIL");
    
    // 8. SOC 계산
    printf("[8] SOC Calculation... ");
    result->soc_calculation = TestSOC();
    printf("%s\n", result->soc_calculation ? "PASS" : "FAIL");
    
    // 9. CAN 통신
    printf("[9] CAN Communication... ");
    result->can_communication = TestCAN();
    printf("%s\n", result->can_communication ? "PASS" : "FAIL");
    
    // 10. 데이터 로깅
    printf("[10] Data Logging... ");
    result->data_logging = TestLogging();
    printf("%s\n", result->data_logging ? "PASS" : "FAIL");
    
    // 종합
    int pass_count = 0;
    pass_count += result->cell_voltage_read;
    pass_count += result->temperature_read;
    pass_count += result->balancing_test;
    pass_count += result->overvoltage_alert;
    pass_count += result->undervoltage_alert;
    pass_count += result->communication_stable;
    pass_count += result->current_sense;
    pass_count += result->soc_calculation;
    pass_count += result->can_communication;
    pass_count += result->data_logging;
    
    printf("\n=== Result: %d/10 PASSED ===\n", pass_count);
}
```

## 첫 연결 성공!

```
╔════════════════════════════════════════════╗
║         FIRST CONNECTION SUCCESS!          ║
╠════════════════════════════════════════════╣
║  Pack Voltage:  76.82 V                    ║
║  Cell Count:    24                         ║
║  Min Cell:      3195 mV                    ║
║  Max Cell:      3208 mV                    ║
║  Imbalance:     13 mV                      ║
║  Temperature:   23.5 °C                    ║
║  Status:        HEALTHY                    ║
╚════════════════════════════════════════════╝
```

## 정리

| 단계 | 내용 | 체크 |
|------|------|------|
| 준비 | 안전 장비, 환경 | □ |
| 보드 테스트 | 저전압 단독 동작 | □ |
| 배터리 점검 | 전압, 외관, 커넥터 | □ |
| 배선 점검 | 저항 측정 | □ |
| 연결 | GND → 셀탭 → 센서 순 | □ |
| 전원 ON | 전류 제한, LED 확인 | □ |
| 기능 검증 | 10개 항목 테스트 | □ |

**핵심 교훈**:
1. 안전이 최우선
2. 순서를 지켜라
3. 한 단계씩 확인하라
4. 이상하면 멈춰라

**다음 글에서**: 현장 트러블슈팅 - 예상 못한 문제들.

---

## 시리즈 네비게이션

**Part 8: 실전 적용편**
- **#25 - 72V 팩 첫 연결** ← 현재 글
- #26 - 현장 트러블슈팅
- #27 - EMC 대응
- #28 - 양산 검사 지그

---

## 참고 자료

- [High Voltage Safety](https://www.osha.gov/electrical)
- [Battery Pack Assembly Guide](https://batteryuniversity.com/article/bu-803-how-to-make-battery-packs)
- [LiFePO4 Handling Precautions](https://www.lithiumbatterypower.com/pages/safety)
