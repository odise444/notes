---
title: "STM32 부트로더 개발기 #2 - STM32 메모리 맵"
date: 2024-12-17
tags: ["STM32", "Bootloader", "Flash", "메모리"]
categories: ["개발기"]
series: ["STM32 부트로더 개발기"]
summary: "Flash가 어떻게 생겼는지부터 알아야 한다."
---

부트로더 만들려면 메모리 구조를 알아야 한다.

어디에 부트로더 넣고, 어디에 앱 넣을지 정해야 하니까.

---

## STM32F103VE 메모리 맵

512KB Flash, 64KB RAM.

```
주소              크기      용도
─────────────────────────────────────
0x0800_0000      512KB    Flash
0x2000_0000      64KB     SRAM
0x4000_0000      ~        Peripherals
0x1FFF_F000      2KB      System Memory (내장 부트로더)
```

Flash가 0x08000000부터 시작.

---

## Flash 구조

F103은 페이지 단위로 관리된다.

```
STM32F103VE (512KB):
- 페이지 크기: 2KB
- 페이지 수: 256개
- 0x0800_0000 ~ 0x0807_FFFF
```

**주의**: F103 시리즈마다 페이지 크기가 다름.

| 칩 | Flash | 페이지 크기 |
|----|-------|-------------|
| F103C8 (Bluepill) | 64KB | 1KB |
| F103RB | 128KB | 1KB |
| F103VE | 512KB | 2KB |
| F103ZE | 512KB | 2KB |

---

## 부트로더 영역 설계

내가 정한 레이아웃:

```
0x0800_0000 ┌─────────────────────┐
            │    Bootloader       │  16KB (8페이지)
0x0800_4000 ├─────────────────────┤
            │    Buffer           │  254KB (127페이지)
            │    (펌웨어 수신용)   │
0x0804_2800 ├─────────────────────┤
            │    Application      │  246KB (123페이지)
            │                     │
0x0808_0000 └─────────────────────┘
```

**왜 이렇게?**

1. **부트로더 16KB** - 충분히 여유 있음
2. **버퍼 영역** - 수신한 펌웨어 임시 저장
3. **앱 영역** - 실제 펌웨어

버퍼가 있어서 수신 중 전원 꺼져도 기존 앱은 살아있음.

---

## 주소 계산

```c
#define FLASH_BASE          0x08000000
#define BOOTLOADER_SIZE     0x4000      // 16KB
#define BUFFER_ADDR         (FLASH_BASE + BOOTLOADER_SIZE)  // 0x08004000
#define BUFFER_SIZE         0x3E800     // 254KB
#define APP_ADDR            (BUFFER_ADDR + BUFFER_SIZE)     // 0x08042800
```

---

## Vector Table

STM32는 부팅 시 0x08000000의 Vector Table을 읽는다.

```
0x0800_0000: Initial SP (Stack Pointer)
0x0800_0004: Reset Handler 주소
0x0800_0008: NMI Handler 주소
0x0800_000C: HardFault Handler 주소
...
```

부트로더가 0x08000000에 있으니까 먼저 실행됨.

앱으로 점프할 때 VTOR 레지스터로 Vector Table 위치를 바꿔줘야 함.

---

## RAM 사용

부트로더와 앱이 RAM을 공유한다.

```
0x2000_0000 ┌─────────────────────┐
            │    (공유)           │
            │                     │
0x2001_0000 └─────────────────────┘
```

부트로더에서 앱으로 점프하면 RAM은 초기화 안 됨.

특정 주소에 매직 값 써놓으면 앱에서 읽을 수 있음. 소프트웨어 리셋 트리거 용도로 활용.

---

다음 글에서 영역 분리 실습.

[#3 - 부트로더 vs 앱 영역 분리](/posts/stm32-bootloader/stm32-bootloader-03/)
