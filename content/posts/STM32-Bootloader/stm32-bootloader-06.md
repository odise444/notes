---
title: "STM32 부트로더 개발기 #6 - Vector Table 재배치"
date: 2024-12-17
tags: ["STM32", "Bootloader", "VTOR", "인터럽트"]
categories: ["개발기"]
series: ["STM32 부트로더 개발기"]
summary: "앱으로 점프하기 전에 VTOR 설정 안 하면 인터럽트가 엉뚱한 데로 간다."
---

부트로더에서 앱으로 점프했다.

근데 인터럽트 터지면 HardFault.

---

## 문제

STM32는 기본적으로 0x08000000의 Vector Table을 참조한다.

```
인터럽트 발생
    ↓
0x08000000 + offset 주소 읽음
    ↓
부트로더의 핸들러로 점프  ← 문제!
```

앱 실행 중인데 부트로더 핸들러가 호출됨.

---

## 해결: VTOR

Vector Table Offset Register.

Cortex-M3/M4에서 지원. (M0는 없음)

```c
SCB->VTOR = APP_ADDR;  // 앱의 Vector Table 주소로 변경
```

이제 인터럽트 발생하면:

```
인터럽트 발생
    ↓
0x08042800 + offset 주소 읽음  ← VTOR 값 사용
    ↓
앱의 핸들러로 점프  ← 정상!
```

---

## VTOR 위치

```c
// core_cm3.h 또는 core_cm4.h
#define SCB_VTOR  (*((volatile uint32_t *)0xE000ED08))
```

또는 CMSIS 헤더 사용:

```c
#include "stm32f1xx.h"
SCB->VTOR = APP_ADDR;
```

---

## VTOR 정렬 조건

VTOR 값은 정렬되어야 함.

```
Cortex-M3/M4: 128바이트 또는 벡터 수에 따라
              최소 0x80 (128) 배수
```

0x08042800은 0x80의 배수라 OK.

---

## 부트로더에서 설정

앱으로 점프하기 직전에:

```c
void jump_to_app(void) {
    uint32_t app_sp = *(volatile uint32_t *)APP_ADDR;
    uint32_t app_reset = *(volatile uint32_t *)(APP_ADDR + 4);
    
    // VTOR 재배치
    SCB->VTOR = APP_ADDR;
    
    // MSP 설정
    __set_MSP(app_sp);
    
    // 앱으로 점프
    void (*reset_handler)(void) = (void (*)(void))app_reset;
    reset_handler();
}
```

---

## 앱에서도 설정?

앱 쪽 `system_stm32f1xx.c`에서:

```c
#define VECT_TAB_OFFSET  0x00042800U  // 앱 오프셋
```

이러면 앱이 SystemInit()에서 VTOR 설정함.

근데 부트로더에서 이미 했으니까 중복. 둘 중 하나만 해도 됨.

개인적으론 부트로더에서 하는 게 깔끔함.

---

## Cortex-M0는?

VTOR 없음. Vector Table 재배치 불가.

대신 RAM에 Vector Table 복사하고 리맵핑하는 방법 있음. 복잡함.

F103은 Cortex-M3라 VTOR 사용 가능.

---

다음 글에서 앱으로 점프하기.

[#7 - 앱으로 점프하기](/posts/stm32-bootloader/stm32-bootloader-07/)
