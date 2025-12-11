---
title: "Ghidra로 STM32 부트로더 역분석한 썰 #6 - 메모리 맵 추정하기"
date: 2024-12-10
draft: false
tags: ["Ghidra", "STM32", "리버스엔지니어링", "역분석", "ARM", "메모리맵"]
categories: ["역분석"]
summary: "주소만 보고 Flash인지 RAM인지 주변장치인지 구분하는 법. STM32 메모리 맵 완전 정복."
---

## 지난 글 요약

[지난 글](/posts/ghidra/ghidra-stm32-re-05/)에서 Vector Table을 분석했다. Reset_Handler와 CAN 인터럽트 핸들러를 찾았다. 이제 코드에서 접근하는 주소들이 뭘 의미하는지 알아보자.

## 왜 메모리 맵이 중요한가

디컴파일 결과를 보면 이런 코드가 나온다:

```c
*(uint32_t *)0x40021018 = 0x00000004;
*(uint32_t *)0x40010c00 = 0x44444444;
*(uint32_t *)0x20000100 = some_value;
```

**이게 뭔지 모르면 분석 불가능.**

- `0x40021018` → RCC 레지스터 (클럭 설정)
- `0x40010c00` → GPIOA 레지스터 (핀 설정)
- `0x20000100` → RAM 변수

## STM32F103 메모리 맵

ARM Cortex-M3는 4GB 주소 공간을 영역별로 나눈다:

```
┌─────────────────────┐ 0xFFFFFFFF
│   Vendor Specific   │
├─────────────────────┤ 0xE0100000
│   Private Periph    │ (NVIC, SysTick, Debug)
├─────────────────────┤ 0xE0000000
│     External RAM    │
├─────────────────────┤ 0xA0000000
│   External Device   │
├─────────────────────┤ 0x60000000
│   Peripheral        │ ← 주변장치!
├─────────────────────┤ 0x40000000
│       SRAM          │ ← RAM!
├─────────────────────┤ 0x20000000
│       Code          │ ← Flash!
└─────────────────────┘ 0x00000000
```

## 영역별 상세

### 1. Flash (0x08000000)

```
0x08000000 ~ 0x0807FFFF: Main Flash (512KB)
```

**특징**:
- 코드 실행 영역
- 읽기 전용 (런타임에서)
- 쓰려면 Flash 프로그래밍 절차 필요

디컴파일에서 보이는 패턴:
```c
// 상수 데이터 접근
const_val = *(uint32_t *)0x08001234;

// 함수 호출
((void(*)(void))0x08002800)();
```

### 2. SRAM (0x20000000)

```
0x20000000 ~ 0x2000FFFF: SRAM (64KB for STM32F103VE)
```

**특징**:
- 변수 저장
- 스택
- 힙

디컴파일 패턴:
```c
// 전역 변수
global_var = *(uint32_t *)0x20000100;
*(uint32_t *)0x20000100 = new_value;

// 스택 (SP 근처)
local = *(uint32_t *)0x20005200;
```

### 3. Peripherals (0x40000000)

**가장 중요한 영역!** 하드웨어 제어.

```
0x40000000 ~ 0x4002FFFF: APB1/APB2/AHB Peripherals
```

STM32F103 주요 주변장치:

| 주소 범위 | 주변장치 | 용도 |
|-----------|----------|------|
| 0x40000000 | TIM2 | 타이머 2 |
| 0x40004400 | USART2 | 시리얼 2 |
| 0x40005400 | I2C1 | I2C 1 |
| 0x40006400 | **CAN1** | CAN 통신 |
| 0x40010800 | **GPIOA** | GPIO A |
| 0x40010C00 | **GPIOB** | GPIO B |
| 0x40011000 | GPIOC | GPIO C |
| 0x40013800 | USART1 | 시리얼 1 |
| 0x40021000 | **RCC** | 클럭 제어 |
| 0x40022000 | **FLASH** | 플래시 제어 |

## 실전: 주소로 기능 추정

### 예시 1: RCC 설정

```c
*(uint32_t *)0x40021018 = *(uint32_t *)0x40021018 | 0x4;
```

**분석**:
- `0x40021000` = RCC 베이스
- `0x40021018` = RCC + 0x18 = **RCC_APB2ENR**
- `| 0x4` = 비트 2 세트 = **IOPAEN** (GPIOA 클럭 활성화)

**해석**: GPIOA 클럭을 켠다!

### 예시 2: GPIO 설정

```c
*(uint32_t *)0x40010800 = 0x44444444;
```

**분석**:
- `0x40010800` = GPIOA 베이스 = **GPIOA_CRL**
- `0x44444444` = 모든 핀 Input (floating)

### 예시 3: CAN 설정

```c
*(uint32_t *)0x40006400 = 0x00010002;
*(uint32_t *)0x4000641C = 0x001C0006;
```

**분석**:
- `0x40006400` = CAN1_MCR (Master Control)
- `0x4000641C` = CAN1_BTR (Bit Timing)
- `0x001C0006` = 500kbps 설정값

## Ghidra에 메모리 맵 추가

### Memory Map 창 열기

**Window → Memory Map**

기본 상태:
```
Name    Start       End         R  W  X
ram     0x08000000  0x08003fff  ✓  ✓  ✓
```

### 영역 추가

**+ 아이콘** 클릭:

```
┌─────────────────────────────────────┐
│        Add Memory Block             │
├─────────────────────────────────────┤
│  Block Name: [SRAM               ]  │
│  Start Addr: [0x20000000         ]  │
│  Length:     [0x10000            ]  │ (64KB)
│  [✓] Read  [✓] Write  [ ] Execute  │
│  Block Type: [Default            ]  │
└─────────────────────────────────────┘
```

### 권장 메모리 맵

| 이름 | 시작 | 길이 | R | W | X |
|------|------|------|---|---|---|
| flash | 0x08000000 | 0x80000 | ✓ | | ✓ |
| sram | 0x20000000 | 0x10000 | ✓ | ✓ | |
| apb1 | 0x40000000 | 0x8000 | ✓ | ✓ | |
| apb2 | 0x40010000 | 0x8000 | ✓ | ✓ | |
| ahb | 0x40020000 | 0x4000 | ✓ | ✓ | |

### 효과

메모리 영역 추가 후:

```c
// Before
*(undefined4 *)0x40006400 = 0x10002;

// After (주소가 유효한 영역으로 인식됨)
CAN1_MCR = 0x10002;  // 구조체 정의하면 이렇게!
```

## 주변장치 구조체 정의

**Window → Data Type Manager**

### CAN_TypeDef 예시

```c
struct CAN_TypeDef {
    uint32_t MCR;      // 0x00 - Master Control
    uint32_t MSR;      // 0x04 - Master Status
    uint32_t TSR;      // 0x08 - Transmit Status
    uint32_t RF0R;     // 0x0C - Receive FIFO 0
    uint32_t RF1R;     // 0x10 - Receive FIFO 1
    uint32_t IER;      // 0x14 - Interrupt Enable
    uint32_t ESR;      // 0x18 - Error Status
    uint32_t BTR;      // 0x1C - Bit Timing
    // ... 계속
};
```

### 구조체 적용

1. `0x40006400` 주소로 이동
2. 우클릭 → **Data → CAN_TypeDef**
3. 라벨 지정: `CAN1`

디컴파일 결과:
```c
// Before
*(uint32_t *)0x40006400 = 0x10002;
*(uint32_t *)0x4000641c = 0x1c0006;

// After
CAN1->MCR = 0x10002;
CAN1->BTR = 0x1c0006;
```

**훨씬 읽기 쉽다!**

## 주소 패턴으로 기능 추정

| 주소 패턴 | 추정 기능 |
|-----------|-----------|
| 0x080xxxxx | Flash 접근 (코드, 상수) |
| 0x200xxxxx | RAM 변수 |
| 0x40006xxx | CAN 통신 |
| 0x40010xxx | GPIO 제어 |
| 0x40013xxx | USART1 |
| 0x40021xxx | RCC (클럭) |
| 0x40022xxx | Flash 프로그래밍 |

## 실전 분석: Flash 쓰기 함수 찾기

IAP 부트로더니까 **Flash 쓰기 코드**가 있을 것.

### Flash 레지스터 주소 검색

**Search → For Scalars → Value: 0x40022000**

결과:
```
0x08001234: ldr r0, =0x40022000
0x08001456: ldr r1, =0x40022000
```

### 해당 함수 분석

```c
void flash_unlock(void) {
    // FLASH_KEYR에 키 시퀀스 쓰기
    *(uint32_t *)0x40022004 = 0x45670123;  // KEY1
    *(uint32_t *)0x40022004 = 0xCDEF89AB;  // KEY2
}

void flash_erase_page(uint32_t addr) {
    *(uint32_t *)0x40022010 |= 0x02;       // FLASH_CR.PER = 1
    *(uint32_t *)0x40022014 = addr;        // FLASH_AR = addr
    *(uint32_t *)0x40022010 |= 0x40;       // FLASH_CR.STRT = 1
    while (*(uint32_t *)0x4002200C & 1);   // Wait BSY
}
```

**Flash 프로그래밍 로직 발견!**

## 메모리 맵 검증

### SP 초기값으로 검증

Vector Table에서:
```
Initial SP = 0x20005218
```

이 값이 SRAM 범위 안에 있어야 함:
```
0x20000000 ≤ 0x20005218 < 0x20010000 ✓
```

### 코드 주소로 검증

모든 함수 주소가 Flash 범위:
```
0x08000000 ≤ 0x08002800 < 0x08080000 ✓
```

### 전역 변수로 검증

데이터 참조 주소가 SRAM 범위:
```
0x20000000 ≤ 0x20000100 < 0x20010000 ✓
```

## 삽질: 잘못된 메모리 영역

메모리 맵 없이 분석하면:

```c
// Ghidra가 주소를 상수로 해석
FUN_08001000(0x40006400);  // CAN1 베이스인데 그냥 숫자

// 또는 잘못된 메모리 접근으로 분석 실패
*(undefined4 *)DAT_40006400 = ...  // undefined4?
```

메모리 맵 추가 후:
```c
CAN_Init(CAN1);  // 명확!
```

## 정리

| 주소 범위 | 영역 | 용도 |
|-----------|------|------|
| 0x08000000 | Flash | 코드, 상수 |
| 0x20000000 | SRAM | 변수, 스택 |
| 0x40000000 | APB1 | TIM, CAN, I2C, USART |
| 0x40010000 | APB2 | GPIO, SPI, USART1 |
| 0x40020000 | AHB | RCC, DMA, Flash |

**핵심**: 주소 보면 바로 영역 파악 → 기능 추정 가능!

**다음 글에서**: 부트로더 경계 찾기 - 16KB 확인, 앱 시작 주소 분석.

---

## 시리즈 목차

1. [왜 역분석을 하게 됐나](/posts/ghidra/ghidra-stm32-re-01/)
2. [HEX 파일 구조 이해하기](/posts/ghidra/ghidra-stm32-re-02/)
3. [Ghidra 설치와 첫 로드](/posts/ghidra/ghidra-stm32-re-03/)
4. [디스어셈블리 vs 디컴파일](/posts/ghidra/ghidra-stm32-re-04/)
5. [Vector Table 분석](/posts/ghidra/ghidra-stm32-re-05/)
6. **메모리 맵 추정하기** ← 현재 글
7. 부트로더 경계 찾기
8. ...

---

## 참고 자료

- [STM32F103 Reference Manual - Memory Map](https://www.st.com/resource/en/reference_manual/rm0008.pdf)
- [ARM Cortex-M3 Memory Map](https://developer.arm.com/documentation/dui0552/a/the-cortex-m3-processor/memory-model)
- [STM32F103 Datasheet](https://www.st.com/resource/en/datasheet/stm32f103ve.pdf)
