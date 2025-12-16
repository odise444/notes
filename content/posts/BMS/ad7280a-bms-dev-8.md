---
title: "AD7280A BMS 개발기 #8 - 자가진단, IC가 살아있나?"
date: 2024-01-22
draft: false
tags: ["AD7280A", "BMS", "STM32", "Self-test", "진단"]
categories: ["BMS 개발"]
series: ["AD7280A BMS 개발"]
summary: "셀 전압은 읽히는데 이게 맞는 건지 모르겠다. Self-test로 확인해보자."
---

셀 전압이 읽히긴 하는데, IC가 제대로 동작하는 건지 확인이 필요했다.

AD7280A에 Self-test 기능이 있다.

---

## Self-test란

내부 기준 전압으로 ADC를 테스트한다. 

- 셀 입력을 내부 기준 전압에 연결
- 변환 실행
- 결과가 예상 범위 내인지 확인

외부 연결 상태와 무관하게 ADC 자체가 정상인지 확인할 수 있다.

---

## Self-test 실행

CTRL_HB에서 Self-test 모드 설정:

```c
void AD7280A_SelfTest(uint16_t *results) {
    // Self-test 모드 설정 + 변환 시작
    uint32_t frame = AD7280A_BuildWriteFrame(
        0x1F,              // 브로드캐스트
        AD7280A_CTRL_HB,
        0x23               // Self-test 모드 + 변환 시작
    );
    AD7280A_Transfer(frame);
    
    HAL_Delay(2);  // 변환 대기
    
    // 결과 읽기
    for (int i = 0; i < 6; i++) {
        results[i] = AD7280A_ReadCellVoltage(0, i);
    }
}
```

---

## 예상 결과

Self-test 시 내부 기준 전압 값이 읽혀야 한다.

데이터시트에 따르면:

```
Self-test 1: 약 1V
Self-test 2: 약 2V  
Self-test 3: 약 3V
Self-test 4: 약 4V
```

각 셀 채널마다 다른 기준 전압이 연결된다.

---

## 테스트 결과 검증

```c
bool AD7280A_VerifySelfTest(uint16_t *results) {
    // 예상 범위 (±100mV 허용)
    const uint16_t expected[] = {
        102,   // ~1V → raw 약 0
        1126,  // ~2.1V → raw 약 1126
        2150,  // ~3.1V → raw 약 2150
        3174,  // ~4.1V → raw 약 3174
        3174,  // Cell 5도 4V 근처
        3174   // Cell 6도 4V 근처
    };
    
    for (int i = 0; i < 6; i++) {
        int diff = abs(results[i] - expected[i]);
        if (diff > 100) {  // 100 LSB 허용
            return false;
        }
    }
    return true;
}
```

실제로 돌려보니까 조금씩 다르더라. IC 개체 차이가 있는 것 같다.

---

## Open Wire 검출

Self-test 외에 Open Wire 검출 기능도 있다.

셀 연결선이 끊어졌는지 확인하는 기능. 풀업/풀다운 전류를 흘려서 비정상적인 전압이 읽히면 단선.

```c
bool AD7280A_CheckOpenWire(uint8_t device) {
    // Pull-up 테스트
    AD7280A_Write(device, AD7280A_CTRL_HB, 0x43);  // Open wire detect
    HAL_Delay(1);
    
    uint16_t pu_result = AD7280A_ReadCellVoltage(device, 0);
    
    // Pull-down 테스트
    AD7280A_Write(device, AD7280A_CTRL_HB, 0x63);
    HAL_Delay(1);
    
    uint16_t pd_result = AD7280A_ReadCellVoltage(device, 0);
    
    // 비정상적인 차이가 있으면 단선
    if (abs(pu_result - pd_result) > 500) {
        return true;  // Open wire detected
    }
    return false;
}
```

이건 현장에서 유용했다. 셀 탭 연결 불량 찾을 때.

---

## 부팅 시 진단

시스템 부팅할 때 Self-test 돌려서 IC 상태 확인:

```c
void BMS_Init(void) {
    AD7280A_Init();
    
    // Self-test
    uint16_t test_results[6];
    AD7280A_SelfTest(test_results);
    
    if (!AD7280A_VerifySelfTest(test_results)) {
        Error_Handler();  // IC 이상
    }
    
    // 정상이면 계속
    printf("AD7280A Self-test OK\n");
}
```

---

## 정리

- Self-test: 내부 기준 전압으로 ADC 확인
- Open Wire: 셀 연결선 단선 검출
- 부팅 시 한 번씩 돌려주면 좋다

IC가 살아있는지 확인하는 용도. 실제 셀 전압 정확도는 캘리브레이션이 필요하다.

---

다음은 온도 측정. AUX ADC로 NTC 읽기.

[#9 - 온도 측정](/posts/bms/ad7280a-bms-dev-9/)
