---
title: "Ghidra STM32 역분석 #15 - 상태 머신 복원"
date: 2024-12-13
draft: false
tags: ["Ghidra", "STM32", "리버스엔지니어링", "역분석", "상태머신"]
categories: ["역분석"]
series: ["Ghidra 역분석"]
summary: "부트로더의 전체 흐름을 상태 머신으로 복원한다."
---

명령 처리 함수에서 공통으로 참조하는 주소가 있다:

```c
if (*(uint8_t *)0x20000300 != 3) {
    return;
}
```

`0x20000300`이 상태 변수다.

---

## 상태값

| 값 | 의미 |
|----|------|
| 0 | IDLE |
| 1 | WAIT_KEY |
| 2 | CONNECTED |
| 3 | WAIT_DATA |
| 4 | PROGRAMMING |
| 5 | VERIFY |
| 6 | COMPLETE |

---

## 상태 전이

```
IDLE ──0x30──> WAIT_KEY ──0x31──> CONNECTED
                                      │
                                     0x32
                                      ▼
                                 WAIT_DATA
                                      │
                                     0x33 (반복)
                                      ▼
                                PROGRAMMING
                                      │
                                     0x35
                                      ▼
                                   VERIFY
                                      │
                                     0x36
                                      ▼
                                  COMPLETE ──> Jump to App
```

---

## 타임아웃

```c
void CheckTimeout(void) {
    if (g_tick - g_last_rx > 5000) {  // 5초
        g_iap_state = IDLE;
        g_timeout_flag = 1;
    }
}
```

5초 동안 명령 없으면 IDLE로 복귀.

---

다음 글에서 데이터 수신/Flash 쓰기 분석.

[#16 - 데이터 프레임 처리](/posts/ghidra/ghidra-stm32-re-16/)
