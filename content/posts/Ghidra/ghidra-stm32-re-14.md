---
title: "Ghidra STM32 역분석 #14 - IAP 명령 코드 분석"
date: 2024-12-13
draft: false
tags: ["Ghidra", "STM32", "리버스엔지니어링", "역분석", "CAN", "프로토콜"]
categories: ["역분석"]
series: ["Ghidra 역분석"]
summary: "0x30, 0x31, 0x32... 각 명령 코드의 의미를 밝혀낸다."
---

명령 디스패처의 switch-case를 분석해서 각 명령의 의미를 파악하자.

---

## 명령 코드 목록

| 요청 | 응답 | 의미 |
|------|------|------|
| 0x30 | 0x40 | Connection 요청/응답 |
| 0x31 | 0x41 | Key 계산/확인 |
| 0x32 | 0x42 | 펌웨어 크기 |
| 0x33 | 0x43 | 데이터 프레임 |
| 0x34 | 0x44 | 쓰기 완료 |
| 0x35 | 0x45 | 검증 요청 |
| 0x36 | 0x46 | 점프 명령 |

---

## 0x30: Connection

```c
void Handle_Connection(void) {
    uint8_t response[8];
    response[0] = 0x40;
    response[1] = fw_check[0];  // 펌웨어 체크섬
    response[2] = fw_check[1];
    response[3] = fw_check[2];
    response[4] = fw_check[3];
    response[5] = fw_date[0];   // 날짜 (20)
    response[6] = fw_date[1];   // 날짜 (04)
    response[7] = fw_date[2];   // 날짜 (29)
    
    CAN_Send(0x5FE, response, 8);
}
```

응답에 펌웨어 체크섬과 날짜가 들어간다. `20 04 29` = 2020-04-29.

---

## 0x31: Key 계산

```c
void Handle_KeyCalc(uint8_t *data) {
    uint32_t key = *(uint32_t *)&data[1];  // 받은 키
    uint32_t expected = CalcKey(fw_check);  // 예상 키
    
    if (key == expected) {
        g_iap_state = STATE_CONNECTED;
        response[0] = 0x41;  // OK
    } else {
        response[0] = 0x4F;  // FAIL
    }
}
```

인증 단계다. PC가 보낸 키가 맞아야 연결 성공.

---

## 키 알고리즘

```c
uint32_t CalcKey(uint8_t *check) {
    uint32_t key = check[0] ^ check[1];
    key = (key << 8) | (check[2] ^ check[3]);
    key = key * 0x1234 + 0x5678;
    return key;
}
```

단순한 XOR + 곱셈 연산. 역분석으로 알아냈다.

---

## 프로토콜 흐름

```
PC                    BMS
 │                     │
 │──── 0x30 ──────────>│  Connection 요청
 │<─── 0x40 ───────────│  체크섬 + 날짜
 │                     │
 │──── 0x31 + Key ────>│  키 계산값
 │<─── 0x41 ───────────│  OK
 │                     │
 │──── 0x32 + Size ───>│  펌웨어 크기
 │<─── 0x42 ───────────│  OK
 │                     │
 │──── 0x33 + Data ───>│  데이터 (반복)
 │<─── 0x43 ───────────│  OK
 │                     │
 │──── 0x35 ──────────>│  검증 요청
 │<─── 0x45 ───────────│  CRC OK
 │                     │
 │──── 0x36 ──────────>│  점프 명령
```

---

다음 글에서 데이터 수신/Flash 쓰기 분석.

[#15 - 데이터 수신 분석](/posts/ghidra/ghidra-stm32-re-15/)
