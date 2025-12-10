---
title: "Ghidra로 STM32 부트로더 역분석한 썰 #5 - Vector Table 분석"
date: 2024-12-10
draft: false
tags: ["Ghidra", "STM32", "리버스엔지니어링", "역분석", "ARM", "Vector Table", "인터럽트"]
categories: ["역분석"]
summary: "STM32 펌웨어의 출발점. Vector Table을 읽으면 전체 구조가 보인다."
---

## 지난 글 요약

[지난 글](/posts/ghidra/ghidra-stm32-re-04/)에서 디스어셈블리와 디컴파일의 차이를 배웠다. 이제 본격적으로 STM32 구조를 분석해보자. 첫 번째는 **Vector Table**.

## Vector Table이란?

ARM Cortex-M이 부팅할 때 **가장 먼저 읽는 테이블**.

```
0x08000000: Initial Stack Pointer (SP)
0x08000004: Reset_Handler 주소
0x08000008: NMI_Handler 주소
0x0800000C: HardFault_Handler 주소
...
```

CPU 전원이 들어오면:
1. `0x08000000`에서 SP 값 로드
2. `0x08000004`에서 Reset_Handler 주소 로드
3. Reset_Handler로 점프 → **펌웨어 시작!**

## 200429.hex Vector Table

Ghidra에서 `0x08000000`으로 이동 (G 키):

```
                     RAM
08000000 18 52 00 20     addr       DAT_20005218      ; Initial SP
08000004 01 28 00 08     addr       FUN_08002800+1    ; Reset
08000008 1f 2b 00 08     addr       FUN_08002b1e+1    ; NMI
0800000c ab 20 00 08     addr       FUN_080020aa+1    ; HardFault
08000010 79 20 00 08     addr       FUN_08002078+1    ; MemManage
08000014 00 00 00 00     addr       00000000          ; BusFault (미사용)
08000018 00 00 00 00     addr       00000000          ; UsageFault
0800001c 00 00 00 00     addr       00000000          ; Reserved
08000020 00 00 00 00     addr       00000000          ; Reserved
08000024 00 00 00 00     addr       00000000          ; Reserved
08000028 00 00 00 00     addr       00000000          ; Reserved
0800002c b1 20 00 08     addr       FUN_080020b0+1    ; SVCall
08000030 00 00 00 00     addr       00000000          ; Debug Monitor
08000034 00 00 00 00     addr       00000000          ; Reserved
08000038 b5 20 00 08     addr       FUN_080020b4+1    ; PendSV
0800003c b9 20 00 08     addr       FUN_080020b8+1    ; SysTick
```

## 리틀 엔디안 주의

바이트 순서가 뒤집혀 있다:

```
08000000: 18 52 00 20
          │  │  │  │
          └──┴──┴──┴─→ 0x20005218 (리틀 엔디안)
```

**읽는 법**: 뒤에서부터 `20 00 52 18` → `0x20005218`

## 핵심 엔트리 해석

### 1. Initial Stack Pointer (0x08000000)

```
값: 0x20005218
```

SRAM 영역 (`0x20000000` ~ `0x2000FFFF`) 안에 있다. ✓

STM32F103VE의 SRAM은 64KB. 스택은 SRAM **끝**에서 시작:

```
SRAM 시작: 0x20000000
SRAM 끝:   0x20010000 (64KB)
SP 초기값: 0x20005218

→ 스택 크기 ≈ 0x10000 - 0x5218 = 약 44KB
   (나머지는 .data, .bss, 힙 등)
```

### 2. Reset_Handler (0x08000004)

```
값: 0x08002801
실제 주소: 0x08002800 (Thumb 비트 제거)
```

**Thumb 비트?**

ARM Cortex-M은 Thumb 모드만 지원. 주소 LSB가 1이면 Thumb 모드 표시.

```
0x08002801 = 0x08002800 | 0x01 (Thumb)
```

Ghidra가 자동으로 `FUN_08002800+1`로 표시해준다.

### 3. Fault Handlers

```
NMI:        0x08002B1E  - Non-Maskable Interrupt
HardFault:  0x080020AA  - 심각한 오류
MemManage:  0x08002078  - 메모리 보호 위반
BusFault:   0x00000000  - 미사용
UsageFault: 0x00000000  - 미사용
```

**0x00000000**은 핸들러가 없다는 뜻. 해당 예외 발생 시 HardFault로 에스컬레이션.

### 4. System Handlers

```
SVCall:     0x080020B0  - Supervisor Call (RTOS용)
PendSV:     0x080020B4  - Pending SV (컨텍스트 스위칭)
SysTick:    0x080020B8  - 시스템 타이머
```

부트로더에서 RTOS 안 쓰면 보통 빈 함수거나 무한 루프.

## 외부 인터럽트 (IRQ)

`0x08000040`부터 외부 인터럽트 벡터:

```
08000040 bd 20 00 08     addr       FUN_080020bc+1    ; WWDG
08000044 c1 20 00 08     addr       FUN_080020c0+1    ; PVD
08000048 c5 20 00 08     addr       FUN_080020c4+1    ; TAMPER
0800004c c9 20 00 08     addr       FUN_080020c8+1    ; RTC
08000050 cd 20 00 08     addr       FUN_080020cc+1    ; FLASH
08000054 d1 20 00 08     addr       FUN_080020d0+1    ; RCC
...
08000090 dd 22 00 08     addr       FUN_080022dc+1    ; USB_LP_CAN1_RX0 ← CAN 수신!
...
```

### STM32F103 IRQ 번호

| 오프셋 | IRQ# | 이름 | 용도 |
|--------|------|------|------|
| 0x040 | 0 | WWDG | 윈도우 워치독 |
| 0x044 | 1 | PVD | 전압 감지 |
| 0x048 | 2 | TAMPER | 탬퍼 감지 |
| 0x04C | 3 | RTC | 실시간 클럭 |
| 0x050 | 4 | FLASH | 플래시 |
| 0x054 | 5 | RCC | 클럭 |
| 0x058 | 6 | EXTI0 | 외부 인터럽트 0 |
| ... | ... | ... | ... |
| 0x090 | 20 | USB_LP_CAN1_RX0 | **CAN 수신** |
| 0x094 | 21 | CAN1_RX1 | CAN FIFO1 |
| 0x098 | 22 | CAN1_SCE | CAN 에러 |

## 핵심 발견: CAN 인터럽트 핸들러

IAP 부트로더니까 **CAN 수신 핸들러**가 핵심!

```
0x08000090: USB_LP_CAN1_RX0 → FUN_080022dc
```

`0x080022DC`로 이동:

```asm
                     FUN_080022dc
080022dc 70 b5           push       { r4, r5, r6, lr }
080022de 0d 4c           ldr        r4, [DAT_08002314]
080022e0 01 25           movs       r5, #0x1
080022e2 a5 60           str        r5, [r4, #0x8]
080022e4 21 68           ldr        r1, [r4, #0x0]
080022e6 09 06           lsls       r1, r1, #0x18
080022e8 09 0e           lsrs       r1, r1, #0x18
...
```

디컴파일:

```c
void CAN1_RX0_IRQHandler(void)
{
  uint32_t *fifo = (uint32_t *)0x40006400;  // CAN1 base
  
  // FIFO에서 메시지 읽기
  uint8_t dlc = fifo[0] & 0xFF;
  uint32_t id = fifo[1];
  // ...
  
  // 메시지 처리
  process_can_message(id, data, dlc);
}
```

**IAP 프로토콜의 핵심 로직이 여기에!**

## Vector Table로 알 수 있는 것

### 1. 사용 중인 인터럽트

```
0이 아닌 핸들러 = 사용 중
0x00000000 = 미사용
```

이 부트로더가 사용하는 인터럽트:
- Reset, NMI, HardFault, MemManage
- SVCall, PendSV, SysTick
- CAN1_RX0 ← **핵심!**
- 기타 몇 개

### 2. 코드 영역 범위

모든 핸들러 주소가 `0x08002xxx` ~ `0x08003xxx` 범위.

```
부트로더 코드: 0x08002000 ~ 0x08003FFF (약 8KB)
Vector Table:  0x08000000 ~ 0x080001FF (512B)
중간 영역:     0x08000200 ~ 0x08001FFF (데이터?)
```

### 3. 폴트 핸들러 구현 수준

```
HardFault:  구현됨 (0x080020AA)
BusFault:   미구현 (0x00000000)
UsageFault: 미구현 (0x00000000)
```

→ 간단한 에러 처리만 구현. 프로덕션 코드 수준은 아님.

## Ghidra에서 라벨 지정

분석 편의를 위해 이름 지정 (L 키):

| 주소 | 원래 이름 | 변경 후 |
|------|-----------|---------|
| 0x08002800 | FUN_08002800 | Reset_Handler |
| 0x08002B1E | FUN_08002b1e | NMI_Handler |
| 0x080020AA | FUN_080020aa | HardFault_Handler |
| 0x080022DC | FUN_080022dc | CAN1_RX0_IRQHandler |

디컴파일 결과가 훨씬 읽기 쉬워진다:

```c
// Before
void FUN_080022dc(void) { ... }

// After
void CAN1_RX0_IRQHandler(void) { ... }
```

## 삽질: Thumb 비트 혼동

Vector Table에서 주소 따라갈 때:

```
0x08000004: 01 28 00 08 → 0x08002801

잘못: G 키 → 08002801 입력 → 엉뚱한 위치
올바름: G 키 → 08002800 입력 → Reset_Handler
```

Ghidra가 대부분 자동 처리하지만, 수동 탐색 시 주의.

## Vector Table 크기

STM32F103의 인터럽트는 총 68개:
- 시스템 예외: 16개 (0x00 ~ 0x3C)
- 외부 IRQ: 52개 (0x40 ~ 0x10C)

```
Vector Table 끝: 0x08000000 + 0x10C = 0x0800010C
크기: 약 268바이트
```

보통 512바이트(0x200) 정렬해서 배치.

## 정리

| 항목 | 값 | 의미 |
|------|-----|------|
| SP 초기값 | 0x20005218 | 스택 위치 |
| Reset_Handler | 0x08002800 | 진입점 |
| CAN1_RX0 | 0x080022DC | IAP 핵심 |
| 코드 범위 | 0x08002000~0x08003FFF | 약 8KB |

**다음 글에서**: 메모리 맵 추정하기 - Flash, RAM, 주변장치 영역 분석.

---

## 시리즈 목차

1. [왜 역분석을 하게 됐나](/posts/ghidra/ghidra-stm32-re-01/)
2. [HEX 파일 구조 이해하기](/posts/ghidra/ghidra-stm32-re-02/)
3. [Ghidra 설치와 첫 로드](/posts/ghidra/ghidra-stm32-re-03/)
4. [디스어셈블리 vs 디컴파일](/posts/ghidra/ghidra-stm32-re-04/)
5. **Vector Table 분석** ← 현재 글
6. 메모리 맵 추정하기
7. ...

---

## 참고 자료

- [ARM Cortex-M3 Vector Table](https://developer.arm.com/documentation/dui0552/a/the-cortex-m3-processor/exception-model/vector-table)
- [STM32F103 Reference Manual - Interrupts](https://www.st.com/resource/en/reference_manual/rm0008.pdf)
- [PM0056 - STM32 Cortex-M3 Programming Manual](https://www.st.com/resource/en/programming_manual/pm0056.pdf)
