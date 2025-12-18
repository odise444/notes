---
title: "AD7280A BMS 개발기 #6 - CRC의 늪, 3일간의 삽질"
date: 2024-01-20
draft: false
tags: ["AD7280A", "BMS", "STM32", "CRC"]
categories: ["BMS 개발"]
series: ["AD7280A BMS 개발"]
summary: "CRC 계산이 안 맞아서 3일을 날렸다. 비트 순서가 문제였다."
---

AD7280A는 모든 프레임에 CRC가 들어간다. CRC가 틀리면 명령을 무시한다.

여기서 3일 날렸다.

---

## AD7280A CRC 스펙

- CRC-8
- 다항식: 0x2F (x^8 + x^5 + x^3 + x^2 + x + 1)
- 초기값: 0x00
- 입력 데이터: 프레임의 bit 31~11 (21비트)

데이터시트에 이렇게 써있다. 간단해 보였다.

---

## 첫 번째 시도

```c
uint8_t AD7280A_CalcCRC(uint32_t data) {
    uint8_t crc = 0;
    
    for (int i = 20; i >= 0; i--) {
        uint8_t bit = (data >> i) & 0x01;
        if ((crc ^ bit) & 0x01) {
            crc = (crc >> 1) ^ 0x2F;
        } else {
            crc >>= 1;
        }
    }
    return crc;
}
```

결과: AD7280A가 응답 안 함.

---

## 두 번째 시도

CRC 방향이 잘못됐나? MSB first로 바꿔봤다.

```c
uint8_t AD7280A_CalcCRC(uint32_t data) {
    uint8_t crc = 0;
    
    for (int i = 20; i >= 0; i--) {
        uint8_t bit = (data >> i) & 0x01;
        if ((crc ^ (bit << 7)) & 0x80) {
            crc = (crc << 1) ^ 0x2F;
        } else {
            crc <<= 1;
        }
    }
    return crc;
}
```

결과: 여전히 응답 없음.

---

## 데이터시트 예제 확인

데이터시트에 예제가 있었다.

```
Input: 0x1C0E10 (21bit)
Expected CRC: 0xA7
```

내 함수로 계산하면 0x5E가 나온다. 완전히 다르다.

---

## 결국 찾은 문제

하루 종일 삽질하다가 EngineerZone에서 힌트를 찾았다.

> "The CRC polynomial should be bit-reversed: 0xF4 instead of 0x2F"

다항식을 비트 반전해야 한다고? 0x2F를 뒤집으면 0xF4.

```c
// 0x2F = 0b00101111
// 뒤집으면 0b11110100 = 0xF4
```

그리고 입력 데이터도 비트 순서가 다르다.

---

## 최종 코드

```c
static uint8_t crc_table[256];

void AD7280A_BuildCRCTable(void) {
    for (int i = 0; i < 256; i++) {
        uint8_t crc = i;
        for (int j = 0; j < 8; j++) {
            if (crc & 0x80) {
                crc = (crc << 1) ^ 0x2F;
            } else {
                crc <<= 1;
            }
        }
        crc_table[i] = crc;
    }
}

uint8_t AD7280A_CalcCRC(uint32_t val) {
    // 21비트 데이터를 3바이트로 처리
    uint8_t crc;
    crc = crc_table[(val >> 16) & 0xFF];
    crc = crc_table[crc ^ ((val >> 8) & 0xFF)];
    crc = crc_table[crc ^ (val & 0xFF)];
    return crc;
}
```

룩업 테이블 방식으로 바꿨다. 초기화할 때 테이블 만들어놓고 참조만 하면 된다.

결과: **드디어 응답이 온다!**

---

## 검증

데이터시트 예제로 검증:

```c
AD7280A_BuildCRCTable();
uint8_t crc = AD7280A_CalcCRC(0x1C0E10);
printf("CRC: 0x%02X\n", crc);  // 출력: 0xA7
```

맞다!

---

## 왜 이렇게 헷갈리게 해놨을까

솔직히 데이터시트 설명이 불친절하다.

"Polynomial: 0x2F"라고만 써있고, 구현 방식(반영 여부, 비트 순서)은 안 써있다.

결국 예제 하나 맞춰보는 게 제일 확실하다.

---

## CRC 검증 함수

수신 프레임의 CRC 검증:

```c
bool AD7280A_CheckCRC(uint32_t frame) {
    uint32_t data = frame >> 11;  // bit 31~11 추출
    uint8_t received_crc = (frame >> 3) & 0xFF;
    uint8_t calculated_crc = AD7280A_CalcCRC(data);
    
    return (received_crc == calculated_crc);
}
```

수신할 때마다 CRC 체크해서 통신 오류 검출.

---

## 정리

- 다항식: 0x2F
- 21비트 입력 (bit 31~11)
- 룩업 테이블 방식이 빠르고 편함
- 데이터시트 예제로 꼭 검증

3일 걸렸지만 한번 되고 나니까 그 뒤론 문제없었다.

---

Part 2 드라이버편 끝.

다음은 셀 전압 읽기. 드디어 뭔가 측정한다.

[#7 - 셀 전압 읽기](/posts/bms/ad7280a-bms-dev-7/)
