---
title: "Ghidra로 STM32 부트로더 역분석한 썰 #9 - RCC 설정 복원하기"
date: 2024-12-12
draft: false
tags: ["Ghidra", "STM32", "리버스엔지니어링", "역분석", "RCC", "클럭"]
categories: ["역분석"]
summary: "MCU의 심장, 클럭 설정을 역분석해보자. PLL 설정값을 찾아라!"
---

## 지난 글 요약

[지난 글](/posts/ghidra/ghidra-stm32-re-08/)에서 전역 변수 영역(.data, .bss)을 분석했다. 이제 주변장치 초기화 코드를 역분석할 차례. **RCC(Reset and Clock Control)**부터 시작하자.

## 왜 RCC부터?

모든 주변장치는 **클럭**이 있어야 동작한다:

```
전원 인가 → HSI(8MHz) 동작 → RCC 설정 → PLL(72MHz) → 주변장치 활성화
```

RCC 설정을 모르면:
- 타이머 주기 계산 불가
- UART 보레이트 계산 불가
- CAN 비트 타이밍 계산 불가

## STM32F103 클럭 트리

```
┌─────────────────────────────────────────────────────────┐
│                    Clock Tree                           │
├─────────────────────────────────────────────────────────┤
│                                                         │
│  HSI (8MHz) ──┬──→ SYSCLK ──→ AHB ──→ APB1 (max 36MHz) │
│               │       ↑           └──→ APB2 (max 72MHz) │
│  HSE (8MHz) ──┼───────┤                                 │
│               │       │                                 │
│               └──→ PLL ──→ PLLCLK (max 72MHz)          │
│                    (×9)                                 │
│                                                         │
└─────────────────────────────────────────────────────────┘
```

**일반적인 설정**: HSE(8MHz) × PLL(9) = 72MHz

## RCC 레지스터 주소

| 레지스터 | 주소 | 용도 |
|----------|------|------|
| RCC_CR | 0x40021000 | 클럭 제어 (HSE/PLL ON) |
| RCC_CFGR | 0x40021004 | 클럭 설정 (PLL 배수, 분주) |
| RCC_APB2ENR | 0x40021018 | APB2 주변장치 클럭 활성화 |
| RCC_APB1ENR | 0x4002101C | APB1 주변장치 클럭 활성화 |

## Ghidra에서 RCC 접근 찾기

### 방법 1: 주소 검색

**Search → For Scalars → Value: 0x40021000**

결과:
```
08002850: ldr r0, [0x40021000]   ; RCC_CR
08002900: ldr r0, [0x40021004]   ; RCC_CFGR
08002950: str r1, [0x40021018]   ; RCC_APB2ENR
```

### 방법 2: Reset_Handler 추적

Reset_Handler에서 main() 호출 전에 클럭 설정:

```c
void Reset_Handler(void) {
    // 1. .data 복사
    // 2. .bss 초기화
    // 3. SystemInit() ← 클럭 설정!
    // 4. main()
}
```

## SystemInit() 함수 찾기

Reset_Handler 디컴파일:

```c
void Reset_Handler(void) {
    // ... 초기화 코드 ...
    
    FUN_08002500();  // ← SystemInit 후보
    FUN_08002800();  // ← main 후보
}
```

`FUN_08002500` 디컴파일:

```c
void FUN_08002500(void) {
    // RCC_CR 조작
    *(uint32_t *)0x40021000 = *(uint32_t *)0x40021000 | 0x10000;
    
    // 대기 루프
    while ((*(uint32_t *)0x40021000 & 0x20000) == 0);
    
    // RCC_CFGR 설정
    *(uint32_t *)0x40021004 = 0x001D0402;
    
    // PLL ON
    *(uint32_t *)0x40021000 = *(uint32_t *)0x40021000 | 0x01000000;
    
    // PLL Ready 대기
    while ((*(uint32_t *)0x40021000 & 0x02000000) == 0);
    
    // SYSCLK = PLL
    *(uint32_t *)0x40021004 = (*(uint32_t *)0x40021004 & 0xFFFFFFFC) | 0x02;
}
```

**이게 SystemInit()이다!**

## RCC_CR 분석

```c
// 코드: 0x40021000 |= 0x10000
// 의미: RCC_CR의 bit 16 set

RCC_CR 비트맵:
┌────────────────────────────────────────────┐
│ Bit 24: PLLON   - PLL 활성화               │
│ Bit 25: PLLRDY  - PLL Ready (읽기 전용)    │
│ Bit 16: HSEON   - HSE 활성화               │
│ Bit 17: HSERDY  - HSE Ready (읽기 전용)    │
│ Bit 0:  HSION   - HSI 활성화               │
│ Bit 1:  HSIRDY  - HSI Ready                │
└────────────────────────────────────────────┘
```

**해석**:
```c
RCC->CR |= 0x10000;           // HSEON = 1 (HSE 켜기)
while (!(RCC->CR & 0x20000)); // HSERDY 대기
RCC->CR |= 0x01000000;        // PLLON = 1 (PLL 켜기)
while (!(RCC->CR & 0x02000000)); // PLLRDY 대기
```

## RCC_CFGR 분석

```c
// 코드: 0x40021004 = 0x001D0402
```

CFGR 비트 분해:

```
0x001D0402 = 0000 0000 0001 1101 0000 0100 0000 0010

Bit [1:0]   = 10      → SW: PLL 선택 (SYSCLK = PLLCLK)
Bit [3:2]   = 00      → SWS: (상태, 읽기 전용)
Bit [7:4]   = 0000    → HPRE: AHB 분주 = 1 (HCLK = SYSCLK)
Bit [10:8]  = 100     → PPRE1: APB1 분주 = 2 (PCLK1 = HCLK/2 = 36MHz)
Bit [13:11] = 000     → PPRE2: APB2 분주 = 1 (PCLK2 = HCLK = 72MHz)
Bit [15:14] = 00      → ADCPRE: ADC 분주 = 2
Bit 16      = 1       → PLLSRC: HSE 선택
Bit 17      = 0       → PLLXTPRE: HSE 분주 안 함
Bit [21:18] = 0111    → PLLMUL: ×9
```

**결론**:
```
HSE(8MHz) × 9 = 72MHz (SYSCLK)
AHB = 72MHz
APB1 = 36MHz (max)
APB2 = 72MHz
```

## 복원된 SystemInit()

```c
void SystemInit(void) {
    // 1. HSE 활성화
    RCC->CR |= RCC_CR_HSEON;
    while (!(RCC->CR & RCC_CR_HSERDY));
    
    // 2. Flash Latency 설정 (72MHz → 2 Wait State)
    FLASH->ACR |= FLASH_ACR_LATENCY_2;
    
    // 3. PLL 설정
    // PLLSRC = HSE, PLLMUL = 9, APB1 = /2
    RCC->CFGR = RCC_CFGR_PLLSRC_HSE | 
                RCC_CFGR_PLLMULL9 |
                RCC_CFGR_PPRE1_DIV2;
    
    // 4. PLL 활성화
    RCC->CR |= RCC_CR_PLLON;
    while (!(RCC->CR & RCC_CR_PLLRDY));
    
    // 5. SYSCLK = PLL
    RCC->CFGR |= RCC_CFGR_SW_PLL;
    while ((RCC->CFGR & RCC_CFGR_SWS) != RCC_CFGR_SWS_PLL);
}
```

## 주변장치 클럭 활성화 찾기

### APB2ENR (0x40021018)

```c
// 발견된 코드
*(uint32_t *)0x40021018 |= 0x0000007C;
```

비트 분해:
```
0x7C = 0111 1100

Bit 2: IOPAEN - GPIOA 클럭 활성화
Bit 3: IOPBEN - GPIOB 클럭 활성화
Bit 4: IOPCEN - GPIOC 클럭 활성화
Bit 5: IOPDEN - GPIOD 클럭 활성화
Bit 6: IOPEEN - GPIOE 클럭 활성화
```

**사용 GPIO**: A, B, C, D, E 전부!

### APB1ENR (0x4002101C)

```c
// 발견된 코드
*(uint32_t *)0x4002101C |= 0x02000000;
```

비트 분해:
```
0x02000000 = Bit 25

Bit 25: CAN1EN - CAN1 클럭 활성화
```

**CAN1 사용 확인!**

## Ghidra 라벨 정리

```
주소 → 라벨
─────────────────────────────
0x40021000 → RCC_CR
0x40021004 → RCC_CFGR
0x40021018 → RCC_APB2ENR
0x4002101C → RCC_APB1ENR
FUN_08002500 → SystemInit
```

## 구조체 정의 적용

**Data Type Manager → New → Structure**:

```c
typedef struct {
    volatile uint32_t CR;       // 0x00
    volatile uint32_t CFGR;     // 0x04
    volatile uint32_t CIR;      // 0x08
    volatile uint32_t APB2RSTR; // 0x0C
    volatile uint32_t APB1RSTR; // 0x10
    volatile uint32_t AHBENR;   // 0x14
    volatile uint32_t APB2ENR;  // 0x18
    volatile uint32_t APB1ENR;  // 0x1C
    volatile uint32_t BDCR;     // 0x20
    volatile uint32_t CSR;      // 0x24
} RCC_TypeDef;
```

**0x40021000에 적용** → 디컴파일 가독성 향상!

```c
// Before
*(uint32_t *)0x40021000 |= 0x10000;

// After
RCC->CR |= 0x10000;
```

## 삽질: Flash Latency

72MHz에서 Flash Latency 설정 안 하면 **Hard Fault**!

```c
// 빠뜨리기 쉬운 코드
*(uint32_t *)0x40022000 |= 0x12;  // FLASH_ACR

// 해석
FLASH->ACR = FLASH_ACR_LATENCY_2 | FLASH_ACR_PRFTBE;
// 2 Wait State + Prefetch Buffer Enable
```

**0x40022000 = FLASH_ACR**도 체크하자!

## 삽질: HSE Bypass

외부 오실레이터 vs 크리스탈:

```c
// 크리스탈 (일반적)
RCC->CR |= RCC_CR_HSEON;

// 외부 클럭 입력 (Bypass)
RCC->CR |= RCC_CR_HSEON | RCC_CR_HSEBYP;
```

Bypass 비트(0x40000) 확인 필요!

## 정리

| 발견 내용 | 값 |
|-----------|-----|
| 클럭 소스 | HSE (8MHz 크리스탈) |
| PLL 배수 | ×9 |
| SYSCLK | 72MHz |
| AHB (HCLK) | 72MHz |
| APB1 (PCLK1) | 36MHz |
| APB2 (PCLK2) | 72MHz |
| Flash Latency | 2 Wait State |
| 활성화 GPIO | A, B, C, D, E |
| 활성화 주변장치 | CAN1 |

**다음 글에서**: GPIO 초기화 분석 - LED, 버튼 핀 찾기.

---

## 시리즈 목차

**Part 1~2**: [이전 글 목록](/tags/ghidra/)

**Part 3: 주변장치 역분석편**
- **#9 - RCC 설정 복원하기** ← 현재 글
- #10 - GPIO 초기화 분석
- #11 - CAN 초기화 역분석
- #12 - Flash 접근 함수 찾기

---

## 참고 자료

- [STM32F103 Reference Manual - RCC](https://www.st.com/resource/en/reference_manual/rm0008.pdf)
- [STM32 Clock Configuration](https://www.st.com/resource/en/application_note/an4850.pdf)
