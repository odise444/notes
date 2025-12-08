---
title: "AD7280A BMS 개발 삽질기 #10 - 패시브 밸런싱 기초"
date: 2024-12-08
draft: false
tags: ["BMS", "AD7280A", "임베디드", "배터리", "삽질", "밸런싱"]
categories: ["BMS 개발"]
summary: "셀 전압이 다 다르다. AD7280A 내장 밸런싱 스위치로 맞춰보자."
---

## 지난 글 요약

[지난 글](/posts/bms/ad7280a-bms-dev-09/)에서 8개 데이지체인(48셀) 구성을 완료했다. 이제 측정은 되는데... 셀 전압이 제각각이다.

## 왜 밸런싱이 필요한가?

리튬이온 배터리 팩에서 각 셀은 완벽히 동일하지 않다.

```
Cell 1: 4.18V
Cell 2: 4.15V
Cell 3: 4.20V ← 가장 높음
Cell 4: 4.12V ← 가장 낮음
Cell 5: 4.17V
Cell 6: 4.16V
```

**문제점:**
- 충전 시: Cell 3이 먼저 4.2V 도달 → 충전 중단 → 다른 셀 덜 충전
- 방전 시: Cell 4가 먼저 3.0V 도달 → 방전 중단 → 용량 손해

밸런싱 없으면 **가장 약한 셀이 전체 용량을 결정**한다.

## 밸런싱 방식

| 방식 | 원리 | 장점 | 단점 |
|------|------|------|------|
| 패시브 | 높은 셀 방전 (저항) | 단순, 저렴 | 에너지 낭비 (열) |
| 액티브 | 셀 간 에너지 전송 | 효율적 | 복잡, 비쌈 |

AD7280A는 **패시브 밸런싱**을 위한 내장 스위치가 있다.

## AD7280A 밸런싱 회로

데이터시트 Figure 28을 보면:

```
        VINx ──┬──────────────────┐
               │                  │
              [R] 방전 저항        │
               │                  │
        CBx ───┴──[내장 SW]───────┤
                                  │
        VINx-1 ────────────────────┘
```

- **내장 스위치**: 각 셀마다 1개 (총 6개/디바이스)
- **외부 방전 저항**: 사용자가 선정
- **CBx 핀**: Cell Balance 출력 (스위치 제어)

## Cell Balance 레지스터

```c
// Cell Balance Register (Address 0x14)
// D5:D0 = CB6:CB1 (각 셀 밸런싱 ON/OFF)

#define AD7280A_REG_CELL_BALANCE    0x14

#define AD7280A_CB1    (1 << 0)  // Cell 1 밸런싱
#define AD7280A_CB2    (1 << 1)  // Cell 2 밸런싱
#define AD7280A_CB3    (1 << 2)  // Cell 3 밸런싱
#define AD7280A_CB4    (1 << 3)  // Cell 4 밸런싱
#define AD7280A_CB5    (1 << 4)  // Cell 5 밸런싱
#define AD7280A_CB6    (1 << 5)  // Cell 6 밸런싱
```

## 방전 저항 선정

### 전류 계산

```
밸런싱 전류 = 셀 전압 / 저항

예: 4.2V 셀, 100Ω 저항
I = 4.2V / 100Ω = 42mA
```

### 저항 선정 기준

| 저항값 | 전류 (4.2V) | 발열 (P=I²R) | 밸런싱 속도 |
|--------|-------------|--------------|-------------|
| 33Ω | 127mA | 0.53W | 빠름 |
| 68Ω | 62mA | 0.26W | 중간 |
| 100Ω | 42mA | 0.18W | 느림 |
| 150Ω | 28mA | 0.12W | 매우 느림 |

**권장**: 68Ω ~ 100Ω (발열과 속도의 균형)

### 밸런싱 시간 계산

```
셀 용량: 3000mAh
전압 차이: 50mV
밸런싱 전류: 50mA

필요 용량 = (50mV / 4200mV) × 3000mAh ≈ 36mAh
밸런싱 시간 = 36mAh / 50mA = 0.72시간 ≈ 43분
```

## 삽질 1: 스위치가 안 켜진다

Cell Balance 레지스터에 값을 썼는데 아무 일도 안 일어난다.

```c
// 안 되는 코드
ad7280a_write_reg(0, AD7280A_REG_CELL_BALANCE, 0x01);
```

**원인**: CB Timer가 0이면 스위치가 즉시 꺼진다!

### CB Timer 설정 필수

```c
// Control HB 레지스터 (Address 0x0D)
// D7:D4 = CB Timer

#define AD7280A_CB_TIMER_OFF      (0 << 4)   // 0초 (꺼짐)
#define AD7280A_CB_TIMER_71s      (1 << 4)   // 71.5초
#define AD7280A_CB_TIMER_143s     (2 << 4)   // 143초
#define AD7280A_CB_TIMER_214s     (3 << 4)   // 214.5초
#define AD7280A_CB_TIMER_286s     (4 << 4)   // 286초
#define AD7280A_CB_TIMER_357s     (5 << 4)   // 357.5초
#define AD7280A_CB_TIMER_429s     (6 << 4)   // 429초
#define AD7280A_CB_TIMER_INF      (7 << 4)   // 무한 (SW 제어)
```

**무한 모드 추천**: 소프트웨어로 직접 제어하는 게 유연하다.

```c
// 올바른 설정
void ad7280a_enable_balancing(void) {
    // Control HB: CB Timer = 무한
    uint8_t ctrl_hb = AD7280A_CB_TIMER_INF | 
                      AD7280A_CONV_AVG_8;
    ad7280a_write_all(AD7280A_REG_CONTROL_HB, ctrl_hb);
    
    // Cell 1 밸런싱 ON
    ad7280a_write_reg(0, AD7280A_REG_CELL_BALANCE, AD7280A_CB1);
}
```

## 삽질 2: 밸런싱 중 전압 측정 오류

밸런싱 스위치 ON 상태에서 셀 전압을 측정하면 값이 이상하다.

```
밸런싱 OFF: Cell 1 = 4180mV
밸런싱 ON:  Cell 1 = 4050mV ← ???
```

**원인**: 방전 저항에 전류가 흐르면서 전압 강하 발생.

### 해결: 측정 시 밸런싱 일시 중단

```c
void ad7280a_read_with_balance(float *voltages, uint8_t *balance_state) {
    // 1. 현재 밸런싱 상태 저장
    *balance_state = ad7280a_read_reg(0, AD7280A_REG_CELL_BALANCE);
    
    // 2. 밸런싱 OFF
    ad7280a_write_reg(0, AD7280A_REG_CELL_BALANCE, 0x00);
    
    // 3. 안정화 대기 (RC 시정수)
    sleep_ms(10);
    
    // 4. 전압 측정
    ad7280a_read_all_cells(voltages);
    
    // 5. 밸런싱 복원
    ad7280a_write_reg(0, AD7280A_REG_CELL_BALANCE, *balance_state);
}
```

## 간단한 밸런싱 로직

```c
#define BALANCE_THRESHOLD_MV    20   // 20mV 이상 차이나면 밸런싱
#define BALANCE_TARGET_MV       10   // 10mV 이내로 맞추기

void simple_balance(float *voltages, uint8_t num_cells) {
    // 최저 전압 찾기
    float min_voltage = voltages[0];
    for (int i = 1; i < num_cells; i++) {
        if (voltages[i] < min_voltage) {
            min_voltage = voltages[i];
        }
    }
    
    // 밸런싱 마스크 생성
    uint8_t balance_mask = 0;
    for (int i = 0; i < num_cells; i++) {
        float delta = voltages[i] - min_voltage;
        if (delta > BALANCE_THRESHOLD_MV) {
            balance_mask |= (1 << i);
            printf("Cell %d: %.0fmV (delta: %.0fmV) → BALANCE\n",
                   i + 1, voltages[i], delta);
        }
    }
    
    // 밸런싱 적용
    ad7280a_write_reg(0, AD7280A_REG_CELL_BALANCE, balance_mask);
}
```

## 실행 결과

```
=== Balance Cycle 1 ===
Cell 1: 4180mV (delta: 68mV) → BALANCE
Cell 2: 4150mV (delta: 38mV) → BALANCE
Cell 3: 4200mV (delta: 88mV) → BALANCE
Cell 4: 4112mV (delta: 0mV)
Cell 5: 4170mV (delta: 58mV) → BALANCE
Cell 6: 4160mV (delta: 48mV) → BALANCE
Balance mask: 0x37

... 30분 후 ...

=== Balance Cycle 50 ===
Cell 1: 4120mV (delta: 8mV)
Cell 2: 4118mV (delta: 6mV)
Cell 3: 4125mV (delta: 13mV) → BALANCE
Cell 4: 4112mV (delta: 0mV)
Cell 5: 4119mV (delta: 7mV)
Cell 6: 4117mV (delta: 5mV)
Balance mask: 0x04
```

## 주의사항

### 1. 충전 중에만 밸런싱

방전 중에 밸런싱하면 약한 셀이 더 빨리 죽는다.

```c
typedef enum {
    BMS_STATE_IDLE,
    BMS_STATE_CHARGING,
    BMS_STATE_DISCHARGING
} bms_state_t;

void balance_control(bms_state_t state) {
    if (state == BMS_STATE_CHARGING) {
        simple_balance(voltages, 6);
    } else {
        // 밸런싱 OFF
        ad7280a_write_reg(0, AD7280A_REG_CELL_BALANCE, 0x00);
    }
}
```

### 2. 온도 모니터링

밸런싱 저항이 뜨거워진다. 서미스터로 온도 감시 필수.

```c
#define BALANCE_TEMP_LIMIT_C    60

void balance_with_temp_check(void) {
    float temp = read_thermistor();
    
    if (temp > BALANCE_TEMP_LIMIT_C) {
        ad7280a_write_reg(0, AD7280A_REG_CELL_BALANCE, 0x00);
        printf("Balance suspended: temp %.1f°C\n", temp);
    }
}
```

### 3. 인접 셀 동시 밸런싱 주의

인접한 셀을 동시에 밸런싱하면 발열이 집중된다. 체커보드 패턴 권장.

```c
// 홀수 셀만 먼저
balance_mask = original_mask & 0x15;  // Cell 1, 3, 5
apply_balance(balance_mask);
sleep_s(60);

// 짝수 셀
balance_mask = original_mask & 0x2A;  // Cell 2, 4, 6
apply_balance(balance_mask);
```

## 삽질 포인트 정리

1. **CB Timer 설정 필수** - 0이면 스위치 즉시 OFF
2. **측정 시 밸런싱 중단** - 전압 강하 방지
3. **충전 중에만 밸런싱** - 방전 중 금지
4. **온도 감시** - 과열 보호
5. **방전 저항 선정** - 발열 vs 속도 균형

다음 글에서는 더 스마트한 밸런싱 알고리즘을 다룬다.

---

## 참고 자료

- [AD7280A Datasheet - Cell Balancing (p.23-24)](https://www.analog.com/media/en/technical-documentation/data-sheets/ad7280a.pdf)
- [AN-1012 - Passive Cell Balancing](https://www.analog.com/en/resources/technical-articles/passive-cell-balancing.html)
