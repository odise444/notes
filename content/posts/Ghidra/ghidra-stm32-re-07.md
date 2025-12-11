---
title: "Ghidra로 STM32 부트로더 역분석한 썰 #7 - 부트로더 경계 찾기"
date: 2024-12-10
draft: false
tags: ["Ghidra", "STM32", "리버스엔지니어링", "역분석", "부트로더", "메모리"]
categories: ["역분석"]
summary: "부트로더는 어디서 끝나고 앱은 어디서 시작하나? Flash 레이아웃을 파악하자."
---

## 지난 글 요약

[지난 글](/posts/ghidra/ghidra-stm32-re-06/)에서 STM32 메모리 맵을 분석했다. 이제 부트로더의 정확한 경계를 찾아보자.

## 왜 경계가 중요한가

IAP 부트로더는 Flash를 **영역별로 나눠서** 사용한다:

```
┌─────────────────┐
│   Application   │ ← 사용자 펌웨어
├─────────────────┤
│     Buffer      │ ← 수신 데이터 임시 저장
├─────────────────┤
│   Bootloader    │ ← 우리가 분석 중인 것
└─────────────────┘
```

경계를 알아야:
- 앱 시작 주소 확인 (점프 대상)
- 버퍼 크기 확인 (최대 펌웨어 크기)
- 부트로더 보호 영역 설정

## 방법 1: HEX 파일 크기로 추정

```bash
# 200429.hex 변환 후 크기
ls -la 200429.bin
# 16384 bytes = 16KB = 0x4000
```

**부트로더 크기: 16KB (0x4000)**

```
부트로더: 0x08000000 ~ 0x08003FFF
```

## 방법 2: 코드 끝 주소 확인

Ghidra에서 **마지막 함수** 찾기:

**Window → Functions** → 주소순 정렬

```
FUN_08000100  0x08000100
FUN_08000200  0x08000200
...
FUN_08003f00  0x08003f00  ← 거의 끝
```

또는 **Navigation → Go To → End of Block**

```
마지막 코드: 0x08003F70 근처
패딩 (0xFF): 0x08003F70 ~ 0x08003FFF
```

**검증**: 16KB 맞음!

## 방법 3: 앱 점프 코드에서 찾기

부트로더는 결국 **앱으로 점프**한다. 그 주소를 찾으면 됨.

### 점프 코드 패턴

```c
// 전형적인 앱 점프 코드
#define APP_ADDRESS  0x08004000

void jump_to_app(void) {
    uint32_t app_stack = *(uint32_t *)APP_ADDRESS;
    uint32_t app_entry = *(uint32_t *)(APP_ADDRESS + 4);
    
    __set_MSP(app_stack);
    ((void(*)(void))app_entry)();
}
```

### Ghidra에서 찾기

**Search → For Scalars → Value: 0x08004000**

결과:
```
0x08002100: ldr r0, =0x08004000
0x08002234: cmp r1, #0x08004000
```

`0x08002100` 함수 분석:

```asm
08002100 0b 48           ldr        r0, [DAT_08002130]   ; 0x08004000
08002102 01 68           ldr        r1, [r0, #0x0]       ; app_stack
08002104 41 68           ldr        r1, [r0, #0x4]       ; app_entry
08002106 81 46           mov        sp, r1               ; MSP 설정
08002108 08 47           bx         r1                   ; 점프!
```

디컴파일:
```c
void jump_to_application(void)
{
    uint32_t *app_base = (uint32_t *)0x08004000;
    __set_MSP(app_base[0]);
    ((void(*)(void))app_base[1])();
}
```

**앱 시작 주소: 0x08004000** 확정!

## 방법 4: Flash 쓰기 주소에서 찾기

IAP는 수신한 펌웨어를 Flash에 쓴다. 쓰기 대상 주소를 찾으면 버퍼/앱 영역 확인 가능.

### Flash 프로그래밍 함수 찾기

**Search → For Scalars → Value: 0x40022000** (FLASH 베이스)

Flash 쓰기 함수에서:
```c
void flash_write(uint32_t addr, uint16_t *data, uint32_t len) {
    // addr 범위 체크
    if (addr < 0x08004000 || addr >= 0x08042800) {
        return;  // 범위 밖
    }
    // ...
}
```

**발견한 주소들**:
- `0x08004000`: 버퍼/앱 시작
- `0x08042800`: 앱 시작 (버퍼 끝)
- `0x08080000`: Flash 끝

## 200429.hex 메모리 레이아웃

분석 결과 종합:

```
┌─────────────────────┐ 0x08080000 (512KB)
│                     │
│    Application      │
│      (246KB)        │
│   0x08042800 ~      │
│                     │
├─────────────────────┤ 0x08042800
│                     │
│    Buffer           │
│   (수신 펌웨어 저장) │
│      (254KB)        │
│   0x08004000 ~      │
│                     │
├─────────────────────┤ 0x08004000
│                     │
│    Bootloader       │
│      (16KB)         │
│   0x08000000 ~      │
│                     │
└─────────────────────┘ 0x08000000
```

### 크기 계산

| 영역 | 시작 | 끝 | 크기 |
|------|------|-----|------|
| Bootloader | 0x08000000 | 0x08003FFF | 16KB |
| Buffer | 0x08004000 | 0x080427FF | 254KB |
| Application | 0x08042800 | 0x0807FFFF | 246KB |

**버퍼가 254KB?** 꽤 크다. 전체 앱을 받아서 검증 후 복사하는 방식.

## 더블 버퍼 구조

왜 버퍼가 따로 있나?

```
1. PC → 부트로더: 펌웨어 전송 (Buffer에 저장)
2. 부트로더: CRC 검증
3. 검증 OK → Buffer → Application 복사
4. 앱으로 점프
```

**장점**: 전송 중 오류 시 기존 앱 보존

```
실패 시:
┌─────────────┐
│ App (정상)  │ ← 그대로 유지
├─────────────┤
│ Buffer (X)  │ ← 버림
├─────────────┤
│ Bootloader  │
└─────────────┘
```

## 앱 유효성 검사 코드

부트로더는 앱이 유효한지 검사 후 점프한다.

### Stack Pointer 검증

```c
bool is_app_valid(void) {
    uint32_t app_sp = *(uint32_t *)0x08042800;
    
    // SP가 SRAM 범위 안에 있는지 확인
    if (app_sp < 0x20000000 || app_sp > 0x20010000) {
        return false;  // 유효하지 않음
    }
    return true;
}
```

Ghidra에서 찾은 코드:
```asm
08001500 08 48           ldr        r0, [DAT_08001524]   ; 0x08042800
08001502 00 68           ldr        r0, [r0, #0x0]       ; app SP
08001504 0a 49           ldr        r1, [DAT_08001528]   ; 0x20000000
08001506 88 42           cmp        r0, r1
08001508 03 d3           blo        LAB_08001512         ; 실패
0800150a 09 49           ldr        r1, [DAT_0800152c]   ; 0x20010000
0800150c 88 42           cmp        r0, r1
0800150e 01 d2           bhs        LAB_08001512         ; 실패
08001510 01 20           movs       r0, #0x1             ; 성공
08001512 00 20           movs       r0, #0x0             ; 실패
```

### CRC 검증

```c
bool verify_app_crc(void) {
    uint32_t *app = (uint32_t *)0x08042800;
    uint32_t size = app[HEADER_SIZE_OFFSET];
    uint32_t stored_crc = app[HEADER_CRC_OFFSET];
    
    uint32_t calc_crc = crc32(&app[HEADER_LEN], size);
    
    return (calc_crc == stored_crc);
}
```

## 링커 스크립트 역추정

이 정보로 원래 링커 스크립트를 추정할 수 있다:

```ld
/* 부트로더용 링커 스크립트 */
MEMORY
{
    FLASH (rx)  : ORIGIN = 0x08000000, LENGTH = 16K
    RAM (rwx)   : ORIGIN = 0x20000000, LENGTH = 64K
}

/* 앱용 링커 스크립트 */
MEMORY
{
    FLASH (rx)  : ORIGIN = 0x08042800, LENGTH = 246K
    RAM (rwx)   : ORIGIN = 0x20000000, LENGTH = 64K
}
```

## 부트로더 진입 조건

언제 부트로더에 머무르고, 언제 앱으로 점프하나?

### 일반적인 조건들

```c
void bootloader_main(void) {
    // 조건 1: GPIO 체크 (버튼)
    if (GPIO_READ(BOOT_PIN) == 0) {
        enter_iap_mode();
    }
    
    // 조건 2: 앱 유효성
    if (!is_app_valid()) {
        enter_iap_mode();
    }
    
    // 조건 3: CAN 매직 메시지
    if (received_boot_command()) {
        enter_iap_mode();
    }
    
    // 조건 4: 타임아웃 (몇 초 대기)
    if (wait_for_command(3000) == TIMEOUT) {
        jump_to_app();
    }
}
```

### 이 부트로더의 진입 조건

분석 결과:
1. **CAN 메시지 대기** (0x30 Connection Request)
2. **타임아웃 시 앱 점프** (약 3초)
3. **앱 무효하면 IAP 모드 유지**

## 삽질: 경계 주소 혼동

처음에 앱 시작을 `0x08004000`으로 착각했다.

```
실제로는:
0x08004000 = 버퍼 시작
0x08042800 = 앱 시작
```

버퍼에 받아서 → 검증 후 → 앱 영역에 복사하는 구조였다.

**단서**: Flash 복사 함수에서 `0x08042800` 발견

```c
void copy_buffer_to_app(void) {
    flash_copy(0x08004000, 0x08042800, firmware_size);
}
```

## 정리

| 항목 | 주소 | 크기 |
|------|------|------|
| 부트로더 시작 | 0x08000000 | 16KB |
| 부트로더 끝 | 0x08003FFF | |
| 버퍼 시작 | 0x08004000 | 254KB |
| 버퍼 끝 | 0x080427FF | |
| 앱 시작 | 0x08042800 | 246KB |
| 앱 끝 | 0x0807FFFF | |

**다음 글에서**: 전역 변수 영역 분석 - .data, .bss, 구조체 추정.

---

## 시리즈 목차

1. [왜 역분석을 하게 됐나](/posts/ghidra/ghidra-stm32-re-01/)
2. [HEX 파일 구조 이해하기](/posts/ghidra/ghidra-stm32-re-02/)
3. [Ghidra 설치와 첫 로드](/posts/ghidra/ghidra-stm32-re-03/)
4. [디스어셈블리 vs 디컴파일](/posts/ghidra/ghidra-stm32-re-04/)
5. [Vector Table 분석](/posts/ghidra/ghidra-stm32-re-05/)
6. [메모리 맵 추정하기](/posts/ghidra/ghidra-stm32-re-06/)
7. **부트로더 경계 찾기** ← 현재 글
8. 전역 변수 영역 분석

---

## 참고 자료

- [STM32 Flash Memory Organization](https://www.st.com/resource/en/reference_manual/rm0008.pdf)
- [AN2606 - STM32 Bootloader](https://www.st.com/resource/en/application_note/an2606.pdf)
