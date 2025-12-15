---
title: "AD7280A BMS 개발 삽질기 #27 - EMC 대응"
date: 2024-12-14
draft: false
tags: ["AD7280A", "BMS", "STM32", "EMC", "EMI", "노이즈"]
categories: ["BMS 개발"]
series: ["AD7280A BMS 개발"]
summary: "인버터 스위칭, 모터 구동... 노이즈의 바다에서 정확한 측정을 하려면? EMC 대응 실전 노하우."
---

## 지난 글 요약

[지난 글](/posts/bms/ad7280a-bms-dev-26/)에서 현장 트러블슈팅을 다뤘다. 많은 문제의 근본 원인은 **노이즈**였다. 이번엔 EMC를 정면으로 다루자.

## EMC란?

```
EMC (Electromagnetic Compatibility)
= 전자기 양립성

┌─────────────────────────────────────────┐
│                                         │
│  EMI (Emission)    +   EMS (Suscept.)   │
│  노이즈 방출           노이즈 내성       │
│                                         │
│  "남에게 피해 주지    "남의 노이즈에도   │
│   않기"               잘 동작하기"       │
│                                         │
└─────────────────────────────────────────┘
```

## BMS의 노이즈 환경

### 노이즈 발생원

```
           ┌──────────────────────────────────────┐
           │         전기차/산업 환경              │
           │                                      │
   ┌───────┴───────┐                              │
   │   인버터      │  10~20kHz 스위칭             │
   │  ████████████ │  dV/dt: 5000V/μs            │
   └───────────────┘                              │
           │                                      │
   ┌───────┴───────┐                              │
   │   모터        │  PWM 구동, 회전 노이즈       │
   │   ◐◐◐◐◐     │                              │
   └───────────────┘                              │
           │                                      │
   ┌───────┴───────┐                              │
   │  DC-DC 컨버터 │  고주파 스위칭               │
   │  ≋≋≋≋≋≋≋≋≋≋ │  100kHz ~ 2MHz              │
   └───────────────┘                              │
           │                                      │
           ▼                                      │
   ┌───────────────┐                              │
   │     BMS       │  ← 이 속에서 mV 측정!        │
   │  AD7280A      │                              │
   └───────────────┘                              │
           │                                      │
└──────────┴──────────────────────────────────────┘
```

### 노이즈 경로

```
노이즈 전파 방식:

1. 전도성 (Conducted)
   ────────→────────→────────→
   전원선, 신호선 통해 직접 전달

2. 복사성 (Radiated)
        ))))
       ))))    전자기파로 공간 전파
      ))))
     ))))

3. 커플링 (Coupling)
   ═══════════════  ← 인접 배선
   ───────────────  ← BMS 신호선
   용량성/유도성 결합
```

## EMC 문제 증상

### 증상 1: ADC 노이즈

```
셀 전압 측정 (노이즈 환경):

정상:     3.250V ──────────────────────────
실제:     3.250V ∿∿∿∿∿∿∿∿∿∿∿∿∿∿∿∿∿∿∿∿∿∿
                  ↑ ±30mV 노이즈

노이즈가 심하면:
- 셀 전압 오측정
- 과전압/저전압 오경보
- 밸런싱 오동작
```

### 증상 2: 통신 오류

```
SPI 파형 (이상적):
CLK:  ┌┐┌┐┌┐┌┐┌┐┌┐┌┐┌┐
MOSI: ──┘└──┘└──┘└──┘└

SPI 파형 (노이즈):
CLK:  ┌┐┌╥┐┌┐╥┌┐┌┐┌┐  ← 글리치
MOSI: ──┘└╨─┘└──┘└──┘  ← 데이터 오류
```

### 증상 3: 리셋/오동작

```
MCU 리셋 원인:
┌─────────────────────────────────────┐
│  전원 노이즈 → 브라운아웃 리셋      │
│  ESD → 래치업 → IC 손상            │
│  클럭 노이즈 → 오동작              │
└─────────────────────────────────────┘
```

## 하드웨어 대책

### 1. 전원 필터링

```
전원 입력 필터:
             L1                 L2
    ┌───────╥╥╥───┬───────╥╥╥───┐
VIN─┤       10μH  │       4.7μH  ├─VDD
    │             │              │
    ┴ C1         ┴ C2           ┴ C3
    │ 100μF      │ 10μF         │ 0.1μF
    ┴            ┴              ┴
   GND          GND            GND

• 벌크 캡: 저주파 리플 제거
• 세라믹 캡: 고주파 노이즈 제거
• 인덕터: 전도성 노이즈 차단
```

### 2. 디커플링 캐패시터

```c
// AD7280A 디커플링
// 각 VDD 핀 근처에 배치

/*
                AD7280A
              ┌─────────┐
   100nF ──┬─┤VDD1     │
           │ │         │
   100nF ──┼─┤VDD2     │
           │ │         │
   4.7μF ──┴─┤AVDD     │  ← 아날로그 전원 특히 중요
              └─────────┘
              
배치 규칙:
• IC 핀에서 3mm 이내
• 캡 → 핀 경로 최단
• GND 비아 바로 옆에
*/
```

### 3. 입력 필터

```
셀 전압 입력 필터 (각 채널):

Cell+ ───R1───┬───────── AD7280A Cell Input
              │
        1kΩ  ┴ C1
              │ 100nF
             GND

• RC 필터: fc = 1/(2π×1k×100n) ≈ 1.6kHz
• 고주파 노이즈 감쇠
• R 값 주의: 측정 오차 유발 가능
```

### 4. 실드 및 접지

```
케이블 실드:
┌────────────────────────────────────────┐
│  ═══════════════════════════════════   │ ← 실드
│  ─────────────────────────────────── ─ │ ← 신호선
│  ═══════════════════════════════════   │ ← 실드
└────────────────────────────────────────┘
        │                         │
        ▼                         ×  ← 한쪽만 접지 (Ground Loop 방지)
       GND

PCB 접지:
┌─────────────────────────────────────┐
│  ████████████████████████████████   │ ← 솔리드 GND 플레인
│  ████████████████████████████████   │
│  ████████████████████████████████   │
│    │        │        │        │     │
│    ▼        ▼        ▼        ▼     │ ← 다중 비아
└─────────────────────────────────────┘
```

### 5. PCB 레이아웃

```
레이어 스택업:
┌─────────────────┐
│   Top (Signal)  │ ← 신호선
├─────────────────┤
│   GND Plane     │ ← 솔리드 GND (중요!)
├─────────────────┤
│   Power Plane   │ ← 전원
├─────────────────┤
│ Bottom (Signal) │ ← 신호선
└─────────────────┘

배선 규칙:
• 고속 신호: GND 바로 위/아래
• 아날로그/디지털 분리
• 대전류 경로: 신호선과 직교
• Loop 면적 최소화
```

## 소프트웨어 대책

### 1. 디지털 필터링

```c
// 이동 평균 필터
#define FILTER_SIZE 16

typedef struct {
    float buffer[FILTER_SIZE];
    int index;
    float sum;
} moving_avg_t;

float MovingAvg_Update(moving_avg_t *f, float new_val) {
    f->sum -= f->buffer[f->index];
    f->buffer[f->index] = new_val;
    f->sum += new_val;
    f->index = (f->index + 1) % FILTER_SIZE;
    
    return f->sum / FILTER_SIZE;
}

// 중간값 필터 (스파이크 노이즈에 강함)
float MedianFilter_3(float a, float b, float c) {
    if (a > b) {
        if (b > c) return b;      // a > b > c
        if (a > c) return c;      // a > c > b
        return a;                  // c > a > b
    } else {
        if (a > c) return a;      // b > a > c
        if (b > c) return c;      // b > c > a
        return b;                  // c > b > a
    }
}
```

### 2. 오버샘플링 + 평균

```c
// AD7280A 여러 번 읽고 평균
float AD7280A_ReadCellFiltered(uint8_t dev, uint8_t cell) {
    #define OVERSAMPLE 8
    
    float sum = 0;
    
    for (int i = 0; i < OVERSAMPLE; i++) {
        AD7280A_StartConversion(CONV_ALL_CELLS);
        HAL_Delay(2);  // 변환 완료 대기
        
        uint16_t raw = AD7280A_ReadCell(dev, cell);
        float mv = AD7280A_RawToMillivolt(raw);
        sum += mv;
    }
    
    return sum / OVERSAMPLE;
}
```

### 3. 글리치 제거

```c
// 급격한 변화 필터링
float DeglitchFilter(float *last_val, float new_val, float max_delta) {
    float delta = fabsf(new_val - *last_val);
    
    // 변화가 너무 크면 이상값으로 판단
    if (delta > max_delta) {
        // 무시하거나 제한된 변화만 적용
        if (new_val > *last_val) {
            *last_val += max_delta;
        } else {
            *last_val -= max_delta;
        }
    } else {
        *last_val = new_val;
    }
    
    return *last_val;
}

// 사용 예
void ProcessCellVoltage(int cell, float raw_mv) {
    static float last_mv[24] = {0};
    
    // 한 번에 100mV 이상 변화 불가능 (물리적으로)
    float filtered = DeglitchFilter(&last_mv[cell], raw_mv, 100.0f);
    
    g_bms.cell_mv[cell] = filtered;
}
```

### 4. 통신 에러 검출/복구

```c
// SPI 통신 무결성 확인
HAL_StatusTypeDef AD7280A_SafeRead(uint8_t dev, uint8_t reg, 
                                    uint8_t *data) {
    uint8_t read1, read2;
    int retries = 0;
    
    do {
        // 두 번 읽기
        AD7280A_ReadRegister(dev, reg, &read1);
        AD7280A_ReadRegister(dev, reg, &read2);
        
        // 같으면 신뢰
        if (read1 == read2) {
            *data = read1;
            return HAL_OK;
        }
        
        retries++;
        HAL_Delay(1);
        
    } while (retries < 3);
    
    // 3번 다 다르면 에러
    return HAL_ERROR;
}
```

### 5. 측정 타이밍 조정

```c
// 노이즈가 심한 구간 피하기
void SmartMeasurement(void) {
    // 인버터 PWM 주기: 100μs (10kHz)
    // 스위칭 순간 (0%, 100% 근처) 피하기
    
    // 방법 1: PWM 동기화 (SYNC 신호 있는 경우)
    WaitForPWMSync();
    HAL_Delay_us(25);  // 25% 위치
    AD7280A_StartConversion();
    
    // 방법 2: 인버터 OFF 시 측정
    if (g_inverter.state == INV_IDLE) {
        AD7280A_StartConversion();
    }
}
```

## EMC 테스트

### 자가 테스트 방법

```c
// ADC 노이즈 레벨 측정
void MeasureNoiseLevel(void) {
    #define NOISE_SAMPLES 100
    
    float samples[NOISE_SAMPLES];
    float sum = 0, sum_sq = 0;
    
    // 정적 상태에서 연속 측정
    for (int i = 0; i < NOISE_SAMPLES; i++) {
        AD7280A_StartConversion(CONV_ALL_CELLS);
        HAL_Delay(2);
        samples[i] = AD7280A_ReadCell_mV(0, 0);
        sum += samples[i];
    }
    
    float mean = sum / NOISE_SAMPLES;
    
    // 표준편차 계산
    for (int i = 0; i < NOISE_SAMPLES; i++) {
        float diff = samples[i] - mean;
        sum_sq += diff * diff;
    }
    
    float std_dev = sqrtf(sum_sq / NOISE_SAMPLES);
    
    // 최대-최소
    float min_val = samples[0], max_val = samples[0];
    for (int i = 1; i < NOISE_SAMPLES; i++) {
        if (samples[i] < min_val) min_val = samples[i];
        if (samples[i] > max_val) max_val = samples[i];
    }
    
    printf("=== ADC Noise Analysis ===\n");
    printf("Mean: %.2f mV\n", mean);
    printf("Std Dev: %.2f mV\n", std_dev);
    printf("Peak-Peak: %.2f mV\n", max_val - min_val);
    printf("SNR: %.1f dB\n", 20 * log10f(mean / std_dev));
    
    // 판정
    if (std_dev < 2.0f) {
        printf("Status: EXCELLENT\n");
    } else if (std_dev < 5.0f) {
        printf("Status: GOOD\n");
    } else if (std_dev < 10.0f) {
        printf("Status: ACCEPTABLE\n");
    } else {
        printf("Status: POOR - EMC improvement needed!\n");
    }
}
```

### 스트레스 테스트

```c
// 인버터 구동 중 테스트
void EMC_StressTest(void) {
    printf("=== EMC Stress Test ===\n");
    printf("Start inverter and motor...\n\n");
    
    int error_count = 0;
    int total_count = 0;
    
    for (int i = 0; i < 1000; i++) {
        // SPI 통신
        if (AD7280A_ReadAllCells() != HAL_OK) {
            error_count++;
            printf("SPI Error at %d\n", i);
        }
        
        // CAN 통신
        if (CAN_TxTest() != HAL_OK) {
            error_count++;
            printf("CAN Error at %d\n", i);
        }
        
        total_count += 2;
        HAL_Delay(10);
    }
    
    printf("\nResult: %d errors / %d operations (%.2f%%)\n",
           error_count, total_count, 
           (float)error_count / total_count * 100);
}
```

## 실전 체크리스트

```
EMC 대책 체크리스트:
┌────────────────────────────────────────┐
│ 하드웨어                               │
├────────────────────────────────────────┤
│ □ 전원 필터 (π 필터)                   │
│ □ 디커플링 캡 (각 IC)                  │
│ □ 입력 RC 필터 (셀 탭)                 │
│ □ GND 플레인 (솔리드)                  │
│ □ 케이블 실드/트위스트                 │
│ □ 커넥터 필터 (페라이트)               │
│ □ TVS/ESD 보호                         │
├────────────────────────────────────────┤
│ 소프트웨어                             │
├────────────────────────────────────────┤
│ □ 디지털 필터 (이동평균/중간값)        │
│ □ 오버샘플링                           │
│ □ 글리치 제거                          │
│ □ 통신 재시도                          │
│ □ CRC 검증                             │
│ □ 타이밍 조정                          │
├────────────────────────────────────────┤
│ 테스트                                 │
├────────────────────────────────────────┤
│ □ 노이즈 레벨 측정                     │
│ □ 스트레스 테스트                      │
│ □ 장시간 안정성                        │
└────────────────────────────────────────┘
```

## 정리

| 대책 | 효과 | 비용 |
|------|------|------|
| 전원 필터 | 높음 | 낮음 |
| 디커플링 | 높음 | 낮음 |
| RC 입력 필터 | 중간 | 낮음 |
| GND 플레인 | 높음 | 중간 |
| 케이블 실드 | 높음 | 중간 |
| SW 필터 | 중간 | 없음 |
| 오버샘플링 | 중간 | 없음 |

**우선순위**:
1. GND 플레인 확보 (설계 단계)
2. 전원/디커플링 (필수)
3. 소프트웨어 필터 (무료)
4. 케이블 실드 (효과 큼)
5. 입력 필터 (미세 조정)

**다음 글에서**: 양산 검사 지그 - EOL 테스트 자동화.

---

## 시리즈 네비게이션

**Part 8: 실전 적용편**
- [#25 - 72V 팩 첫 연결](/posts/bms/ad7280a-bms-dev-25/)
- [#26 - 현장 트러블슈팅](/posts/bms/ad7280a-bms-dev-26/)
- **#27 - EMC 대응** ← 현재 글
- #28 - 양산 검사 지그

---

## 참고 자료

- [EMC for Automotive](https://www.nxp.com/docs/en/application-note/AN11160.pdf)
- [PCB Design for EMC](https://www.analog.com/en/technical-articles/pcb-design-for-emc.html)
- [Noise Reduction Techniques](https://www.ti.com/lit/an/slva531/slva531.pdf)
