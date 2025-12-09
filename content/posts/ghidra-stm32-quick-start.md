---
title: "Ghidra로 STM32 펌웨어 역분석하기 - 빠른 시작 가이드"
date: 2024-12-09
draft: false
tags: ["Ghidra", "STM32", "리버스엔지니어링", "역분석", "임베디드"]
categories: ["역분석"]
summary: "HEX 파일 하나로 부트로더 구조를 파악하는 방법. Ghidra 설치부터 함수 분석까지."
---

## Ghidra란?

![](Pasted%20image%2020251209100003.png)


NSA에서 공개한 무료 리버스 엔지니어링 도구. IDA Pro 대안으로 ARM Cortex-M 분석에 충분하다.

## 1. 설치

**다운로드**: https://ghidra-sre.org/

**요구사항**: JDK 17 이상

```bash
# Windows
ghidraRun.bat

# Linux/Mac
./ghidraRun
```

## 2. HEX → BIN 변환

Ghidra는 raw binary를 선호한다.

```bash
# objcopy 사용
objcopy -I ihex -O binary firmware.hex firmware.bin

# 또는 Python
pip install intelhex
python -c "from intelhex import IntelHex; ih=IntelHex('firmware.hex'); ih.tobinfile('firmware.bin')"
```

## 3. 프로젝트 생성 & 파일 로드

1. **File → New Project** → Non-Shared Project
2. **File → Import File** → firmware.bin 선택
3. **Language 설정** (중요!):

```
Processor: ARM
Variant: v7
Size: 32
Endian: little
Compiler: default

→ "ARM:LE:32:Cortex" 선택
```

4. **Options → Base Address**: `0x08000000` (STM32 Flash 시작)

## 4. 자동 분석

Import 후 "Analyze?" 물으면 **Yes**.

주요 분석 옵션 (기본값 OK):
- ✅ ARM Aggressive Instruction Finder
- ✅ Create Address Tables
- ✅ Decompiler Parameter ID

## 5. Vector Table 확인

STM32는 0x08000000부터 Vector Table이 있다.

```
0x08000000: Initial SP (예: 0x20010000)
0x08000004: Reset_Handler 주소
0x08000008: NMI_Handler
0x0800000C: HardFault_Handler
...
```

**Reset_Handler 찾기**:
1. `0x08000004` 주소로 이동 (G 키)
2. 4바이트 값 확인 (예: `0x08000101`)
3. 해당 주소로 이동 → **메인 진입점!**

## 6. 함수 만들기

자동 분석이 놓친 함수는 수동 생성:

1. 함수 시작 주소로 이동
2. **F 키** → Create Function
3. **L 키** → 이름 지정 (예: `CAN_Init`)

## 7. 디컴파일 창 활용

함수 선택하면 오른쪽에 **C 유사 코드** 표시.

```c
void FUN_08000a00(void) {
    *(uint *)0x40021018 = *(uint *)0x40021018 | 4;  // RCC 설정
    *(uint *)0x40010c00 = 0x44444444;               // GPIO 설정
    ...
}
```

**해석 팁**:
- `0x4002xxxx` → RCC (클럭)
- `0x4001xxxx` → GPIO
- `0x4000xxxx` → CAN, UART 등

## 8. 주요 단축키

| 키 | 기능 |
|----|------|
| G | Go to address |
| D | Disassemble |
| F | Create function |
| L | Label (이름 지정) |
| T | Set data type |
| ; | Comment 추가 |
| Ctrl+E | 함수 그래프 |

## 9. 문자열 검색

**Search → For Strings**

펌웨어에 남은 디버그 문자열로 기능 추정:

```
"CAN Init OK"
"Flash Erase"
"Boot Mode"
```

## 10. 상수 검색

**Search → For Scalars**

CAN ID, 명령 코드 등 찾기:

```
Value: 0x5FF  → CAN 수신 ID
Value: 0x30   → IAP 명령 코드
```

## 실전 예시: CAN 초기화 찾기

1. **0x40006400** 검색 (CAN1 베이스 주소)
2. 해당 주소 참조하는 함수들 확인
3. 레지스터 접근 패턴으로 기능 추정

```c
// Ghidra 디컴파일 결과
*(uint *)0x40006400 = 0;           // CAN_MCR = 0 (초기화 모드 해제)
*(uint *)0x4000641c = 0x001c0006;  // CAN_BTR (타이밍 설정)
```

## 메모리 맵 참고 (STM32F103)

| 주소 | 영역 |
|------|------|
| 0x08000000 | Flash (부트로더/앱) |
| 0x20000000 | SRAM |
| 0x40000000 | APB1 (CAN, UART, TIM) |
| 0x40010000 | APB2 (GPIO, SPI, ADC) |
| 0x40020000 | AHB (RCC, DMA) |

## 다음 단계

1. **Vector Table** → Reset_Handler → main() 흐름 파악
2. **주변장치 초기화** 함수들 이름 지정
3. **핵심 로직** (부트로더면 Flash 쓰기, CAN 수신 등) 분석
4. **C 코드로 복원** → 빌드 → 바이너리 비교

---

## 참고 자료

- [Ghidra 공식 문서](https://ghidra-sre.org/)
- [STM32F103 Reference Manual](https://www.st.com/resource/en/reference_manual/rm0008.pdf)
- [ARM Cortex-M3 Technical Reference](https://developer.arm.com/documentation/ddi0337/latest)
