---
title: "Ghidra STM32 역분석 #19 - 부트 진입 조건"
date: 2024-12-14
draft: false
tags: ["Ghidra", "STM32", "리버스엔지니어링", "역분석", "부트로더"]
categories: ["역분석"]
series: ["Ghidra 역분석"]
summary: "부트로더 모드로 들어가는 조건이 뭘까?"
---

부트로더가 매번 실행되면 안 된다. 특정 조건에서만 IAP 모드로 진입해야 한다.

Reset_Handler 이후 main() 초반부를 보자.

---

## 진입 조건 체크

```c
void main(void) {
    // 클럭, GPIO 초기화
    RCC_Init();
    GPIO_Init();
    
    // 부트 진입 조건 체크
    if (Check_BootCondition() == 0) {
        // 조건 불충족 → 앱으로 점프
        Jump_To_App();
    }
    
    // IAP 모드 진입
    CAN_Init();
    IAP_Main();
}
```

`Check_BootCondition()` 함수를 찾아보자.

---

## 조건 1: GPIO 체크

```c
uint8_t Check_BootCondition(void) {
    // PB2 버튼 체크 (Active Low)
    if ((*(uint32_t *)0x40010C08 & 0x04) == 0) {
        return 1;  // 버튼 눌림 → 부트 모드
    }
```

`0x40010C08`은 GPIOB_IDR. PB2가 눌려있으면 부트 모드.

---

## 조건 2: 매직 플래그

```c
    // RAM 매직 값 체크
    if (*(uint32_t *)0x20000000 == 0xDEADBEEF) {
        *(uint32_t *)0x20000000 = 0;  // 클리어
        return 1;  // 매직 값 → 부트 모드
    }
```

앱에서 부트로더로 진입하고 싶을 때 RAM에 `0xDEADBEEF` 쓰고 리셋하면 된다.

---

## 조건 3: 앱 유효성

```c
    // 앱 영역 Stack Pointer 검증
    uint32_t app_sp = *(uint32_t *)0x08042800;
    if ((app_sp < 0x20000000) || (app_sp > 0x20010000)) {
        return 1;  // 유효하지 않음 → 부트 모드
    }
    
    return 0;  // 모든 조건 불충족 → 앱 실행
}
```

앱 시작 주소의 첫 4바이트(Stack Pointer)가 RAM 범위인지 확인한다. Flash가 비어있으면 0xFFFFFFFF라서 부트 모드로 들어간다.

---

## 정리

| 조건 | 값 | 결과 |
|------|-----|------|
| PB2 버튼 | LOW | 부트 모드 |
| RAM 매직 | 0xDEADBEEF | 부트 모드 |
| 앱 SP | RAM 범위 밖 | 부트 모드 |
| 위 모두 아님 | - | 앱 실행 |

---

다음 글에서 앱 유효성 검사 상세 분석.

[#20 - 앱 유효성 검사](/posts/ghidra/ghidra-stm32-re-20/)
