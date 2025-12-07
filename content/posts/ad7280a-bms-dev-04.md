---
title: "AD7280A BMS 개발 삽질기 #4 - VIN5/VIN6에서 0만 읽힌다"
date: 2024-12-07
draft: false
tags: ["BMS", "AD7280A", "임베디드", "배터리", "삽질"]
categories: ["BMS 개발"]
summary: "6셀 중 4개는 잘 읽히는데, Cell 5와 Cell 6만 0이 나온다. 데이터시트를 다시 읽어야 했다."
---

## 지난 글 요약

[지난 글](/posts/ad7280a-bms-dev-03/)에서 SPI 통신에 성공했다. 드디어 셀 전압이 읽힌다! 근데 뭔가 이상하다.

## 증상

Cell 1~4는 정상적으로 약 4V씩 읽히는데, **Cell 5와 Cell 6이 0V**다.

```
Cell 1: 4012 mV ✓
Cell 2: 4008 mV ✓
Cell 3: 4015 mV ✓
Cell 4: 4010 mV ✓
Cell 5: 0 mV    ✗
Cell 6: 0 mV    ✗
```

저항 분배기 연결을 눈으로 확인했는데 문제없어 보인다. 멀티미터로 찍어봐도 VIN5, VIN6 노드에 전압이 잘 들어가고 있다.

그럼 뭐가 문제지?

## Self-Test 해보기

ADC 자체가 고장났나 싶어서 Self-Test를 돌려봤다.

AD7280A는 내부 1.2V 밴드갭 레퍼런스로 Self-Test를 할 수 있다. Control 레지스터에서 `CONV_INPUT` 비트를 `11`로 설정하면 된다.

```c
// Self-Test 모드 활성화
uint8_t ctrl_hb = (0x03 << 6) |  // CONV_INPUT = 11 (Self-Test)
                  (0x00 << 4) |  // CONV_RST
                  (0x03 << 1);   // CONV_AVG = 8회 평균
                  
uint32_t cmd = ad7280a_crc_write(
    ((uint32_t)AD7280A_REG_CONTROL_HB << 21) |
    ((uint32_t)ctrl_hb << 13) |
    (1 << 12)
);
ad7280a_transfer_32bits(cmd);
```

Self-Test 결과:
```
Self-Test Cell 1: 985 ✓
Self-Test Cell 2: 982 ✓
Self-Test Cell 3: 987 ✓
Self-Test Cell 4: 984 ✓
Self-Test Cell 5: 983 ✓
Self-Test Cell 6: 986 ✓
```

**정상 범위(970~990) 안에 다 들어온다.** ADC는 멀쩡하다. 그럼 외부 입력 경로 문제인데...

## 데이터시트를 다시 읽다

뭔가 놓친 게 있을 것 같아서 데이터시트를 다시 읽었다. "Connection of Fewer Than Six Voltage Cells" 섹션에서 답을 찾았다.

> **VIN6 입력의 전압은 항상 VDD 공급 핀의 전압보다 크거나 같아야 합니다.**

뭔 소리지? AD7280A의 VDD는 VIN6에서 내부적으로 공급받는 구조다. 그래서 VIN6 < VDD면 전원 자체가 불안정해진다.

내 테스트 환경에서 문제를 발견했다.

## 문제 발견

저항 분배기를 24V로만 테스트하고 있었다!

```
24V ─── VIN6 (= VDD)
     │
    4kΩ
     │
20V ─── VIN5
     │
    4kΩ
     │
16V ─── VIN4
     ...
```

24V면 각 셀당 4V, 여기까진 맞다. 근데 문제는 **VIN6 = 24V = VDD**인 상황에서 내부 LDO 등의 전압 강하로 실제 VDD가 VIN6보다 살짝 낮아질 수 있다.

거기다 내가 사용한 저항 공차(±5%)도 영향을 줬다. VIN6 노드 전압이 VDD보다 낮아지는 순간이 생기면서 Cell 5, Cell 6 채널이 이상해진 것이다.

## 해결 방법

### 방법 1: 테스트 전압 높이기

24V 대신 **30V 이상**으로 테스트한다. VDD 마진이 충분해지면서 문제 해결.

### 방법 2: 4셀 구성이라면 VIN6를 VDD에 직결

4셀만 사용하는 실제 애플리케이션에서는:

```
배터리 4S+ ─── VIN6 ─── VIN5 (쇼트)
                │
               VDD
```

VIN6와 VIN5를 쇼트하고, VIN4부터 실제 셀을 연결한다. 이렇게 하면 VIN6 ≥ VDD 조건이 항상 만족된다.

### 방법 3: Acquisition Time 늘리기

입력 임피던스가 높으면 셋틀링 시간이 부족할 수 있다. Control 레지스터에서 ACQ_TIME을 늘려본다.

```c
// Acquisition Time 설정
#define AD7280A_ACQ_TIME_400ns   0  // 기본값
#define AD7280A_ACQ_TIME_800ns   1
#define AD7280A_ACQ_TIME_1200ns  2
#define AD7280A_ACQ_TIME_1600ns  3  // 최대

uint8_t ctrl_lb = AD7280A_CTRL_LB_MUST_SET | 
                  (AD7280A_ACQ_TIME_1600ns << 5) |  // 1600ns로 설정
                  AD7280A_CTRL_LB_LOCK_DEV_ADDR |
                  AD7280A_CTRL_LB_DAISY_CHAIN_RB_EN;
```

## Alert에서 미사용 채널 제외

4셀 구성에서 VIN5, VIN6를 쇼트하면 이 채널들은 항상 0V로 읽힌다. 그럼 Alert 기능에서 과방전 알람이 계속 뜬다.

이걸 방지하려면 Alert Register에서 미사용 채널을 제외해야 한다.

```c
// Linux 드라이버 참고
#define AD7280A_ALERT_REMOVE_VIN5        (1 << 2)
#define AD7280A_ALERT_REMOVE_VIN4_VIN5   (2 << 2)

// 4셀 구성: VIN4, VIN5 Alert 제외
uint8_t alert_setting = AD7280A_ALERT_REMOVE_VIN4_VIN5;
```

## 트러블슈팅 체크리스트

VIN5/VIN6에서 0이 읽힐 때:

1. **Self-Test 먼저** - ADC 내부 동작 확인 (970~990 정상)
2. **VIN6 ≥ VDD 확인** - 멀티미터로 VIN6 핀 전압 측정
3. **저항 분배기 점검** - 연결 불량, 저항값 확인
4. **Acquisition Time** - 1600ns로 늘려보기
5. **솔더링 확인** - LQFP 핀 1(VIN6), 핀 3(VIN5)

## 삽질 포인트 정리

1. **VIN6 ≥ VDD 필수** - 데이터시트 안 읽으면 모름
2. **Self-Test 활용** - ADC 문제인지 외부 문제인지 구분
3. **4셀 미만 구성** - VIN6, VIN5 쇼트하고 Alert에서 제외

다음 글에서는 Acquisition Time 설정 삽질을 다룬다.

---

## 참고 자료

- [AD7280A Datasheet - Connection of Fewer Than Six Voltage Cells (p.22)](https://www.analog.com/media/en/technical-documentation/data-sheets/ad7280a.pdf)
- [Linux AD7280A IIO Driver](https://wiki.analog.com/resources/tools-software/linux-drivers/iio-adc/ad7280a)
