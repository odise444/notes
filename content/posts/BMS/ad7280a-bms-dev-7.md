---
title: "AD7280A BMS 개발기 #7 - 드디어 셀 전압이 읽힌다"
date: 2024-12-07
draft: false
tags: ["AD7280A", "BMS", "STM32", "전압측정"]
categories: ["BMS 개발"]
series: ["AD7280A BMS 개발"]
summary: "CRC 문제 해결하고 드디어 셀 전압을 읽었다. 실제 값이랑 비교해봤다."
---

CRC가 드디어 맞았다. 이제 셀 전압을 읽어볼 차례다.

---

셀 전압 읽기 순서:

1. 변환 시작 명령
2. 변환 완료 대기 (~500us)
3. 결과 레지스터 읽기

```c
void AD7280A_StartConversion(void) {
    // CONTROL_HB에 CONV_START 비트 세팅
    uint8_t ctrl = 0x80;  // Bit 7 = CONV_START
    AD7280A_WriteReg(REG_CONTROL_HB, ctrl);
}

uint16_t AD7280A_ReadCellVoltage(uint8_t cell) {
    // cell 0~5
    uint32_t resp = AD7280A_ReadReg(REG_CELL_VTG1 + cell);
    uint16_t raw = (resp >> 13) & 0x0FFF;  // 12비트 추출
    return raw;
}
```

---

처음 읽었을 때 값이 이상했다.

```
Cell 0: 0x8A3 (2211 decimal)
mV 변환: 2211/4096 * 5000 + 1000 = 3699 mV
```

3.7V? 근데 실제로 멀티미터로 재보니 3.25V였다.

변환 공식이 틀렸나 싶었는데, 데이터시트를 다시 보니까 수식이 좀 달랐다.

```c
// 올바른 변환
// Vcell = (raw + 512) * 1.0mV
float raw_to_mv(uint16_t raw) {
    return (float)(raw + 512);
}
```

이렇게 하니까 3.25V 근처가 나왔다.

---

6셀 전체를 읽어서 찍어봤다.

![셀 전압 측정 결과](/imgs/ad7280a-bms-dev-07-1.png)

실제 측정값과 5mV 이내로 맞았다. 드디어 된다!

---

24셀 전체를 읽으려면 4개 디바이스를 순서대로 읽어야 한다.

```c
void AD7280A_ReadAllCells(uint16_t *cells) {
    // 모든 디바이스에 동시 변환 명령
    AD7280A_WriteAll(REG_CONTROL_HB, 0x80);
    HAL_Delay(1);  // 변환 대기
    
    // 각 디바이스에서 6셀씩 읽기
    for (int dev = 0; dev < 4; dev++) {
        for (int cell = 0; cell < 6; cell++) {
            cells[dev * 6 + cell] = AD7280A_ReadCell(dev, cell);
        }
    }
}
```

나중에 알았는데 이렇게 하면 느리다. 연속 읽기(Read All) 모드를 쓰면 훨씬 빠르다. 그건 나중에 최적화하기로.

---

실제 셀에 연결해서 테스트했다.

![6셀 테스트](/imgs/6cells.jpg)

일단 6셀로 단일 디바이스 검증하고, 괜찮으면 확장했다.

---

다음 글에서 자가진단 기능을 테스트한다.

[#8 - 자가진단](/posts/bms/ad7280a-bms-dev-8/)
