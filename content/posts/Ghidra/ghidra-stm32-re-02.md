---
title: "Ghidra로 STM32 부트로더 역분석한 썰 #2 - HEX 파일 구조 이해하기"
date: 2024-12-09
draft: false
tags: ["Ghidra", "STM32", "리버스엔지니어링", "역분석", "Intel HEX", "임베디드"]
categories: ["역분석"]
summary: "HEX 파일이 뭔지도 모르고 역분석? 먼저 포맷부터 이해하자."
---

## 지난 글 요약

[지난 글](/posts/ghidra/ghidra-stm32-re-01/)에서 왜 역분석을 하게 됐는지 설명했다. 이제 본격적으로 시작하기 전에, HEX 파일이 뭔지부터 알아보자.

## Intel HEX 포맷

STM32 펌웨어 배포에 가장 많이 쓰이는 포맷. 텍스트 파일이라 메모장으로 열 수 있다.

```
:020000040800F2
:1000000018520020012800081F2B0008AB200008BE
:100010007920000800000000000000000000000048
:10002000000000000000000079010008000000003E
...
:00000001FF
```

**어? 바이너리 아니었어?**

HEX는 바이너리를 **텍스트로 인코딩**한 것이다.

## 레코드 구조

각 줄이 하나의 "레코드". 구조:

```
:LLAAAATT[DD...]CC

: = 시작 마커
LL = 데이터 길이 (바이트)
AAAA = 주소 (16비트)
TT = 레코드 타입
DD = 데이터 (LL 바이트)
CC = 체크섬
```

### 예시 분석

```
:1000000018520020012800081F2B0008AB200008BE
 ││    ││                              ││
 ││    ││                              └┴ 체크섬: 0xBE
 ││    │└ 타입: 00 (데이터)
 ││    └─ 주소: 0x0000
 │└────── 길이: 0x10 (16바이트)
 └─────── 시작 마커
```

**데이터**: `18520020012800081F2B0008AB200008`

16바이트가 주소 0x0000에 들어간다.

## 레코드 타입

| 타입 | 이름 | 설명 |
|------|------|------|
| 00 | Data | 실제 바이너리 데이터 |
| 01 | EOF | 파일 끝 |
| 02 | Extended Segment | 세그먼트 주소 (16비트 확장) |
| 04 | Extended Linear | 선형 주소 (32비트 확장) |
| 05 | Start Linear | 시작 주소 (진입점) |

### 타입 04: Extended Linear Address

STM32는 32비트 주소를 쓴다. 16비트 주소 필드로는 부족.

```
:020000040800F2
 ││    ││  ││
 ││    ││  └┴ 상위 16비트: 0x0800
 ││    │└ 타입: 04
 ││    └─ 주소: 0x0000 (무시)
 │└────── 길이: 0x02 (2바이트)
```

**의미**: 이후 데이터의 기준 주소가 `0x0800 << 16 = 0x08000000`

STM32 Flash 시작 주소가 `0x08000000`이니까 맞다!

### 타입 01: EOF

```
:00000001FF

길이=0, 타입=01 → 파일 끝
```

## 체크섬 계산

모든 바이트 합의 **2의 보수**.

```python
def calc_checksum(record):
    # ':' 제외하고 바이트 합산
    data = bytes.fromhex(record[1:-2])  # 체크섬 제외
    total = sum(data) & 0xFF
    checksum = (~total + 1) & 0xFF
    return checksum

# 검증
record = ":1000000018520020012800081F2B0008AB200008BE"
print(hex(calc_checksum(record)))  # 0xBE ✓
```

## 실제 파일 분석

`200429.hex` 앞부분:

```
:020000040800F2          ← 기준 주소 0x08000000 설정
:1000000018520020012800081F2B0008AB200008BE  ← Vector Table 시작!
:100010007920000800000000000000000000000048
:10002000000000000000000079010008000000003E
```

### Vector Table 해석

첫 데이터 레코드:
```
18520020 01280008 1F2B0008 AB200008
```

리틀 엔디안으로 읽으면:

| 오프셋 | 값 | 의미 |
|--------|-----|------|
| 0x00 | 0x20005218 | Initial SP |
| 0x04 | 0x08002801 | Reset_Handler |
| 0x08 | 0x08002B1F | NMI_Handler |
| 0x0C | 0x080020AB | HardFault_Handler |

**Reset_Handler = 0x08002801**

`0x08002800`이 아니라 `0x08002801`인 이유? **Thumb 모드** 표시. 실제 주소는 `0x08002800`.

## HEX → BIN 변환

Ghidra는 raw binary를 선호한다. 변환 방법:

### 방법 1: objcopy (권장)

```bash
arm-none-eabi-objcopy -I ihex -O binary 200429.hex 200429.bin
```

### 방법 2: Python

```python
from intelhex import IntelHex

ih = IntelHex('200429.hex')
ih.tobinfile('200429.bin')

# 시작 주소 확인
print(f"Start: 0x{ih.minaddr():08X}")  # 0x08000000
print(f"End:   0x{ih.maxaddr():08X}")  # 0x08003FFF
print(f"Size:  {ih.maxaddr() - ih.minaddr() + 1} bytes")  # 16384 = 16KB
```

### 방법 3: 온라인 변환기

급하면 [hex2bin.com](https://hex2bin.sourceforge.net/) 같은 사이트.

## 변환 결과 확인

```bash
# 파일 크기
ls -la 200429.bin
# -rw-r--r-- 1 user user 16384 Dec 09 10:00 200429.bin

# hexdump로 확인
hexdump -C 200429.bin | head -5
# 00000000  18 52 00 20 01 28 00 08  1f 2b 00 08 ab 20 00 08  |.R. .(...+... ..|
# 00000010  79 20 00 08 00 00 00 00  00 00 00 00 00 00 00 00  |y ..............|
```

HEX 파일 데이터와 동일한 것 확인!

## 주소 정보 손실 주의

BIN 변환 시 **시작 주소 정보가 사라진다**.

```
HEX: "이 데이터는 0x08000000부터야"
BIN: "그냥 바이너리 덩어리" (주소 모름)
```

Ghidra 로드 시 **Base Address를 수동 지정**해야 한다.

## 다중 영역 HEX

가끔 HEX 파일에 **불연속 영역**이 있다:

```
:020000040800F2          ← 0x08000000
:10000000...             ← 부트로더
...
:020000040804EE          ← 0x08040000 (갑자기 점프!)
:10000000...             ← 앱 시작
```

이런 경우 BIN 변환하면 **중간이 0xFF로 채워진다**. 파일 크기 급증!

```bash
# 해결: 영역별로 따로 추출
python -c "
from intelhex import IntelHex
ih = IntelHex('full.hex')
ih.tobinfile('boot.bin', start=0x08000000, end=0x08003FFF)
ih.tobinfile('app.bin', start=0x08040000, end=0x0807FFFF)
"
```

## 정리

| 항목 | 설명 |
|------|------|
| HEX 포맷 | 텍스트 기반, 주소 정보 포함 |
| 레코드 | `:LLAAAATTDDCC` 구조 |
| 타입 04 | 32비트 주소 확장 (STM32 필수) |
| 변환 | objcopy 또는 intelhex |
| 주의 | BIN 변환 시 주소 정보 손실 |

**다음 글에서**: Ghidra 설치하고 BIN 파일 로드하기.

---

## 시리즈 목차

1. [왜 역분석을 하게 됐나](/posts/ghidra/ghidra-stm32-re-01/)
2. **HEX 파일 구조 이해하기** ← 현재 글
3. Ghidra 설치와 첫 로드
4. 디스어셈블리 vs 디컴파일
5. Vector Table 분석
6. ...

---

## 참고 자료

- [Intel HEX Wikipedia](https://en.wikipedia.org/wiki/Intel_HEX)
- [intelhex Python 라이브러리](https://pypi.org/project/intelhex/)
- [ARM Cortex-M Vector Table](https://developer.arm.com/documentation/dui0552/a/the-cortex-m3-processor/exception-model/vector-table)
