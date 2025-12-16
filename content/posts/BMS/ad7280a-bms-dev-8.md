---
title: "AD7280A BMS 개발기 #8 - 자가진단 기능"
date: 2024-12-08
draft: false
tags: ["AD7280A", "BMS", "STM32", "자가진단"]
categories: ["BMS 개발"]
series: ["AD7280A BMS 개발"]
summary: "IC가 고장났는지 알 수 있는 Self-test 기능이 있다."
---

BMS에서 중요한 게 "IC가 제대로 동작하는지" 확인하는 거다.

AD7280A에는 Self-test 기능이 있다. 내부 기준 전압을 측정해서 IC 상태를 확인할 수 있다.

---

Self-test 방법:

1. CONTROL_HB에 Self-test 모드 설정
2. 변환 시작
3. 결과 읽기
4. 예상 범위와 비교

```c
typedef struct {
    uint16_t self_test;
    uint16_t vref_2v;
    uint16_t vref_1v;
} AD7280A_SelfTest_t;

bool AD7280A_RunSelfTest(AD7280A_SelfTest_t *result) {
    // Self-test 모드 설정
    AD7280A_WriteReg(REG_CONTROL_HB, 0x82);  // CONV + SELF_TEST
    HAL_Delay(1);
    
    // 결과 읽기
    result->self_test = AD7280A_ReadReg(REG_SELF_TEST);
    
    // 2V, 1V 기준 전압 확인
    result->vref_2v = AD7280A_ReadReg(REG_AUX_ADC1);
    result->vref_1v = AD7280A_ReadReg(REG_AUX_ADC2);
    
    // 범위 확인
    // Self-test는 대략 4V (raw 3500~3700)
    // 2V ref는 대략 2V (raw 1400~1600)
    // 1V ref는 대략 1V (raw 400~600)
    
    if (result->self_test < 3500 || result->self_test > 3700)
        return false;
    if (result->vref_2v < 1400 || result->vref_2v > 1600)
        return false;
    if (result->vref_1v < 400 || result->vref_1v > 600)
        return false;
    
    return true;
}
```

---

처음에 Self-test가 계속 실패했다.

알고 보니 Self-test 변환 시간이 일반 변환보다 길다. 1ms 대기로 부족해서 2ms로 늘렸더니 됐다.

---

Self-test는 부팅할 때 한 번 하고, 이후에는 주기적으로 (1시간에 한 번 정도) 하면 된다.

실패하면 IC 불량이거나 배선 문제다. 이 경우 BMS 동작을 중단하고 알람을 띄워야 한다.

```c
void BMS_Init(void) {
    AD7280A_Init();
    
    AD7280A_SelfTest_t st;
    if (!AD7280A_RunSelfTest(&st)) {
        // 치명적 오류
        Error_Handler();
    }
}
```

---

다음 글에서 온도 센서(보조 ADC) 읽기를 다룬다.

[#9 - 온도 센서](/posts/bms/ad7280a-bms-dev-9/)
