---
title: "Ghidra로 STM32 펌웨어 역분석 - 빠른 시작"
date: 2024-12-09
draft: false
tags: ["Ghidra", "STM32", "리버스엔지니어링", "역분석", "임베디드"]
categories: ["역분석"]
series: ["Ghidra 역분석"]
summary: "HEX 파일 하나로 부트로더 구조 파악하기. Ghidra 설치부터 함수 분석까지."
---

NSA에서 공개한 무료 리버스 엔지니어링 도구 Ghidra. IDA Pro 대안으로 ARM Cortex-M 분석에 충분하다.

![Ghidra 메인 화면](/imgs/Pasted%20image%2020251209100003.png)

---

## 설치

https://ghidra-sre.org/ 에서 다운로드. JDK 17 이상 필요.

```bash
# Windows
ghidraRun.bat

# Linux/Mac
./ghidraRun
```

---

## HEX → BIN 변환

Ghidra는 raw binary를 선호한다.

```bash
objcopy -I ihex -O binary firmware.hex firmware.bin
```

---

## 프로젝트 생성 & 파일 로드

File → New Project → Non-Shared Project

File → Import File → firmware.bin 선택

![Import 화면](/imgs/Pasted%20image%2020251209100328.png)

Language 설정이 중요하다:

```
Processor: ARM
Variant: v7
Size: 32
Endian: little

→ "ARM:LE:32:Cortex" 선택
```

![Language 선택](/imgs/ghidra-stm32-quick-start-1.png)

Options → Base Address: `0x08000000` (STM32 Flash 시작)

---

## 자동 분석

![분석 옵션](/imgs/ghidra-stm32-quick-start-2.png)

Import 후 "Analyze?" 물으면 Yes. 기본 옵션으로 충분하다.

---

## Vector Table 확인

STM32는 0x08000000부터 Vector Table이 있다.

```
0x08000000: Initial SP (예: 0x20010000)
0x08000004: Reset_Handler 주소
0x08000008: NMI_Handler
0x0800000C: HardFault_Handler
```

Reset_Handler 찾기:
1. G 키 → `0x08000004` 입력
2. 4바이트 값 확인 (예: `0x08000101`)
3. 해당 주소로 이동 → 메인 진입점

---

## 함수 만들기

자동 분석이 놓친 함수는 수동 생성:

- F 키 → Create Function
- L 키 → 이름 지정 (예: `CAN_Init`)

---

## 디컴파일 창

함수 선택하면 오른쪽에 C 유사 코드가 나온다.

```c
void FUN_08000a00(void) {
    *(uint *)0x40021018 = *(uint *)0x40021018 | 4;  // RCC 설정
    *(uint *)0x40010c00 = 0x44444444;               // GPIO 설정
}
```

주소 해석:
- `0x4002xxxx` → RCC (클럭)
- `0x4001xxxx` → GPIO
- `0x4000xxxx` → CAN, UART 등

---

## 주요 단축키

| 키 | 기능 |
|----|------|
| G | Go to address |
| D | Disassemble |
| F | Create function |
| L | Label (이름 지정) |
| ; | Comment 추가 |

---

## 문자열 검색

Search → For Strings

펌웨어에 남은 디버그 문자열로 기능 추정:

```
"CAN Init OK"
"Flash Erase"
"Boot Mode"
```

---

## STM32F103 메모리 맵

| 주소 | 영역 |
|------|------|
| 0x08000000 | Flash |
| 0x20000000 | SRAM |
| 0x40000000 | APB1 (CAN, UART) |
| 0x40010000 | APB2 (GPIO, SPI) |
| 0x40020000 | AHB (RCC, DMA) |

---

다음 글에서 실제 부트로더 역분석 시작.

[#1 - 왜 역분석을 하게 됐나](/posts/ghidra/ghidra-stm32-re-01/)
