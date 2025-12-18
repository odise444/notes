---
title: "Ghidra STM32 역분석 #2 - HEX 파일 구조"
date: 2024-12-09
draft: false
tags: ["Ghidra", "STM32", "리버스엔지니어링", "역분석", "Intel HEX"]
categories: ["역분석"]
series: ["Ghidra 역분석"]
summary: "HEX 파일이 뭔지도 모르고 역분석? 먼저 포맷부터 이해하자."
---

STM32 펌웨어 배포에 가장 많이 쓰이는 포맷. 메모장으로 열어보면:

```
:020000040800F2
:1000000018520020012800081F2B0008AB200008BE
:100010007920000800000000000000000000000048
...
:00000001FF
```

바이너리인 줄 알았는데 텍스트다. HEX는 바이너리를 텍스트로 인코딩한 것.

---

## 레코드 구조

각 줄이 하나의 "레코드":

```
:LLAAAATT[DD...]CC

: = 시작 마커
LL = 데이터 길이
AAAA = 주소 (16비트)
TT = 레코드 타입
DD = 데이터
CC = 체크섬
```

예시:

```
:1000000018520020012800081F2B0008AB200008BE
 ││    ││                              ││
 ││    │└ 타입: 00 (데이터)              └┴ 체크섬
 ││    └─ 주소: 0x0000
 └┴────── 길이: 0x10 (16바이트)
```

---

## 레코드 타입

| 타입 | 의미 |
|------|------|
| 00 | 데이터 |
| 01 | 파일 끝 |
| 04 | 32비트 주소 확장 |

STM32는 32비트 주소인데 레코드의 주소 필드는 16비트다. 타입 04로 상위 16비트를 지정한다:

```
:020000040800F2
           └┴── 상위 16비트: 0x0800
```

이후 데이터의 기준 주소가 `0x0800 << 16 = 0x08000000`. STM32 Flash 시작 주소다.

---

## Vector Table 확인

첫 데이터 레코드를 리틀 엔디안으로 읽으면:

```
18520020 → 0x20005218 (Initial SP)
01280008 → 0x08002801 (Reset_Handler)
1F2B0008 → 0x08002B1F (NMI_Handler)
AB200008 → 0x080020AB (HardFault_Handler)
```

`0x08002801`에서 마지막 `1`은 Thumb 모드 표시. 실제 주소는 `0x08002800`.

---

## HEX → BIN 변환

Ghidra는 raw binary를 선호한다.

```bash
objcopy -I ihex -O binary 200429.hex 200429.bin
```

또는 Python:

```python
from intelhex import IntelHex
ih = IntelHex('200429.hex')
ih.tobinfile('200429.bin')
print(f"Size: {ih.maxaddr() - ih.minaddr() + 1} bytes")  # 16384 = 16KB
```

---

## 주의: 주소 정보 손실

BIN 변환하면 시작 주소 정보가 사라진다. Ghidra 로드할 때 Base Address를 `0x08000000`으로 수동 지정해야 한다.

---

다음 글에서 Ghidra에 로드한다.

[#3 - Ghidra 첫 로드](/posts/ghidra/ghidra-stm32-re-03/)
