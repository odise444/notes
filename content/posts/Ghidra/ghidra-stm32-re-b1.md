---
title: "Ghidra STM32 역분석 번외 #1 - Ghidra 스크립트"
date: 2024-12-14
draft: false
tags: ["Ghidra", "STM32", "리버스엔지니어링", "역분석", "Python"]
categories: ["역분석"]
series: ["Ghidra 역분석"]
summary: "반복 작업을 자동화하는 Ghidra 스크립트."
---

Ghidra는 Python/Java 스크립트를 지원한다. 반복 작업 자동화에 유용.

---

## 스크립트 위치

`Window` → `Script Manager`

또는 직접 실행: `Ctrl + Shift + S`

---

## 예제 1: 주변장치 라벨 자동 지정

```python
# STM32F1_Peripherals.py
from ghidra.program.model.symbol import SourceType

peripherals = {
    0x40000000: "TIM2",
    0x40000400: "TIM3",
    0x40000800: "TIM4",
    0x40004400: "USART2",
    0x40004800: "USART3",
    0x40005400: "I2C1",
    0x40005800: "I2C2",
    0x40006400: "CAN1",
    0x40010800: "GPIOA",
    0x40010C00: "GPIOB",
    0x40011000: "GPIOC",
    0x40011400: "GPIOD",
    0x40011800: "GPIOE",
    0x40013800: "USART1",
    0x40021000: "RCC",
    0x40022000: "FLASH_R",
    0xE000E010: "SysTick",
    0xE000E100: "NVIC",
    0xE000ED00: "SCB",
}

for addr, name in peripherals.items():
    try:
        createLabel(toAddr(addr), name, True, SourceType.USER_DEFINED)
        print(f"Created: {name} at 0x{addr:08X}")
    except:
        print(f"Failed: {name}")
```

---

## 예제 2: Vector Table 분석

```python
# ParseVectorTable.py
base = 0x08000000
vectors = [
    "Initial_SP", "Reset_Handler", "NMI_Handler", "HardFault_Handler",
    "MemManage_Handler", "BusFault_Handler", "UsageFault_Handler",
    "Reserved", "Reserved", "Reserved", "Reserved",
    "SVC_Handler", "DebugMon_Handler", "Reserved", "PendSV_Handler",
    "SysTick_Handler",
]

for i, name in enumerate(vectors):
    addr = base + i * 4
    value = getInt(toAddr(addr))
    
    if value != 0 and name != "Reserved":
        handler_addr = value & ~1  # Thumb 비트 제거
        try:
            createLabel(toAddr(handler_addr), name, True, SourceType.USER_DEFINED)
            createFunction(toAddr(handler_addr), name)
            print(f"{name}: 0x{handler_addr:08X}")
        except:
            pass
```

---

## 예제 3: 문자열 찾기

```python
# FindStrings.py
from ghidra.program.model.data import StringDataType

mem = currentProgram.getMemory()
flash = mem.getBlock("Flash")

addr = flash.getStart()
end = flash.getEnd()

while addr < end:
    data = getByte(addr)
    if 0x20 <= data <= 0x7E:  # printable
        # 문자열 시작?
        s = ""
        temp = addr
        while temp < end:
            b = getByte(temp) & 0xFF
            if b == 0:
                break
            if 0x20 <= b <= 0x7E:
                s += chr(b)
                temp = temp.add(1)
            else:
                break
        
        if len(s) > 4:
            print(f"0x{addr}: {s}")
            addr = temp
    
    addr = addr.add(1)
```

---

## 실행 방법

1. Script Manager 열기
2. 스크립트 파일 선택
3. Run 클릭

또는 `analyzeHeadless`로 배치 실행.

---

다음 번외편에서 IDA Pro vs Ghidra 비교.

[번외 #2 - IDA vs Ghidra](/posts/ghidra/ghidra-stm32-re-b2/)
