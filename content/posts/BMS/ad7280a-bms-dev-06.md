---
title: "AD7280A BMS 개발 삽질기 #6 - Alert가 계속 울린다"
date: 2024-12-07
draft: false
tags: ["BMS", "AD7280A", "임베디드", "배터리", "삽질"]
categories: ["BMS 개발"]
summary: "4셀만 쓰는데 과방전 알람이 멈추질 않는다. 미사용 채널을 Alert에서 제외해야 했다."
---

## 지난 글 요약

[지난 글](/posts/bms/ad7280a-bms-dev-05/)에서 Acquisition Time 설정을 잡았다. 이제 셀 전압이 안정적으로 읽힌다. 근데 또 문제가 생겼다.

## 증상

4셀 배터리 팩을 연결하고 테스트하는데, **ALERT 핀이 계속 LOW**다.

```
ALERT 상태: LOW (알람 발생!)
Cell 1: 4012 mV ✓
Cell 2: 4008 mV ✓
Cell 3: 4015 mV ✓
Cell 4: 4010 mV ✓
Cell 5: 0 mV     ← 미사용 (쇼트)
Cell 6: 0 mV     ← 미사용 (쇼트)
```

Cell 5, 6은 4셀 구성이라 VIN4에 쇼트해뒀다. 당연히 0V로 읽힌다. 근데 이게 **과방전(Undervoltage) 알람**을 트리거하고 있었다.

## Alert 동작 원리

AD7280A는 각 셀 전압을 Overvoltage/Undervoltage 임계값과 비교한다.

```
Cell Undervoltage Register: 기본값 0x000 (약 1V)
Cell Overvoltage Register: 기본값 0xFFF (약 5V)
```

셀 전압이 Undervoltage 아래로 떨어지면 ALERT 핀이 LOW가 된다. Cell 5, 6이 0V니까 당연히 알람이 뜬다.

## 해결 방법: 미사용 채널 제외

Alert Register의 **D3:D2** 비트로 특정 채널을 Alert 감지에서 제외할 수 있다.

| D3:D2 | 동작 |
|-------|------|
| 00 | 6개 채널 모두 감지 (기본값) |
| 01 | VIN5 제외 |
| 10 | VIN5, VIN4 제외 |
| 11 | Reserved |

4셀 구성이면 Cell 5, 6이 미사용이니까... 어? VIN4, VIN5를 제외하라고?

## 잠깐, 채널 번호가 헷갈린다

AD7280A의 채널 번호가 직관적이지 않다.

```
배터리 연결:
Cell 6 = VIN6 - VIN5
Cell 5 = VIN5 - VIN4
Cell 4 = VIN4 - VIN3
Cell 3 = VIN3 - VIN2
Cell 2 = VIN2 - VIN1
Cell 1 = VIN1 - VIN0
```

4셀 구성에서 VIN6=VIN5=VIN4로 쇼트하면:
- Cell 6 = VIN6 - VIN5 = 0V
- Cell 5 = VIN5 - VIN4 = 0V

그래서 **VIN5 제외(01)**하면 Cell 5, Cell 6 둘 다 Alert에서 빠진다.

## 코드 구현

```c
// Alert Register 주소
#define AD7280A_REG_ALERT    0x13

// Alert 설정 비트
#define AD7280A_ALERT_REMOVE_VIN5        (1 << 2)  // D3:D2 = 01
#define AD7280A_ALERT_REMOVE_VIN4_VIN5   (2 << 2)  // D3:D2 = 10

// 4셀 구성: VIN5 Alert 제외
void ad7280a_configure_alert_4cell(void) {
    uint8_t alert_cfg = AD7280A_ALERT_REMOVE_VIN5;  // 01 << 2 = 0x04
    
    uint32_t cmd = ad7280a_crc_write(
        ((uint32_t)AD7280A_REG_ALERT << 21) |
        ((uint32_t)alert_cfg << 13) |
        (1 << 12)  // All devices
    );
    ad7280a_transfer_32bits(cmd);
}
```

## AUX 채널도 마찬가지

AUX ADC 채널도 Alert 제외 설정이 있다. D1:D0 비트.

| D1:D0 | 동작 |
|-------|------|
| 00 | 모든 AUX 채널 감지 |
| 01 | AUX5 제외 |
| 10 | AUX5, AUX3 제외 |
| 11 | Reserved |

온도 센서를 AUX1~AUX4만 쓴다면 AUX5를 제외해두자.

```c
// 4셀 구성 + AUX5 제외
uint8_t alert_cfg = AD7280A_ALERT_REMOVE_VIN5 |  // D3:D2 = 01
                    (1 << 0);                     // D1:D0 = 01

uint32_t cmd = ad7280a_crc_write(
    ((uint32_t)AD7280A_REG_ALERT << 21) |
    ((uint32_t)alert_cfg << 13) |
    (1 << 12)
);
ad7280a_transfer_32bits(cmd);
```

## Alert Register 전체 구조

| 비트 | 이름 | 설명 |
|------|------|------|
| D7:D4 | Reserved | 0으로 유지 |
| D3:D2 | REMOVE_VIN | 전압 채널 Alert 제외 |
| D1:D0 | REMOVE_AUX | AUX 채널 Alert 제외 |

## Linux 드라이버 참고

Linux IIO 드라이버에도 같은 설정이 있다:

```c
// linux/drivers/iio/adc/ad7280a.c
#define AD7280A_ALERT_REMOVE_VIN5        (1 << 2)
#define AD7280A_ALERT_REMOVE_VIN4_VIN5   (2 << 2)

// Platform data로 설정
.alert_flags = AD7280A_ALERT_REMOVE_VIN5,
```

## 결과

Alert 제외 설정 후:

```
ALERT 상태: HIGH (정상) ✓
Cell 1: 4012 mV ✓
Cell 2: 4008 mV ✓
Cell 3: 4015 mV ✓
Cell 4: 4010 mV ✓
Cell 5: 0 mV (무시됨)
Cell 6: 0 mV (무시됨)
```

드디어 알람이 안 울린다!

## 삽질 포인트 정리

1. **미사용 채널 = 0V = Undervoltage** - 당연한 건데 놓침
2. **채널 번호 혼란** - VIN5 제외하면 Cell 5, 6 둘 다 빠짐
3. **AUX도 확인** - 온도 센서 미사용 채널도 제외

다음 글에서는 데이지체인 구성을 다룬다.

---

## 참고 자료

- [AD7280A Datasheet - Alert Register (p.27, Table 12)](https://www.analog.com/media/en/technical-documentation/data-sheets/ad7280a.pdf)
- [Linux AD7280A IIO Driver](https://wiki.analog.com/resources/tools-software/linux-drivers/iio-adc/ad7280a)
