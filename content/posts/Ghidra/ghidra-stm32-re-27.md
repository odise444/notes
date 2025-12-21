---
title: "Ghidra STM32 역분석 #27 - main() 함수 복원"
date: 2024-12-14
draft: false
tags: ["Ghidra", "STM32", "리버스엔지니어링", "역분석", "소스코드"]
categories: ["역분석"]
series: ["Ghidra 역분석"]
summary: "분석 결과를 바탕으로 main() 함수를 복원해보자."
---

지금까지 분석한 걸 바탕으로 소스코드를 복원한다. 100% 똑같진 않겠지만 동작은 같게.

---

## 복원된 main()

```c
#include "stm32f1xx.h"
#include "can_iap.h"
#include "flash.h"

int main(void) {
    // 시스템 클럭 설정 (HSE 8MHz * PLL9 = 72MHz)
    SystemClock_Config();
    
    // GPIO 초기화
    GPIO_Init();
    
    // 부트 조건 체크
    if (!Check_BootCondition()) {
        if (Validate_App()) {
            Jump_To_App();
        }
    }
    
    // IAP 모드 진입
    LED_On(LED_BOOT);
    CAN_Init();
    
    // 메인 루프
    while (1) {
        IAP_Process();
        
        if (g_iap_complete) {
            LED_Off(LED_BOOT);
            NVIC_SystemReset();
        }
    }
}
```

---

## SystemClock_Config

```c
void SystemClock_Config(void) {
    // HSE 켜기
    RCC->CR |= RCC_CR_HSEON;
    while (!(RCC->CR & RCC_CR_HSERDY));
    
    // Flash latency (72MHz면 2 wait state)
    FLASH->ACR |= FLASH_ACR_LATENCY_2;
    
    // PLL 설정 (HSE / 1 * 9 = 72MHz)
    RCC->CFGR |= RCC_CFGR_PLLSRC;      // HSE as PLL source
    RCC->CFGR |= RCC_CFGR_PLLMULL9;    // x9
    
    // PLL 켜기
    RCC->CR |= RCC_CR_PLLON;
    while (!(RCC->CR & RCC_CR_PLLRDY));
    
    // 시스템 클럭을 PLL로
    RCC->CFGR |= RCC_CFGR_SW_PLL;
    while ((RCC->CFGR & RCC_CFGR_SWS) != RCC_CFGR_SWS_PLL);
}
```

---

## GPIO_Init

```c
void GPIO_Init(void) {
    // 클럭 활성화
    RCC->APB2ENR |= RCC_APB2ENR_IOPAEN | RCC_APB2ENR_IOPBEN;
    
    // PB0, PB1: LED 출력
    GPIOB->CRL &= ~0x000000FF;
    GPIOB->CRL |= 0x00000022;  // Push-pull, 2MHz
    
    // PB2: 버튼 입력 (풀업)
    GPIOB->CRL &= ~0x00000F00;
    GPIOB->CRL |= 0x00000800;  // Input pull-up
    GPIOB->ODR |= (1 << 2);    // Pull-up enable
}
```

---

## 복원 팁

- 변수명은 추정. 동작만 같으면 됨
- HAL vs 레지스터 직접 접근 중 선택
- 원본이 HAL이었으면 HAL로 복원하는 게 편함

---

다음 글에서 CAN IAP 모듈 복원.

[#28 - CAN IAP 모듈 복원](/posts/ghidra/ghidra-stm32-re-28/)
