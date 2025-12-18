---
title: "AD7280A BMS 개발기 #13 - 과전압/저전압 알람 설정"
date: 2024-01-27
draft: false
tags: ["AD7280A", "BMS", "STM32", "알람", "보호"]
categories: ["BMS 개발"]
series: ["AD7280A BMS 개발"]
summary: "셀 전압이 위험 범위 넘으면 알람이 떠야 한다. 임계값 설정하기."
---

밸런싱까지 됐으니 이제 보호 기능. 과충전/과방전 되면 알람 띄우고 차단해야 한다.

AD7280A에 하드웨어 알람 기능이 있다. 임계값 설정해두면 자동으로 Alert 핀이 떨어진다.

---

## 임계값 레지스터

- CELL_OV (0x10): 과전압 임계값
- CELL_UV (0x11): 저전압 임계값

8비트 값. 전압 계산 공식:

```
임계값(mV) = 레지스터값 × 6 + 1000
레지스터값 = (임계값 - 1000) / 6
```

---

## LFP 기준 설정

LiFePO4 기준:
- 과전압: 3.65V (만충)
- 저전압: 2.5V (방전 종지)

약간의 마진을 두고:

```c
// 과전압: 3.7V
#define OV_THRESHOLD_MV  3700
#define OV_REG_VALUE     ((3700 - 1000) / 6)  // = 450

// 저전압: 2.4V  
#define UV_THRESHOLD_MV  2400
#define UV_REG_VALUE     ((2400 - 1000) / 6)  // = 233
```

---

## 설정 코드

```c
void AD7280A_SetAlarmThresholds(uint8_t device) {
    // 과전압 설정
    AD7280A_Write(device, AD7280A_CELL_OV, OV_REG_VALUE);
    
    // 저전압 설정
    AD7280A_Write(device, AD7280A_CELL_UV, UV_REG_VALUE);
}

void BMS_InitAlarms(void) {
    // 모든 디바이스에 설정 (브로드캐스트)
    AD7280A_Write(0x1F, AD7280A_CELL_OV, OV_REG_VALUE);
    AD7280A_Write(0x1F, AD7280A_CELL_UV, UV_REG_VALUE);
}
```

---

## 해상도 문제

6mV 단위라서 정밀하게 못 맞춘다.

```
원하는 값: 3650mV
계산: (3650 - 1000) / 6 = 441.67 → 441
실제 값: 441 × 6 + 1000 = 3646mV
```

4mV 오차. 큰 문제는 아닌데 알고는 있어야 한다.

---

## AUX 임계값

온도 알람용 AUX 임계값도 있다:

- AUX_OV (0x12): AUX 과전압
- AUX_UV (0x13): AUX 저전압

NTC 전압 범위에 맞춰 설정하면 과열/저온 알람으로 쓸 수 있다.

```c
// 과열 (60°C): NTC 전압 약 1.5V
#define AUX_OV_VALUE  ((1500 - 1000) / 6)  // 약 83

// 저온 (0°C): NTC 전압 약 3.8V
#define AUX_UV_VALUE  ((3800 - 1000) / 6)  // 약 466
```

근데 나는 AUX 알람은 안 쓰고 소프트웨어로 처리했다. 더 유연해서.

---

## 테스트

파워서플라이로 셀 하나에 과전압 인가:

```
설정: OV = 3.7V
인가: 3.8V
결과: Alert 핀 Low
```

동작한다.

---

## 정리

- OV/UV 레지스터로 임계값 설정
- 공식: mV = reg × 6 + 1000
- 6mV 해상도
- 브로드캐스트로 전체 설정 가능

---

다음은 Alert 핀 인터럽트 처리.

[#14 - Alert 인터럽트](/posts/bms/ad7280a-bms-dev-14/)
