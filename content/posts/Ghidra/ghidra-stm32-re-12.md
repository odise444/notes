---
title: "Ghidra로 STM32 부트로더 역분석한 썰 #12 - Flash 접근 함수 찾기"
date: 2024-12-12
draft: false
tags: ["Ghidra", "STM32", "리버스엔지니어링", "역분석", "Flash", "IAP"]
categories: ["역분석"]
summary: "부트로더의 핵심! Flash Unlock, Erase, Program 함수를 찾아라."
---

## 지난 글 요약

[지난 글](/posts/ghidra/ghidra-stm32-re-11/)에서 CAN 초기화를 분석했다. 500kbps, ID 0x5FF 수신. 이제 **Flash 프로그래밍** 함수를 찾을 차례. IAP 부트로더의 **핵심**이다.

## STM32F103 Flash 특성

| 항목 | 값 |
|------|-----|
| 총 용량 | 512KB |
| 페이지 크기 | 2KB (고밀도 제품) |
| 쓰기 단위 | 16비트 (Half-word) |
| Erase 단위 | 페이지 (2KB) |
| 쓰기 전 | 반드시 Erase 필요 |

## Flash 레지스터 주소

| 레지스터 | 주소 | 용도 |
|----------|------|------|
| FLASH_ACR | 0x40022000 | Access Control |
| FLASH_KEYR | 0x40022004 | Key (Unlock) |
| FLASH_SR | 0x4002200C | Status |
| FLASH_CR | 0x40022010 | Control |
| FLASH_AR | 0x40022014 | Address (Erase) |

## Flash Key 값

Unlock에 필요한 매직 넘버:

```c
#define FLASH_KEY1  0x45670123
#define FLASH_KEY2  0xCDEF89AB
```

## Ghidra에서 Flash 접근 찾기

**Search → For Scalars → Value: 0x45670123**

발견!
```
08002D00: ldr r0, =0x45670123
08002D04: str r0, [0x40022004]   ; FLASH_KEYR
08002D08: ldr r0, =0xCDEF89AB
08002D0C: str r0, [0x40022004]
```

**Unlock 함수 발견!**

## Flash Unlock 함수 분석

```c
void FUN_08002D00(void) {
    // Key 시퀀스로 Unlock
    *(uint32_t *)0x40022004 = 0x45670123;  // KEY1
    *(uint32_t *)0x40022004 = 0xCDEF89AB;  // KEY2
}
```

## Flash Lock 함수

```c
void FUN_08002D20(void) {
    // LOCK 비트 설정
    *(uint32_t *)0x40022010 |= 0x80;  // CR.LOCK = 1
}
```

## Flash Erase 함수 분석

**Search → For Scalars → Value: 0x40022010** (FLASH_CR)

발견된 함수:

```c
void FUN_08002D40(uint32_t page_addr) {
    // BSY 플래그 대기
    while (*(uint32_t *)0x4002200C & 0x01);
    
    // PER 비트 설정 (Page Erase)
    *(uint32_t *)0x40022010 |= 0x02;
    
    // 페이지 주소 설정
    *(uint32_t *)0x40022014 = page_addr;
    
    // STRT 비트로 Erase 시작
    *(uint32_t *)0x40022010 |= 0x40;
    
    // 완료 대기
    while (*(uint32_t *)0x4002200C & 0x01);
    
    // PER 클리어
    *(uint32_t *)0x40022010 &= ~0x02;
}
```

**FLASH_CR 비트**:
```
Bit 0: PG     - Programming
Bit 1: PER    - Page Erase
Bit 2: MER    - Mass Erase
Bit 4: OPTPG  - Option Byte Programming
Bit 6: STRT   - Start
Bit 7: LOCK   - Lock
```

## Flash Program 함수 분석

```c
void FUN_08002D80(uint32_t addr, uint16_t data) {
    // BSY 대기
    while (*(uint32_t *)0x4002200C & 0x01);
    
    // PG 비트 설정
    *(uint32_t *)0x40022010 |= 0x01;
    
    // Half-word 쓰기
    *(volatile uint16_t *)addr = data;
    
    // 완료 대기
    while (*(uint32_t *)0x4002200C & 0x01);
    
    // PG 클리어
    *(uint32_t *)0x40022010 &= ~0x01;
}
```

## 다중 데이터 쓰기 함수

```c
void FUN_08002DC0(uint32_t addr, uint8_t *data, uint32_t len) {
    uint16_t *src = (uint16_t *)data;
    uint32_t count = len / 2;
    
    for (uint32_t i = 0; i < count; i++) {
        FUN_08002D80(addr + i * 2, src[i]);
    }
}
```

## 페이지 단위 쓰기 함수

IAP에서 사용하는 고수준 함수:

```c
void FUN_08002E00(uint32_t page_addr, uint8_t *data) {
    // 1. Unlock
    FUN_08002D00();
    
    // 2. Erase
    FUN_08002D40(page_addr);
    
    // 3. Program (2KB)
    FUN_08002DC0(page_addr, data, 2048);
    
    // 4. Lock
    FUN_08002D20();
}
```

## 에러 처리 분석

```c
uint32_t FUN_08002E40(void) {
    uint32_t sr = *(uint32_t *)0x4002200C;  // FLASH_SR
    
    // 에러 플래그 체크
    if (sr & 0x14) {  // WRPRTERR | PGERR
        // 에러 플래그 클리어
        *(uint32_t *)0x4002200C = 0x14;
        return 1;  // 에러
    }
    
    return 0;  // 성공
}
```

**FLASH_SR 비트**:
```
Bit 0: BSY      - Busy
Bit 2: PGERR    - Programming Error
Bit 4: WRPRTERR - Write Protection Error
Bit 5: EOP      - End of Operation
```

## 복원된 Flash 드라이버

```c
#define FLASH_KEY1      0x45670123
#define FLASH_KEY2      0xCDEF89AB
#define FLASH_BASE      0x40022000

#define FLASH_ACR       (*(volatile uint32_t *)(FLASH_BASE + 0x00))
#define FLASH_KEYR      (*(volatile uint32_t *)(FLASH_BASE + 0x04))
#define FLASH_SR        (*(volatile uint32_t *)(FLASH_BASE + 0x0C))
#define FLASH_CR        (*(volatile uint32_t *)(FLASH_BASE + 0x10))
#define FLASH_AR        (*(volatile uint32_t *)(FLASH_BASE + 0x14))

// Flash Unlock
void Flash_Unlock(void) {
    FLASH_KEYR = FLASH_KEY1;
    FLASH_KEYR = FLASH_KEY2;
}

// Flash Lock
void Flash_Lock(void) {
    FLASH_CR |= (1 << 7);  // LOCK
}

// Wait for BSY
static void Flash_WaitBusy(void) {
    while (FLASH_SR & 0x01);
}

// Page Erase (2KB)
HAL_StatusTypeDef Flash_ErasePage(uint32_t page_addr) {
    Flash_WaitBusy();
    
    FLASH_CR |= (1 << 1);   // PER
    FLASH_AR = page_addr;
    FLASH_CR |= (1 << 6);   // STRT
    
    Flash_WaitBusy();
    
    FLASH_CR &= ~(1 << 1);  // Clear PER
    
    // 에러 체크
    if (FLASH_SR & 0x14) {
        FLASH_SR = 0x14;    // Clear errors
        return HAL_ERROR;
    }
    
    return HAL_OK;
}

// Program Half-word
HAL_StatusTypeDef Flash_ProgramHalfWord(uint32_t addr, uint16_t data) {
    Flash_WaitBusy();
    
    FLASH_CR |= (1 << 0);   // PG
    
    *(volatile uint16_t *)addr = data;
    
    Flash_WaitBusy();
    
    FLASH_CR &= ~(1 << 0);  // Clear PG
    
    // 에러 체크
    if (FLASH_SR & 0x14) {
        FLASH_SR = 0x14;
        return HAL_ERROR;
    }
    
    // 검증
    if (*(volatile uint16_t *)addr != data) {
        return HAL_ERROR;
    }
    
    return HAL_OK;
}

// Program Buffer
HAL_StatusTypeDef Flash_ProgramBuffer(uint32_t addr, uint8_t *data, uint32_t len) {
    uint16_t *src = (uint16_t *)data;
    
    for (uint32_t i = 0; i < len / 2; i++) {
        if (Flash_ProgramHalfWord(addr + i * 2, src[i]) != HAL_OK) {
            return HAL_ERROR;
        }
    }
    
    return HAL_OK;
}

// Write Page (Erase + Program)
HAL_StatusTypeDef Flash_WritePage(uint32_t page_addr, uint8_t *data) {
    Flash_Unlock();
    
    if (Flash_ErasePage(page_addr) != HAL_OK) {
        Flash_Lock();
        return HAL_ERROR;
    }
    
    if (Flash_ProgramBuffer(page_addr, data, 2048) != HAL_OK) {
        Flash_Lock();
        return HAL_ERROR;
    }
    
    Flash_Lock();
    return HAL_OK;
}
```

## IAP에서 Flash 사용 패턴

```c
// 수신된 펌웨어 저장
void IAP_WriteFirmware(uint32_t offset, uint8_t *data, uint32_t len) {
    uint32_t addr = BUFFER_START + offset;  // 0x08004000
    
    // 페이지 경계 체크
    if ((addr % 2048) == 0) {
        // 새 페이지 시작 → Erase 필요
        Flash_Unlock();
        Flash_ErasePage(addr);
    }
    
    // 데이터 쓰기
    Flash_ProgramBuffer(addr, data, len);
    
    if ((addr + len) % 2048 == 0) {
        // 페이지 끝 → Lock
        Flash_Lock();
    }
}

// 버퍼 → 앱 영역 복사
void IAP_CopyToApp(void) {
    uint32_t src = BUFFER_START;     // 0x08004000
    uint32_t dst = APP_START;        // 0x08042800
    uint32_t size = g_firmware_size;
    
    Flash_Unlock();
    
    // 앱 영역 Erase
    for (uint32_t addr = dst; addr < dst + size; addr += 2048) {
        Flash_ErasePage(addr);
    }
    
    // 복사
    for (uint32_t i = 0; i < size; i += 2) {
        uint16_t data = *(uint16_t *)(src + i);
        Flash_ProgramHalfWord(dst + i, data);
    }
    
    Flash_Lock();
}
```

## 삽질: Unlock 순서

Key 순서가 틀리면 **Hard Fault**:

```c
// 잘못된 순서
FLASH_KEYR = FLASH_KEY2;  // ❌
FLASH_KEYR = FLASH_KEY1;

// 올바른 순서
FLASH_KEYR = FLASH_KEY1;  // ✅ 먼저!
FLASH_KEYR = FLASH_KEY2;
```

## 삽질: BSY 체크 누락

BSY 중에 접근하면 **Bus Error**:

```c
// 잘못된 코드
FLASH_CR |= PER;
FLASH_CR |= STRT;
FLASH_CR &= ~PER;  // ❌ 아직 Erase 중!

// 올바른 코드
FLASH_CR |= PER;
FLASH_CR |= STRT;
while (FLASH_SR & BSY);  // ✅ 대기
FLASH_CR &= ~PER;
```

## 삽질: Half-word 정렬

16비트 쓰기는 **짝수 주소**만:

```c
// 잘못된 코드
Flash_ProgramHalfWord(0x08004001, data);  // ❌ 홀수 주소

// 올바른 코드
Flash_ProgramHalfWord(0x08004000, data);  // ✅ 짝수 주소
```

## Ghidra 라벨 정리

```
주소 → 라벨
─────────────────────────────
0x40022000 → FLASH_ACR
0x40022004 → FLASH_KEYR
0x4002200C → FLASH_SR
0x40022010 → FLASH_CR
0x40022014 → FLASH_AR
FUN_08002D00 → Flash_Unlock
FUN_08002D20 → Flash_Lock
FUN_08002D40 → Flash_ErasePage
FUN_08002D80 → Flash_ProgramHalfWord
FUN_08002DC0 → Flash_ProgramBuffer
FUN_08002E00 → Flash_WritePage
```

## 정리

| 함수 | 주소 | 기능 |
|------|------|------|
| Flash_Unlock | 0x08002D00 | KEY1, KEY2 시퀀스 |
| Flash_Lock | 0x08002D20 | CR.LOCK 설정 |
| Flash_ErasePage | 0x08002D40 | 2KB 페이지 삭제 |
| Flash_ProgramHalfWord | 0x08002D80 | 16비트 쓰기 |
| Flash_WritePage | 0x08002E00 | Erase + Program |

**Part 3 완료!** 🎉

주변장치 초기화 분석 완료:
- ✅ RCC: 72MHz 클럭
- ✅ GPIO: CAN 핀, LED, 버튼
- ✅ CAN: 500kbps, ID 0x5FF
- ✅ Flash: Unlock, Erase, Program

**다음 Part**: CAN IAP 프로토콜 역분석 - 명령 코드, 상태 머신.

---

## 시리즈 목차

**Part 3: 주변장치 역분석편** ✅
- [#9 - RCC 설정 복원하기](/posts/ghidra/ghidra-stm32-re-09/)
- [#10 - GPIO 초기화 분석](/posts/ghidra/ghidra-stm32-re-10/)
- [#11 - CAN 초기화 역분석](/posts/ghidra/ghidra-stm32-re-11/)
- **#12 - Flash 접근 함수 찾기** ← 현재 글

**Part 4: CAN IAP 프로토콜 역분석편**
- #13 - CAN 수신 핸들러 분석
- #14 - 명령 코드 체계 파악
- #15 - 상태 머신 복원
- ...

---

## 참고 자료

- [STM32F103 Reference Manual - Flash](https://www.st.com/resource/en/reference_manual/rm0008.pdf)
- [STM32 Flash Programming Manual (PM0075)](https://www.st.com/resource/en/programming_manual/pm0075.pdf)
