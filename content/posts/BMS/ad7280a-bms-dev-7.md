---
title: "AD7280A BMS 개발기 #7 - 셀 전압 읽기, 드디어 측정"
date: 2024-01-21
draft: false
tags: ["AD7280A", "BMS", "STM32", "전압측정", "ADC"]
categories: ["BMS 개발"]
series: ["AD7280A BMS 개발"]
summary: "CRC 문제 해결하고 드디어 셀 전압을 읽었다. 뭔가 숫자가 나온다!"
---

CRC 문제 해결하고 이제 진짜 셀 전압을 읽어보자.

![6셀 테스트](/imgs/6cells.jpg)

일단 6셀로 테스트.

---

## 변환 과정

AD7280A로 셀 전압 읽는 순서:

1. 변환 시작 명령
2. 변환 완료 대기 (~1ms)
3. 결과 읽기

간단하다.

---

## 변환 시작

CTRL_HB 레지스터에 변환 시작 명령:

```c
void AD7280A_StartConversion(void) {
    // 모든 디바이스에 변환 시작
    uint32_t frame = AD7280A_BuildWriteFrame(
        0x1F,              // 브로드캐스트
        AD7280A_CTRL_HB,
        0x03               // 셀 전압 변환, 즉시 시작
    );
    AD7280A_Transfer(frame);
}
```

0x1F는 브로드캐스트 주소. 모든 디바이스가 동시에 변환 시작.

---

## 결과 읽기

변환 끝나면 CELL_V1 ~ CELL_V6 레지스터에 값이 들어간다.

```c
uint16_t AD7280A_ReadCellVoltage(uint8_t device, uint8_t cell) {
    // 읽기 명령 전송
    uint32_t cmd = AD7280A_BuildReadFrame(device, AD7280A_CELL_V1 + cell);
    AD7280A_Transfer(cmd);
    
    // 응답 수신
    uint32_t response = AD7280A_Transfer(0x00000000);  // 더미 전송
    
    // CRC 체크
    if (!AD7280A_CheckCRC(response)) {
        return 0xFFFF;  // 에러
    }
    
    // 12비트 데이터 추출 (bit 23~12)
    return (response >> 11) & 0x0FFF;
}
```

---

## mV 변환

12비트 raw 값을 mV로 변환:

```c
float AD7280A_RawToMillivolt(uint16_t raw) {
    // 공식: V = (raw * 0.976mV) + 1000mV
    return raw * 0.976f + 1000.0f;
}
```

AD7280A는 1V~5V 범위를 측정한다. 0이면 1000mV, 4096이면 약 5000mV.

---

## 전체 셀 읽기

24셀 전체를 읽는 함수:

```c
void AD7280A_ReadAllCells(uint16_t *cells) {
    AD7280A_StartConversion();
    HAL_Delay(2);  // 변환 대기
    
    for (int dev = 0; dev < 4; dev++) {
        for (int cell = 0; cell < 6; cell++) {
            uint16_t raw = AD7280A_ReadCellVoltage(dev, cell);
            cells[dev * 6 + cell] = raw;
        }
    }
}
```

이렇게 하면 cells[0]~cells[23]에 24셀 전압이 들어간다.

---

## 첫 측정 결과

```
Cell  0: 3312 (raw) → 3232mV
Cell  1: 3308 → 3228mV
Cell  2: 3315 → 3235mV
Cell  3: 3310 → 3231mV
Cell  4: 3318 → 3238mV
Cell  5: 3305 → 3225mV
```

![측정 결과](/imgs/ad7280a-bms-dev-07-1.png)

멀티미터로 확인해보니까 대충 맞다. 오차 5mV 정도.

---

## 버스트 읽기

위 코드는 셀마다 SPI 트랜잭션을 한다. 느리다.

AD7280A는 연속 읽기 모드를 지원한다. 변환 한 번 하고 전체 데이터를 줄줄이 읽을 수 있다.

```c
void AD7280A_ReadAllCellsBurst(uint16_t *cells) {
    // 변환 시작
    AD7280A_StartConversion();
    HAL_Delay(2);
    
    // 연속 읽기 모드 설정
    uint32_t cmd = AD7280A_BuildReadFrame(0x1F, AD7280A_CELL_V1);
    AD7280A_Transfer(cmd);
    
    // 24개 연속 수신
    for (int i = 0; i < 24; i++) {
        uint32_t response = AD7280A_Transfer(0x00000000);
        cells[i] = (response >> 11) & 0x0FFF;
    }
}
```

이게 훨씬 빠르다. 50ms → 5ms 정도.

---

## 문제: 간헐적 오류

가끔 이상한 값이 읽혔다. 0이나 4095 같은.

```
Cell 0: 3312
Cell 1: 0      ← 이상
Cell 2: 3315
Cell 3: 4095   ← 이상
```

CRC 에러였다. 노이즈 때문에 통신이 깨지는 경우가 있었다.

CRC 체크 실패하면 다시 읽도록 수정:

```c
uint16_t AD7280A_ReadCellVoltageWithRetry(uint8_t dev, uint8_t cell) {
    for (int retry = 0; retry < 3; retry++) {
        uint16_t raw = AD7280A_ReadCellVoltage(dev, cell);
        if (raw != 0xFFFF) {
            return raw;
        }
    }
    return 0xFFFF;  // 3번 실패
}
```

---

## 정리

- 변환 시작 → 대기 → 읽기
- raw * 0.976 + 1000 = mV
- 버스트 읽기가 빠름
- CRC 에러 시 재시도

---

다음은 자가진단 기능. Self-test로 IC 상태 확인하기.

[#8 - 자가진단](/posts/bms/ad7280a-bms-dev-8/)
