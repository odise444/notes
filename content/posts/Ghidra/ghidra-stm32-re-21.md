---
title: "Ghidra STM32 역분석 #21 - 앱 점프 코드"
date: 2024-12-14
draft: false
tags: ["Ghidra", "STM32", "리버스엔지니어링", "역분석", "부트로더"]
categories: ["역분석"]
series: ["Ghidra 역분석"]
summary: "부트로더에서 앱으로 어떻게 점프할까?"
---

검증 끝났으면 앱으로 점프해야 한다. 단순히 함수 호출하면 안 된다.

---

## 점프 코드

```c
void Jump_To_App(void) {
    uint32_t app_addr = 0x08042800;
    
    // 인터럽트 비활성화
    __disable_irq();
    
    // Vector Table 재배치
    *(uint32_t *)0xE000ED08 = app_addr;  // SCB->VTOR
    
    // Stack Pointer 설정
    __set_MSP(*(uint32_t *)app_addr);
    
    // Reset Handler로 점프
    void (*app_reset)(void) = (void (*)(void))(*(uint32_t *)(app_addr + 4));
    app_reset();
}
```

---

## VTOR 설정

`0xE000ED08`은 SCB->VTOR (Vector Table Offset Register).

앱이 `0x08042800`에서 시작하니까 VTOR도 거기로 바꿔줘야 한다. 안 바꾸면 앱에서 인터럽트 발생할 때 부트로더 Vector Table 참조해서 엉뚱한 데로 간다.

---

## MSP 설정

```c
__set_MSP(*(uint32_t *)app_addr);
```

Main Stack Pointer를 앱의 Initial SP로 설정. 부트로더가 쓰던 스택 버리고 새로 시작.

---

## 함수 포인터로 점프

```c
void (*app_reset)(void) = (void (*)(void))(*(uint32_t *)(app_addr + 4));
app_reset();
```

`app_addr + 4`에 있는 Reset_Handler 주소를 함수 포인터로 캐스팅해서 호출.

Ghidra 디컴파일러는 이걸 이상하게 보여줄 수 있다:

```c
(*(code *)((uint *)(app_addr + 4)))();
```

---

## 어셈블리로 보면

```asm
    ldr   r0, =0x08042800
    ldr   r1, [r0]          ; SP 읽기
    msr   msp, r1           ; MSP 설정
    ldr   r0, [r0, #4]      ; Reset Handler
    bx    r0                ; 점프
```

깔끔하다.

---

다음 글에서 버퍼 → 앱 복사 로직.

[#22 - 버퍼 복사 로직](/posts/ghidra/ghidra-stm32-re-22/)
