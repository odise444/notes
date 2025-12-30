---
title: "STM32 부트로더 개발기 #13 - 프로토콜 설계"
date: 2024-12-17
tags: ["STM32", "Bootloader", "CAN", "프로토콜"]
categories: ["개발기"]
series: ["STM32 부트로더 개발기"]
summary: "PC와 부트로더가 CAN으로 어떻게 대화할지 정해야 한다."
---

이제 CAN으로 펌웨어를 주고받을 프로토콜을 설계하자.

---

## 기본 구조

```
PC (업로더)                BMS (부트로더)
    │                          │
    │──── 연결 요청 ──────────>│
    │<─── 인증 챌린지 ─────────│
    │──── 인증 응답 ──────────>│
    │<─── 인증 성공 ───────────│
    │                          │
    │<─── 크기 요청 ───────────│
    │──── 크기 응답 ──────────>│
    │                          │
    │<─── 데이터 요청 ─────────│
    │──── 페이지 데이터 ──────>│
    │      (반복)              │
    │                          │
    │<─── 검증 완료 ───────────│
```

---

## CAN ID 할당

```c
#define CAN_ID_PC_TO_BMS  0x5FF   // PC → BMS
#define CAN_ID_BMS_TO_PC  0x5FE   // BMS → PC
```

두 ID만 사용. 단순하게.

---

## 명령 코드

**PC → BMS (0x5FF):**

| data[0] | 명령 | 설명 |
|---------|------|------|
| 0x30 | CONNECT | 연결 요청 |
| 0x31 | AUTH | 인증 응답 |
| 0x32 | SIZE | 펌웨어 크기 응답 |
| 0x33 | PAGE_START | 페이지 시작 |
| 0x34 | PAGE_END | 페이지 끝 |
| 0x35 | RESET | 리셋 요청 |

**BMS → PC (0x5FE):**

| data[0] | 응답 | 설명 |
|---------|------|------|
| 0x40 | CHALLENGE | 인증 챌린지 |
| 0x41 | AUTH_OK | 인증 성공 |
| 0x42 | SIZE_REQ | 크기 요청 |
| 0x43 | PAGE_REQ | 페이지 요청 |
| 0x44 | PAGE_ACK | 페이지 수신 완료 |
| 0x45 | VERIFY_START | 검증 시작 |
| 0x46 | VERIFY_OK | 검증 완료 |
| 0x4F | ERROR | 에러 |

---

## 인증

아무나 펌웨어 올리면 안 됨.

간단한 Seed-Key 방식:

```
1. PC: CONNECT (0x30) 전송
2. BMS: CHALLENGE (0x40) + seed값 전송
3. PC: seed로 key 계산해서 AUTH (0x31) + key 전송
4. BMS: key 검증 후 AUTH_OK (0x41) 전송
```

Key 계산:
```c
// 예시: 단순 XOR
uint32_t calc_key(uint32_t seed) {
    return seed ^ 0x12345678;
}
```

실제론 더 복잡하게 해야 함.

---

## 페이지 전송

한 페이지 (2KB) 전송 흐름:

```
BMS: PAGE_REQ (0x43) + 페이지번호
PC:  PAGE_START (0x33)
PC:  data (8바이트) × 256회 = 2048바이트
PC:  PAGE_END (0x34) + CRC
BMS: PAGE_ACK (0x44) 또는 ERROR (0x4F)
```

---

## 데이터 패킷 구조

CAN 프레임은 최대 8바이트.

```
PAGE_START:
  data[0] = 0x33
  data[1-2] = 페이지 번호 (little endian)

DATA (raw):
  data[0-7] = 펌웨어 데이터

PAGE_END:
  data[0] = 0x34
  data[1-4] = CRC32 (little endian)
```

---

## 에러 코드

```c
#define ERR_NONE        0x00
#define ERR_AUTH        0x01  // 인증 실패
#define ERR_SIZE        0x02  // 크기 초과
#define ERR_CRC         0x03  // CRC 불일치
#define ERR_FLASH       0x04  // Flash 에러
#define ERR_TIMEOUT     0x05  // 타임아웃
```

---

## 타임아웃

각 단계별 타임아웃:

- 연결: 3초
- 인증: 5초
- 페이지 수신: 10초 (느린 PC 대비)

타임아웃 발생 시 부트로더는 처음으로 돌아감.

---

다음 글에서 상태 머신 구현.

[#14 - 상태 머신 구현](/posts/stm32-bootloader/stm32-bootloader-14/)
