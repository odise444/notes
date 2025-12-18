---
title: "Ghidra STM32 역분석 #3 - Ghidra 첫 로드"
date: 2024-12-09
draft: false
tags: ["Ghidra", "STM32", "리버스엔지니어링", "역분석", "ARM"]
categories: ["역분석"]
series: ["Ghidra 역분석"]
summary: "Ghidra 설치하고 STM32 바이너리 로드. 설정 하나 잘못하면 분석 망한다."
---

## 설치

JDK 17 이상 필요. https://ghidra-sre.org/ 에서 다운받아서 압축 풀고 실행하면 끝.

```bash
./ghidraRun    # Linux/Mac
ghidraRun.bat  # Windows
```

---

## 프로젝트 생성

File → New Project → Non-Shared Project

프로젝트 이름 정하고 Finish.

---

## 바이너리 Import

File → Import File → `200429.bin` 선택

### Language 선택 (핵심)

Filter에 "ARM" 입력하고:

```
ARM:LE:32:Cortex ← 이거 선택
```

STM32는 전부 Cortex 시리즈니까 이거면 된다. v7이나 v4t 선택하면 Thumb-2 명령어 해석이 안 돼서 분석이 망한다.

### Base Address 설정 (핵심)

Options 버튼 클릭해서:

```
Base Address: 0x08000000
```

이거 안 하면 주소가 0x00000000부터 시작해서 엉망이 된다. Import 후에는 수정이 어려우니까 처음에 제대로 해야 한다.

---

## 자동 분석

Import 직후 "Analyze?" 물으면 Yes. 기본 옵션으로 충분하다.

16KB면 몇 초 만에 끝난다.

---

## 첫 확인

G 키 → `08000000` 입력해서 Vector Table 확인:

```
08000000 18 52 00 20     addr       DAT_20005218      ; SP
08000004 01 28 00 08     addr       FUN_08002800+1    ; Reset
08000008 1f 2b 00 08     addr       FUN_08002b1e+1    ; NMI
```

자동 분석이 Vector Table을 인식했다. Reset_Handler가 `0x08002800`인 것도 찾았다.

---

## Reset_Handler 확인

G 키 → `08002800` 이동:

```asm
08002800 10 b5           push       { r4, lr }
08002802 04 4c           ldr        r4, [DAT_08002814]
08002804 20 46           mov        r0, r4
```

오른쪽 Decompile 패널에 C 유사 코드가 나온다:

```c
void FUN_08002800(void)
{
  FUN_080028a4(DAT_08002814);
  FUN_080028f4(DAT_08002814);
  return;
}
```

이게 main() 또는 SystemInit() 같은 초기화 함수다.

---

## 주요 단축키

| 키 | 기능 |
|----|------|
| G | Go to address |
| F | Create function |
| L | Label (이름 지정) |
| ; | Comment |
| D | Disassemble |

---

다음 글에서 디스어셈블리와 디컴파일 차이.

[#4 - 디스어셈블리 vs 디컴파일](/posts/ghidra/ghidra-stm32-re-04/)
