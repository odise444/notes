---
title: "AD7280A BMS 개발기 #27 - EMC 대응, 노이즈와의 전쟁"
date: 2024-02-10
draft: false
tags: ["AD7280A", "BMS", "STM32", "EMC", "노이즈"]
categories: ["BMS 개발"]
series: ["AD7280A BMS 개발"]
summary: "실험실에서 잘 되던 게 모터 옆에 가면 오작동한다. EMC 문제."
---

모터 돌리면 셀 전압이 흔들린다.

```
모터 OFF: 3320mV ±2mV
모터 ON:  3320mV ±50mV
```

±50mV면 알람 오작동 위험.

---

## 노이즈 원인

1. **전도 노이즈**: 전원 라인 타고 들어옴
2. **방사 노이즈**: 공중으로 날아옴
3. **그라운드 루프**: GND 전위차

모터 드라이버(인버터)가 수십 kHz로 스위칭하면서 노이즈 뿌린다.

---

## 대책 1: 전원 필터

BMS 전원 입력에 LC 필터:

```
Vin ──[L]──┬── Vcc
           │
          [C]
           │
          GND
```

```c
// 부품 값
L = 10uH
C = 100uF + 0.1uF
```

공통 모드 초크도 추가하면 더 좋다.

---

## 대책 2: SPI 라인 보호

SPI 신호선에 페라이트 비드 + RC 필터:

```
MCU ──[FB]──[33Ω]──┬── AD7280A
                   │
                 [100pF]
                   │
                  GND
```

고주파 노이즈 컷.

---

## 대책 3: 아날로그/디지털 분리

PCB에서 AGND와 DGND 분리.

```
      ┌─── AGND (아날로그)
      │
GND ──┤    한 점에서만 연결
      │
      └─── DGND (디지털)
```

MCU 클럭 노이즈가 ADC에 영향 안 가게.

---

## 대책 4: 쉴딩

케이스를 금속으로. 또는 BMS 보드에 쉴드 캔.

커넥터 부분이 취약. 페라이트 코어 끼우면 효과 있다.

---

## 대책 5: 소프트웨어 필터

하드웨어로 다 못 잡으면 소프트웨어로.

```c
// 이동 평균
#define FILTER_SIZE 8
uint16_t filter_buf[24][FILTER_SIZE];
int filter_idx = 0;

uint16_t Filter_CellVoltage(int cell, uint16_t raw) {
    filter_buf[cell][filter_idx] = raw;
    
    uint32_t sum = 0;
    for (int i = 0; i < FILTER_SIZE; i++) {
        sum += filter_buf[cell][i];
    }
    return sum / FILTER_SIZE;
}
```

노이즈 스파이크 제거:

```c
// 중앙값 필터
uint16_t Filter_Median(uint16_t *buf, int size) {
    // 정렬 후 중앙값 반환
    // 튀는 값 제거에 효과적
}
```

---

## 테스트 결과

대책 적용 후:

```
모터 OFF: 3320mV ±2mV
모터 ON:  3320mV ±8mV
```

±50mV → ±8mV. 충분히 안정.

---

## EMC 체크리스트

```
[전원]
□ LC 필터
□ 공통 모드 초크
□ 벌크 캡 + 세라믹 캡 병렬

[신호선]
□ 페라이트 비드
□ RC 필터
□ 시리즈 저항

[PCB]
□ AGND/DGND 분리
□ 그라운드 플레인
□ 짧은 배선

[기구]
□ 금속 케이스
□ 커넥터 페라이트
□ 케이블 쉴드
```

---

다음은 양산 검사 지그.

[#28 - 양산 검사 지그](/posts/bms/ad7280a-bms-dev-28/)
