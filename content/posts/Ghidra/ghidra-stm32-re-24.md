---
title: "Ghidra STM32 역분석 #24 - 구조체 크기 추정"
date: 2024-12-14
draft: false
tags: ["Ghidra", "STM32", "리버스엔지니어링", "역분석", "구조체"]
categories: ["역분석"]
series: ["Ghidra 역분석"]
summary: "배열 인덱싱으로 구조체 크기를 역산하는 방법."
---

전역 변수 영역에 구조체 배열이 있는 것 같았다. 크기를 어떻게 알아낼까?

---

## 힌트: 배열 접근 패턴

```c
*(uint8_t *)(0x20000400 + i * 0x10) = value;
```

`i * 0x10`이 보인다. 구조체 크기가 16바이트라는 뜻.

---

## 어셈블리로 보면

```asm
ldr   r0, =0x20000400   ; 배열 시작
mov   r1, #0x10         ; 구조체 크기
mul   r2, r3, r1        ; index * size
add   r0, r0, r2        ; 주소 계산
strb  r4, [r0]          ; 저장
```

`#0x10 = 16`. 구조체 크기 16바이트.

---

## 패딩 함정

C 구조체는 정렬 때문에 패딩이 들어간다.

```c
struct Cell {
    uint16_t voltage;   // 2바이트
    uint8_t status;     // 1바이트
    // 1바이트 패딩
    uint32_t timestamp; // 4바이트
};  // 총 8바이트
```

필드 합은 7바이트인데 실제 크기는 8바이트.

---

## 실제 삽질

처음에 이렇게 추정했다:

```c
struct Cell {
    uint16_t voltage;
    uint8_t balance;
    uint8_t status;
    uint32_t raw;
    uint32_t filtered;
    uint16_t temp;
};  // 14바이트?
```

근데 배열 인덱싱은 16바이트 단위였다. 2바이트가 어디서 나왔나?

```c
struct Cell {
    uint16_t voltage;
    uint8_t balance;
    uint8_t status;
    uint32_t raw;
    uint32_t filtered;
    uint16_t temp;
    uint16_t reserved;  // 패딩 또는 예약
};  // 16바이트
```

마지막에 2바이트 패딩이 있었다.

---

## 구조체 크기 찾는 법

1. 배열 접근 코드에서 곱셈 찾기
2. `mul` 또는 `lsl` (shift left) 찾기
3. 상수가 구조체 크기

```asm
lsl r0, r1, #4    ; r0 = r1 * 16
```

`#4`는 2^4 = 16. 구조체 크기 16바이트.

---

다음 글에서 인라인 함수 vs 매크로.

[#25 - 인라인 함수 vs 매크로](/posts/ghidra/ghidra-stm32-re-25/)
