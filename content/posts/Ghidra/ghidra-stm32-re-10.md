---
title: "Ghidra로 STM32 부트로더 역분석한 썰 #10 - GPIO 초기화 분석"
date: 2024-12-12
draft: false
tags: ["Ghidra", "STM32", "리버스엔지니어링", "역분석", "GPIO"]
categories: ["역분석"]
summary: "LED는 어디에? 버튼은? GPIO 설정을 역분석해서 핀맵을 복원하자."
---

## 지난 글 요약

[지난 글](/posts/ghidra/ghidra-stm32-re-09/)에서 RCC 설정을 복원했다. HSE 8MHz × PLL 9 = 72MHz. 이제 GPIO 초기화를 분석해서 **핀맵**을 복원하자.

## GPIO 레지스터 주소

STM32F103의 GPIO:

| 포트 | 베이스 주소 | CRL | CRH |
|------|-------------|-----|-----|
| GPIOA | 0x40010800 | 0x40010800 | 0x40010804 |
| GPIOB | 0x40010C00 | 0x40010C00 | 0x40010C04 |
| GPIOC | 0x40011000 | 0x40011000 | 0x40011004 |
| GPIOD | 0x40011400 | 0x40011400 | 0x40011404 |
| GPIOE | 0x40011800 | 0x40011800 | 0x40011804 |

**CRL**: Pin 0~7 설정  
**CRH**: Pin 8~15 설정

## GPIO Configuration 비트

각 핀은 **4비트**로 설정:

```
CNF[1:0] + MODE[1:0] = 4비트/핀

MODE[1:0]:
  00 = Input (리셋 상태)
  01 = Output 10MHz
  10 = Output 2MHz
  11 = Output 50MHz

CNF[1:0] (Output 모드):
  00 = Push-Pull
  01 = Open-Drain
  10 = AF Push-Pull
  11 = AF Open-Drain

CNF[1:0] (Input 모드):
  00 = Analog
  01 = Floating
  10 = Pull-up/Pull-down
  11 = Reserved
```

## Ghidra에서 GPIO 접근 찾기

**Search → For Scalars → Value: 0x40010800**

결과:
```
08002600: str r1, [0x40010800]   ; GPIOA_CRL
08002610: str r1, [0x40010804]   ; GPIOA_CRH
08002650: str r1, [0x40010C00]   ; GPIOB_CRL
...
```

## GPIO 초기화 함수 분석

발견된 함수 `FUN_08002600`:

```c
void FUN_08002600(void) {
    // GPIOA 설정
    *(uint32_t *)0x40010800 = 0x44444444;  // CRL: 모두 Input Floating
    *(uint32_t *)0x40010804 = 0x444B4444;  // CRH: PA11, PA12 특별 설정
    
    // GPIOB 설정
    *(uint32_t *)0x40010C00 = 0x44444433;  // CRL: PB0, PB1 Output
    *(uint32_t *)0x40010C04 = 0x44444444;  // CRH: 모두 Input
    
    // GPIOC 설정
    *(uint32_t *)0x40011004 = 0x44344444;  // CRH: PC13 Output
    
    // 초기 출력값
    *(uint32_t *)0x40010C10 = 0x0003;      // GPIOB_ODR: PB0, PB1 High
}
```

## 핀 설정 분석

### GPIOA_CRH = 0x444B4444

```
0x444B4444 분해 (Pin 8~15, 4비트씩):

Pin 15: 0x4 = 0100 = Input Floating
Pin 14: 0x4 = 0100 = Input Floating
Pin 13: 0x4 = 0100 = Input Floating
Pin 12: 0xB = 1011 = AF Open-Drain, 50MHz  ← CAN_TX!
Pin 11: 0x4 = 0100 = Input Floating        ← CAN_RX!
Pin 10: 0x4 = 0100 = Input Floating
Pin 9:  0x4 = 0100 = Input Floating
Pin 8:  0x4 = 0100 = Input Floating
```

**발견**: PA11 = CAN_RX, PA12 = CAN_TX (AF 모드)

### GPIOB_CRL = 0x44444433

```
Pin 7~2: 0x4 = Input Floating
Pin 1:   0x3 = 0011 = Output Push-Pull 50MHz  ← LED?
Pin 0:   0x3 = 0011 = Output Push-Pull 50MHz  ← LED?
```

**발견**: PB0, PB1 = Output (LED 추정)

### GPIOC_CRH = 0x44344444

```
Pin 15~14: Input
Pin 13:    0x3 = Output Push-Pull 50MHz  ← LED?
Pin 12~8:  Input
```

**발견**: PC13 = Output (보드 내장 LED일 가능성)

## LED 동작 확인

출력 레지스터 조작 찾기:

```c
// GPIOB_ODR (0x40010C0C)
*(uint32_t *)0x40010C0C |= 0x01;   // PB0 = High (LED ON)
*(uint32_t *)0x40010C0C &= ~0x01;  // PB0 = Low (LED OFF)

// GPIOB_BSRR (0x40010C10) - Atomic Set/Reset
*(uint32_t *)0x40010C10 = 0x0001;      // PB0 Set (ON)
*(uint32_t *)0x40010C10 = 0x00010000;  // PB0 Reset (OFF)
```

**패턴 발견**:
- BSRR 하위 16비트 = Set (켜기)
- BSRR 상위 16비트 = Reset (끄기)

## LED 토글 함수 찾기

```c
void FUN_08002700(void) {
    // ODR XOR 연산 = 토글
    *(uint32_t *)0x40010C0C ^= 0x02;  // PB1 토글
}
```

**PB1 = 상태 LED** 확인!

## 버튼 입력 찾기

입력 읽기 패턴:

```c
void FUN_08002750(void) {
    uint32_t idr = *(uint32_t *)0x40010C08;  // GPIOB_IDR
    
    if ((idr & 0x04) == 0) {  // PB2가 Low면
        // 버튼 눌림 처리
    }
}
```

**PB2 = 버튼** (Active Low)

## CAN 핀 리맵 확인

STM32F103의 CAN 핀:

| 옵션 | CAN_RX | CAN_TX |
|------|--------|--------|
| 기본 | PA11 | PA12 |
| Remap | PB8 | PB9 |

**AFIO_MAPR** 레지스터 확인:

```c
// 0x40010004 = AFIO_MAPR
uint32_t mapr = *(uint32_t *)0x40010004;

// CAN Remap 비트 [14:13]
// 00 = PA11/PA12 (기본)
// 10 = PB8/PB9 (Remap)
```

코드에서 Remap 설정 없음 → **기본 핀(PA11/PA12) 사용**

## 복원된 핀맵

| 핀 | 기능 | 모드 | 용도 |
|----|------|------|------|
| PA11 | CAN_RX | AF Input | CAN 수신 |
| PA12 | CAN_TX | AF Open-Drain | CAN 송신 |
| PB0 | LED1 | Output PP | 전원 LED? |
| PB1 | LED2 | Output PP | 상태 LED |
| PB2 | Button | Input Pull-up | 부트 모드 선택? |
| PC13 | LED3 | Output PP | 오류 LED? |

## GPIO 초기화 코드 복원

```c
void GPIO_Init(void) {
    // GPIO 클럭 활성화 (RCC_APB2ENR)
    RCC->APB2ENR |= RCC_APB2ENR_IOPAEN | 
                    RCC_APB2ENR_IOPBEN |
                    RCC_APB2ENR_IOPCEN;
    
    // GPIOA: CAN 핀 설정
    GPIOA->CRL = 0x44444444;  // PA0~7: Input Floating
    GPIOA->CRH = 0x444B4444;  // PA11: Input (CAN_RX)
                              // PA12: AF Open-Drain 50MHz (CAN_TX)
    
    // GPIOB: LED, 버튼
    GPIOB->CRL = 0x44444433;  // PB0, PB1: Output 50MHz
                              // PB2: Input (버튼)
    GPIOB->CRH = 0x44444444;
    GPIOB->ODR = 0x0003;      // LED 초기 ON
    
    // GPIOC: LED
    GPIOC->CRH = 0x44344444;  // PC13: Output 50MHz
}
```

## LED 제어 함수 복원

```c
// LED 정의
#define LED1_PORT   GPIOB
#define LED1_PIN    (1 << 0)
#define LED2_PORT   GPIOB
#define LED2_PIN    (1 << 1)
#define LED3_PORT   GPIOC
#define LED3_PIN    (1 << 13)

void LED_On(GPIO_TypeDef *port, uint16_t pin) {
    port->BSRR = pin;  // Set
}

void LED_Off(GPIO_TypeDef *port, uint16_t pin) {
    port->BSRR = pin << 16;  // Reset
}

void LED_Toggle(GPIO_TypeDef *port, uint16_t pin) {
    port->ODR ^= pin;
}

// 상태 표시
void LED_SetStatus(uint8_t status) {
    switch (status) {
    case 0:  // IDLE
        LED_On(LED1_PORT, LED1_PIN);
        LED_Off(LED2_PORT, LED2_PIN);
        break;
    case 1:  // CONNECTED
        LED_On(LED1_PORT, LED1_PIN);
        LED_On(LED2_PORT, LED2_PIN);
        break;
    case 2:  // ERROR
        LED_Off(LED1_PORT, LED1_PIN);
        LED_Toggle(LED3_PORT, LED3_PIN);
        break;
    }
}
```

## 부트 모드 버튼 감지

```c
#define BOOT_BUTTON_PORT  GPIOB
#define BOOT_BUTTON_PIN   (1 << 2)

bool IsBootButtonPressed(void) {
    // Active Low: 누르면 0
    return (BOOT_BUTTON_PORT->IDR & BOOT_BUTTON_PIN) == 0;
}

void CheckBootMode(void) {
    if (IsBootButtonPressed()) {
        // 부트로더 모드 진입
        g_boot_mode = BOOT_MODE_IAP;
    } else {
        // 앱으로 점프
        g_boot_mode = BOOT_MODE_APP;
    }
}
```

## 삽질: CAN 핀 모드

CAN_TX를 Push-Pull로 설정하면 동작 안 함!

```c
// 잘못된 설정
GPIOA->CRH = 0x44434444;  // PA12: AF Push-Pull ❌

// 올바른 설정
GPIOA->CRH = 0x444B4444;  // PA12: AF Open-Drain ✅
```

CAN 버스는 **Open-Drain** 필수! (Wired-AND 구조)

## 삽질: LED 극성

LED가 Active High인지 Low인지 확인:

```c
// Active High (일반적)
// HIGH = LED ON, LOW = LED OFF
GPIOB->ODR |= LED_PIN;   // ON

// Active Low (보드 내장 LED)
// LOW = LED ON, HIGH = LED OFF
GPIOB->ODR &= ~LED_PIN;  // ON
```

PC13(보드 내장 LED)은 보통 **Active Low**!

## Ghidra 라벨 정리

```
주소 → 라벨
─────────────────────────────
0x40010800 → GPIOA_CRL
0x40010804 → GPIOA_CRH
0x40010808 → GPIOA_IDR
0x4001080C → GPIOA_ODR
0x40010810 → GPIOA_BSRR
0x40010C00 → GPIOB_CRL
...
FUN_08002600 → GPIO_Init
FUN_08002700 → LED_Toggle
```

## 정리

| 발견 내용 | 핀 | 기능 |
|-----------|-----|------|
| CAN 수신 | PA11 | CAN_RX (AF) |
| CAN 송신 | PA12 | CAN_TX (AF O.D.) |
| LED 1 | PB0 | 전원/상태 표시 |
| LED 2 | PB1 | 통신 상태 |
| 버튼 | PB2 | 부트 모드 선택 |
| LED 3 | PC13 | 에러 표시 |

**다음 글에서**: CAN 초기화 역분석 - 500kbps 설정, 필터 구성.

---

## 시리즈 목차

**Part 3: 주변장치 역분석편**
- [#9 - RCC 설정 복원하기](/posts/ghidra/ghidra-stm32-re-09/)
- **#10 - GPIO 초기화 분석** ← 현재 글
- #11 - CAN 초기화 역분석
- #12 - Flash 접근 함수 찾기

---

## 참고 자료

- [STM32F103 Reference Manual - GPIO](https://www.st.com/resource/en/reference_manual/rm0008.pdf)
- [STM32F103 Datasheet - Pinout](https://www.st.com/resource/en/datasheet/stm32f103ve.pdf)
