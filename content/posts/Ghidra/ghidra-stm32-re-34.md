---
title: "Ghidra STM32 역분석 #34 - 부트로더 개선"
date: 2024-12-14
draft: false
tags: ["Ghidra", "STM32", "리버스엔지니어링", "역분석", "부트로더"]
categories: ["역분석"]
series: ["Ghidra 역분석"]
summary: "역분석한 부트로더를 개선해보자. 보안, 안정성, 기능 추가."
---

소스코드 복원했으니 이제 개선할 수 있다.

---

## 개선 1: Key 알고리즘 강화

원본 알고리즘은 너무 단순했다:

```c
// 원본: XOR + 곱셈
key = fw_chk ^ fw_date;
key = key * 0x1234 + 0x5678;
```

개선:

```c
// SHA256 기반
uint8_t hash[32];
SHA256_CTX ctx;
SHA256_Init(&ctx);
SHA256_Update(&ctx, fw_chk, 4);
SHA256_Update(&ctx, fw_date, 3);
SHA256_Update(&ctx, secret_key, 16);  // 비밀 키 추가
SHA256_Final(hash, &ctx);

key = *(uint32_t *)hash;
```

---

## 개선 2: 재전송 요청

원본은 데이터 에러 나면 그냥 실패:

```c
// 개선: 재전송 요청
if (crc_error) {
    Send_NACK(page_num);
    retry_count++;
    if (retry_count > 3) {
        return ERROR;
    }
    continue;
}
```

---

## 개선 3: 진행률 표시

```c
// 페이지 완료할 때마다 진행률 전송
void Send_Progress(uint8_t percent) {
    uint8_t resp[2] = {0x50, percent};
    CAN_Send(resp, 2);
}
```

업로더에서 표시:

```python
if resp[0] == 0x50:
    print(f"Progress: {resp[1]}%")
```

---

## 개선 4: 버전 정보

```c
#define BL_VERSION_MAJOR    2
#define BL_VERSION_MINOR    0

void Handle_GetVersion(void) {
    uint8_t resp[3] = {0x48, BL_VERSION_MAJOR, BL_VERSION_MINOR};
    CAN_Send(resp, 3);
}
```

---

## 개선 5: 강제 부트 명령

앱에서 부트로더로 진입하는 명령 추가:

```c
// 앱에서 호출
void Enter_Bootloader(void) {
    *(uint32_t *)0x20000000 = 0xDEADBEEF;
    NVIC_SystemReset();
}
```

CAN 명령으로도 가능하게:

```c
case CMD_FORCE_BOOT:  // 0x3F
    *(uint32_t *)0x20000000 = 0xDEADBEEF;
    NVIC_SystemReset();
    break;
```

---

## 개선 6: Watchdog

무한 대기 방지:

```c
void main(void) {
    IWDG_Init(10000);  // 10초 타임아웃
    
    while (1) {
        IAP_Process();
        IWDG_Refresh();
        
        if (g_timeout > 30000) {  // 30초 무통신
            NVIC_SystemReset();
        }
    }
}
```

---

## 결과

| 항목 | 원본 | 개선 |
|------|------|------|
| 인증 | XOR | SHA256 |
| 에러 처리 | 실패 | 재전송 |
| 진행률 | 없음 | 표시 |
| 버전 조회 | 없음 | 있음 |
| 강제 부트 | 버튼만 | 명령 추가 |
| Watchdog | 없음 | 10초 |

---

## 시리즈 완료

34개 글에 걸쳐 STM32 부트로더를 역분석하고 복원/개선했다.

소스코드 없이 HEX 파일만으로 시작해서:
- 프로토콜 분석
- 소스코드 복원
- Python 업로더 제작
- 부트로더 개선

까지 완료.

---

다음은 번외편.

[번외 #1 - Ghidra 스크립트](/posts/ghidra/ghidra-stm32-re-b1/)
