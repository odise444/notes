---
title: "Ghidra STM32 역분석 #12 - Flash 접근 함수 찾기"
date: 2024-12-12
draft: false
tags: ["Ghidra", "STM32", "리버스엔지니어링", "역분석", "Flash", "IAP"]
categories: ["역분석"]
series: ["Ghidra 역분석"]
summary: "부트로더의 핵심! Flash Unlock, Erase, Program 함수를 찾아라."
---

IAP 부트로더의 핵심은 Flash 프로그래밍. Unlock, Erase, Write 함수를 찾자.

---

## Flash 레지스터

| 레지스터 | 주소 | 용도 |
|----------|------|------|
| FLASH_KEYR | 0x40022004 | Unlock Key |
| FLASH_SR | 0x4002200C | Status |
| FLASH_CR | 0x40022010 | Control |
| FLASH_AR | 0x40022014 | Erase Address |

---

## Flash Key

Unlock에 필요한 매직 넘버:

```c
#define FLASH_KEY1  0x45670123
#define FLASH_KEY2  0xCDEF89AB
```

---

## Ghidra에서 찾기

Search → For Scalars → `0x45670123`

```c
void Flash_Unlock(void) {
    *(uint32_t *)0x40022004 = 0x45670123;
    *(uint32_t *)0x40022004 = 0xCDEF89AB;
}
```

매직 넘버 순서대로 쓰면 잠금 해제.

---

## Erase 함수

Flash_Unlock 근처에 있다:

```c
void Flash_ErasePage(uint32_t addr) {
    *(uint32_t *)0x40022010 |= 0x02;    // PER (Page Erase)
    *(uint32_t *)0x40022014 = addr;     // 페이지 주소
    *(uint32_t *)0x40022010 |= 0x40;    // STRT
    while (*(uint32_t *)0x4002200C & 0x01);  // BSY 대기
    *(uint32_t *)0x40022010 &= ~0x02;
}
```

---

## Write 함수

```c
void Flash_WriteHalfWord(uint32_t addr, uint16_t data) {
    *(uint32_t *)0x40022010 |= 0x01;    // PG (Program)
    *(uint16_t *)addr = data;           // 16비트 쓰기
    while (*(uint32_t *)0x4002200C & 0x01);
    *(uint32_t *)0x40022010 &= ~0x01;
}
```

STM32 Flash는 Half-word(16비트) 단위로 쓴다.

---

다음 글에서 IAP 프로토콜 분석.

[#13 - IAP 프로토콜 분석](/posts/ghidra/ghidra-stm32-re-13/)
