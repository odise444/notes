---
title: "STM32 부트로더 개발기 #7 - 앱으로 점프하기"
date: 2024-12-17
tags: ["STM32", "Bootloader", "점프", "MSP"]
categories: ["개발기"]
series: ["STM32 부트로더 개발기"]
summary: "VTOR 설정하고, MSP 바꾸고, 함수 포인터로 점프."
---

이제 진짜 앱으로 점프해보자.

---

## 점프 순서

1. 인터럽트 비활성화
2. 주변장치 정리
3. VTOR 재배치
4. MSP (Main Stack Pointer) 설정
5. Reset Handler로 점프

---

## 전체 코드

```c
#define APP_ADDR  0x08042800

void jump_to_app(void) {
    uint32_t app_sp    = *(volatile uint32_t *)(APP_ADDR);
    uint32_t app_reset = *(volatile uint32_t *)(APP_ADDR + 4);
    
    // 1. 인터럽트 비활성화
    __disable_irq();
    
    // 2. SysTick 끄기
    SysTick->CTRL = 0;
    SysTick->LOAD = 0;
    SysTick->VAL  = 0;
    
    // 3. 모든 인터럽트 펜딩 클리어
    for (int i = 0; i < 8; i++) {
        NVIC->ICER[i] = 0xFFFFFFFF;  // 인터럽트 비활성화
        NVIC->ICPR[i] = 0xFFFFFFFF;  // 펜딩 클리어
    }
    
    // 4. VTOR 재배치
    SCB->VTOR = APP_ADDR;
    
    // 5. MSP 설정
    __set_MSP(app_sp);
    
    // 6. 앱으로 점프
    void (*reset_handler)(void) = (void (*)(void))app_reset;
    reset_handler();
    
    // 여기 도달하면 안 됨
    while (1);
}
```

---

## 왜 인터럽트 비활성화?

점프 직후 앱이 아직 초기화 안 됐는데 인터럽트 터지면?

HardFault.

---

## 왜 SysTick 끄나?

HAL 라이브러리가 SysTick 쓴다.

부트로더에서 HAL_Delay() 썼으면 SysTick 돌고 있음.

앱 진입 전에 꺼야 깔끔.

---

## MSP 설정

`__set_MSP()`는 CMSIS 함수.

```c
// 실제 구현 (인라인 어셈블리)
__attribute__((always_inline)) static inline void __set_MSP(uint32_t topOfMainStack) {
    __asm volatile ("MSR msp, %0" : : "r" (topOfMainStack));
}
```

앱의 스택 시작 주소로 MSP 변경.

---

## 함수 포인터로 점프

```c
void (*reset_handler)(void) = (void (*)(void))app_reset;
reset_handler();
```

app_reset 주소를 함수 포인터로 캐스팅해서 호출.

이게 점프.

---

## Thumb 비트 주의

Reset Handler 주소에 이미 Thumb 비트(LSB=1) 포함되어 있음.

```
0x08042801  ← LSB가 1
```

함수 포인터로 호출하면 자동으로 Thumb 모드로 분기.

`& ~1` 하면 안 됨.

---

## 점프 후

앱의 Reset_Handler 실행:
1. `.data` 섹션 복사 (Flash → RAM)
2. `.bss` 섹션 0으로 초기화
3. SystemInit() 호출
4. main() 호출

부트로더는 잊혀짐.

---

## 디버깅 팁

점프 안 되면:

```c
// 점프 전에
printf("SP: 0x%08X\n", app_sp);
printf("Reset: 0x%08X\n", app_reset);
printf("VTOR: 0x%08X\n", SCB->VTOR);
```

값 확인. SP가 이상하면 앱이 제대로 빌드 안 된 거.

---

다음 글에서 부트 진입 조건.

[#8 - 부트 진입 조건](/posts/stm32-bootloader/stm32-bootloader-08/)
