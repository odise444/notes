---
title: "AD7280A BMS 개발기 #6 - CRC 때문에 3일 날렸다"
date: 2024-12-06
draft: false
tags: ["AD7280A", "BMS", "STM32", "CRC"]
categories: ["BMS 개발"]
series: ["AD7280A BMS 개발"]
summary: "CRC-8 다항식 0x2F. 인터넷에 있는 코드 다 안 됐다."
---

AD7280A는 통신 무결성을 위해 CRC를 쓴다. 

쓰기 프레임은 CRC-8 3비트, 읽기 응답은 CRC-8 4비트. 왜 다르게 해놨는지 모르겠다.

다항식은 0x2F라고 데이터시트에 써있다.

---

처음엔 인터넷에서 CRC-8 코드를 복붙했다.

```c
// 인터넷에서 퍼온 코드 (안 됨)
uint8_t crc8(uint8_t *data, int len) {
    uint8_t crc = 0;
    while (len--) {
        crc ^= *data++;
        for (int i = 0; i < 8; i++) {
            if (crc & 0x80)
                crc = (crc << 1) ^ 0x2F;
            else
                crc <<= 1;
        }
    }
    return crc;
}
```

안 됐다. 통신이 계속 실패했다.

---

데이터시트를 다시 읽었다.

"The CRC is calculated on bits D31 to D11 of the write frame."

bit[31:11]이다. 21비트. 그러니까 8비트씩 계산하면 안 되고, 21비트 값을 통째로 CRC 계산해야 한다.

```c
// 두 번째 시도 (또 안 됨)
uint8_t AD7280A_CalcCRC(uint32_t data) {
    uint8_t crc = 0;
    // bit 31부터 11까지, 21비트
    for (int i = 20; i >= 0; i--) {
        uint8_t bit = (data >> (i + 11)) & 0x01;
        if ((crc ^ bit) & 0x01) {
            crc = (crc >> 1) ^ 0x2F;
        } else {
            crc >>= 1;
        }
    }
    return crc & 0x07;  // 3비트만
}
```

이것도 안 됐다.

---

3일째 되던 날, 데이터시트 Table 9를 발견했다.

CRC 계산 예시가 나와있었다! 이걸 왜 이제 봤지.

```
Input: 0x038101E
Expected CRC: 0x02
```

내 코드로 계산하면 0x05가 나왔다. 틀렸다.

---

결국 데이터시트에 있는 pseudo-code를 그대로 옮겼다.

```c
static const uint8_t crc_table[256] = {
    // 미리 계산된 테이블
    0x00, 0x2F, 0x5E, 0x71, 0xBC, 0x93, 0xE2, 0xCD,
    0x57, 0x78, 0x09, 0x26, 0xEB, 0xC4, 0xB5, 0x9A,
    // ... (생략)
};

uint8_t AD7280A_CalcCRC(uint32_t val) {
    uint8_t crc = 0x00;
    
    crc = crc_table[((val >> 16) & 0xFF) ^ crc];
    crc = crc_table[((val >> 8) & 0xFF) ^ crc];
    crc = crc_table[(val & 0xFF) ^ crc];
    
    return crc;
}
```

이제 됐다.

---

교훈:
1. CRC는 데이터시트 예시로 검증하자
2. 인터넷 코드 복붙하지 말고 스펙 확인하자
3. 3일 고생하기 전에 예시 테이블부터 찾자

---

이제 드라이버 기본은 됐다. 다음 글에서 셀 전압을 실제로 읽어본다.

[#7 - 셀 전압 읽기](/posts/bms/ad7280a-bms-dev-7/)
