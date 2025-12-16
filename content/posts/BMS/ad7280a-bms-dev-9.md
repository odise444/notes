---
title: "AD7280A BMS 개발기 #9 - 온도 측정, NTC 연결하기"
date: 2024-01-23
draft: false
tags: ["AD7280A", "BMS", "STM32", "온도", "NTC"]
categories: ["BMS 개발"]
series: ["AD7280A BMS 개발"]
summary: "셀 전압만 재면 안 된다. 배터리 온도도 알아야 한다. AUX ADC로 NTC 읽기."
---

BMS에서 온도 측정은 필수다. 과열 보호, 저온 충전 제한 등.

AD7280A에 AUX ADC 채널이 있다. 여기에 NTC 연결하면 온도 측정 가능.

---

## AUX ADC

AD7280A에 AUX ADC 채널이 2개 있다. 

- AUX_ADC_1 (0x06)
- AUX_ADC_2 (0x07)

셀 전압이랑 같은 12비트 ADC. 전압 범위도 1V~5V 동일.

---

## NTC 연결

NTC(Negative Temperature Coefficient) 서미스터를 분압 회로로 연결:

```
VDD (5V)
   │
   ├──[10kΩ]──┬── AUX_IN
   │          │
   │        [NTC]
   │          │
   └──────────┴── GND
```

NTC 저항이 온도에 따라 변하니까 분압 전압이 변한다.

```
저온(0°C): NTC 저항 높음 → 분압 전압 낮음
고온(60°C): NTC 저항 낮음 → 분압 전압 높음
```

---

## AUX ADC 읽기

CTRL_HB에서 AUX 변환 모드 설정:

```c
uint16_t AD7280A_ReadAuxADC(uint8_t device, uint8_t channel) {
    // AUX 변환 시작
    uint32_t frame = AD7280A_BuildWriteFrame(
        device,
        AD7280A_CTRL_HB,
        0x43  // AUX 변환 + 시작
    );
    AD7280A_Transfer(frame);
    
    HAL_Delay(2);
    
    // 결과 읽기
    uint8_t reg = AD7280A_AUX_ADC1 + channel;
    return AD7280A_ReadCellVoltage(device, reg);
}
```

---

## 전압 → 온도 변환

NTC 특성에 따라 룩업 테이블 사용:

```c
// 10kΩ NTC (B=3950) + 10kΩ 분압저항 기준
// ADC raw → 온도(0.1°C 단위)
const int16_t ntc_table[41] = {
    // raw값 100 단위로 샘플링
    // index 0 = raw 0 ~ 99
    -200,  // 0: -20°C
    -100,  // 1: -10°C
    0,     // 2: 0°C
    50,    // 3: 5°C
    100,   // 4: 10°C
    150,   // 5: 15°C
    200,   // 6: 20°C
    250,   // 7: 25°C
    300,   // 8: 30°C
    350,   // 9: 35°C
    400,   // 10: 40°C
    // ... 계속
};

int16_t AD7280A_RawToTemperature(uint16_t raw) {
    int index = raw / 100;
    if (index >= 40) return 800;  // 80°C 이상
    if (index < 0) return -300;   // -30°C 이하
    
    // 선형 보간
    int16_t t1 = ntc_table[index];
    int16_t t2 = ntc_table[index + 1];
    int frac = raw % 100;
    
    return t1 + (t2 - t1) * frac / 100;
}
```

---

## 문제: NTC B값

처음에 아무 NTC나 달았다가 온도가 6도씩 틀렸다.

NTC마다 B값이 다르다. B값이 다르면 저항-온도 특성이 다르다.

```
10kΩ NTC (B=3380) @ 50°C: 4.16kΩ
10kΩ NTC (B=3950) @ 50°C: 3.60kΩ
```

룩업 테이블을 B=3950 기준으로 만들었는데 B=3380 NTC를 달았던 거다.

B값 맞는 NTC로 교체하거나, 테이블을 새로 만들어야 한다.

---

## 실측 테이블 만들기

정확한 테이블을 원하면 직접 측정:

1. NTC를 항온조에 넣음
2. 0°C, 10°C, 20°C, ... 각 온도에서 ADC 값 기록
3. 테이블 생성

나는 그냥 데이터시트 공식으로 계산했다:

```c
// Steinhart-Hart 방정식
float NTC_CalcTemperature(float resistance) {
    const float B = 3950.0f;
    const float R25 = 10000.0f;  // 25°C에서 저항
    const float T25 = 298.15f;   // 25°C in Kelvin
    
    float t_kelvin = 1.0f / (1.0f/T25 + (1.0f/B) * log(resistance/R25));
    return t_kelvin - 273.15f;  // Celsius
}
```

---

## 온도 보호

온도 측정 목적:

1. 과열 보호: 60°C 이상이면 충방전 차단
2. 저온 충전 제한: 0°C 이하면 충전 금지
3. 온도 보상: SOC 추정 시 온도 보정

```c
void BMS_CheckTemperature(void) {
    int16_t temp = BMS_GetTemperature();
    
    if (temp > 600) {  // 60°C
        BMS_SetFault(FAULT_OVERTEMP);
        BMS_DisableOutput();
    }
    else if (temp < 0 && g_bms.state == STATE_CHARGING) {
        BMS_StopCharging();  // 저온 충전 금지
    }
}
```

---

## 정리

- AUX ADC로 NTC 전압 읽기
- 분압 회로 + 룩업 테이블
- NTC B값 확인 필수
- 과열/저온 보호에 사용

---

Part 3 측정편 끝.

다음은 밸런싱. 셀마다 전압이 다르면 맞춰줘야 한다.

[#10 - 패시브 밸런싱](/posts/bms/ad7280a-bms-dev-10/)
