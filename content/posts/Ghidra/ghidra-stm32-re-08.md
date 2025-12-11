---
title: "Ghidra로 STM32 부트로더 역분석한 썰 #8 - 전역 변수 영역 분석"
date: 2024-12-11
draft: false
tags: ["Ghidra", "STM32", "리버스엔지니어링", "역분석", "RAM", "변수"]
categories: ["역분석"]
summary: ".data, .bss가 뭔지 모르면 전역 변수 분석이 안 된다. RAM 영역을 해부해보자."
---

## 지난 글 요약

[지난 글](/posts/ghidra/ghidra-stm32-re-07/)에서 부트로더 경계(16KB)와 앱 시작 주소(0x08042800)를 찾았다. 이제 RAM에 있는 전역 변수들을 분석해보자.

## 전역 변수는 어디에?

C 프로그램의 메모리 구조:

```
┌─────────────────┐ 높은 주소
│      Stack      │ ← 지역 변수, 함수 호출
├─────────────────┤
│      Heap       │ ← malloc()
├─────────────────┤
│      .bss       │ ← 초기화 안 된 전역 변수
├─────────────────┤
│      .data      │ ← 초기화된 전역 변수
├─────────────────┤ 낮은 주소
```

STM32에서:
- `.data`, `.bss`, `Heap` → **SRAM** (0x20000000)
- `Stack` → **SRAM 끝**에서 아래로 성장

## .data vs .bss

| 섹션 | 내용 | 예시 | Flash 사용 |
|------|------|------|------------|
| .data | 초기값 있는 전역 변수 | `int count = 100;` | 초기값 저장 |
| .bss | 초기값 없는 전역 변수 | `int buffer[256];` | 없음 (0으로 초기화) |

```c
// .data 섹션
int initialized_var = 0x12345678;
const char *message = "Hello";

// .bss 섹션
uint8_t rx_buffer[1024];
volatile uint32_t tick_count;
```

## Ghidra에서 RAM 영역 확인

### 1. SRAM 참조 찾기

**Search → For Scalars → Value: 0x20000000**

또는 디컴파일에서 `0x200xxxxx` 패턴 찾기:

```c
void some_function(void) {
    *(uint32_t *)0x20000100 = 0;           // 전역 변수 접근
    *(uint32_t *)0x20000104 = rx_data;     // 또 다른 변수
    buffer = (uint8_t *)0x20000200;        // 버퍼 주소
}
```

### 2. 데이터 참조 목록

**Window → Symbol References**

RAM 주소 참조 목록:
```
0x20000000  referenced by FUN_08002000
0x20000004  referenced by FUN_08002000, FUN_08002100
0x20000100  referenced by FUN_08002200
0x20000200  referenced by FUN_08002300 (buffer)
...
```

## 초기화 코드에서 .data/.bss 찾기

### Reset_Handler 분석

스타트업 코드는 `.data` 복사와 `.bss` 초기화를 한다:

```asm
                     Reset_Handler
08002800 10 b5           push       { r4, lr }
08002802 0c 4c           ldr        r4, [DAT_08002834]    ; _sdata
08002804 0d 4d           ldr        r5, [DAT_08002838]    ; _edata
08002806 0e 4e           ldr        r6, [DAT_0800283c]    ; _sidata (Flash)
                     
                     ; .data 복사 루프 (Flash → RAM)
08002808 a5 42           cmp        r5, r4
0800280a 04 d0           beq        LAB_08002816
0800280c 36 68           ldr        r6, [r6, #0x0]
0800280e 26 60           str        r6, [r4, #0x0]
08002810 04 34           adds       r4, #0x4
08002812 04 36           adds       r6, #0x4
08002814 f8 e7           b          LAB_08002808
                     
                     ; .bss 초기화 (0으로 채우기)
08002816 08 4c           ldr        r4, [DAT_08002840]    ; _sbss
08002818 09 4d           ldr        r5, [DAT_08002844]    ; _ebss
0800281a 00 26           movs       r6, #0x0
0800281c a5 42           cmp        r5, r4
0800281e 02 d0           beq        LAB_08002826
08002820 26 60           str        r6, [r4, #0x0]
08002822 04 34           adds       r4, #0x4
08002824 fa e7           b          LAB_0800281c
```

### 링커 심볼 추출

데이터 참조에서 링커 심볼 발견:

```
DAT_08002834: 0x20000000  ; _sdata (RAM .data 시작)
DAT_08002838: 0x20000100  ; _edata (RAM .data 끝)
DAT_0800283c: 0x08003800  ; _sidata (Flash 초기값)
DAT_08002840: 0x20000100  ; _sbss (RAM .bss 시작)
DAT_08002844: 0x20001000  ; _ebss (RAM .bss 끝)
```

**RAM 레이아웃 확정!**

```
0x20000000 ~ 0x200000FF: .data (256 bytes)
0x20000100 ~ 0x20000FFF: .bss  (약 4KB)
0x20001000 ~ 0x20005217: Heap/여유
0x20005218 ~ 0x2000FFFF: Stack (약 44KB)
```

## 전역 변수 구조체 추정

### 연속된 접근 패턴

```c
void CAN_Handler(void) {
    *(uint32_t *)0x20000200 = msg_id;       // +0x00
    *(uint8_t *)0x20000204 = msg_dlc;       // +0x04
    *(uint8_t *)0x20000205 = msg_data[0];   // +0x05
    *(uint8_t *)0x20000206 = msg_data[1];   // +0x06
    // ...
}
```

**추정 구조체:**

```c
typedef struct {
    uint32_t id;        // 0x00
    uint8_t dlc;        // 0x04
    uint8_t data[8];    // 0x05 ~ 0x0C
    uint8_t padding[3]; // 정렬
} CAN_Message_t;        // 총 16 bytes

CAN_Message_t rx_msg;   // @ 0x20000200
```

### Ghidra에서 구조체 적용

1. **Window → Data Type Manager**
2. 우클릭 → **New → Structure**
3. 필드 추가:

```
┌─────────────────────────────────────┐
│         Structure Editor            │
├─────────────────────────────────────┤
│  Name: CAN_Message_t                │
│  Size: 16                           │
├─────────────────────────────────────┤
│  Offset  Type      Name             │
│  0x00    uint      id               │
│  0x04    byte      dlc              │
│  0x05    byte[8]   data             │
│  0x0D    byte[3]   padding          │
└─────────────────────────────────────┘
```

4. `0x20000200` 주소에 적용:
   - 주소로 이동 (G → `20000200`)
   - 우클릭 → **Data → CAN_Message_t**
   - 라벨 지정 (L → `rx_msg`)

### 적용 후 디컴파일

```c
// Before
*(uint32_t *)0x20000200 = param_1;
*(byte *)0x20000204 = (byte)param_2;

// After
rx_msg.id = param_1;
rx_msg.dlc = (byte)param_2;
```

**훨씬 읽기 쉽다!**

## 배열 발견하기

### 반복 접근 패턴

```asm
08001500 00 24           movs       r4, #0x0
08001502 0a 4d           ldr        r5, [DAT_0800152c]    ; 0x20000300
                     LAB_08001504
08001504 28 5d           ldrb       r0, [r5, r4]          ; buffer[i]
08001506 00 28           cmp        r0, #0x0
08001508 05 d0           beq        LAB_08001516
0800150a 01 34           adds       r4, #0x1
0800150c b4 42           cmp        r4, r6                ; i < len
0800150e f9 d1           bne        LAB_08001504
```

**패턴 분석:**
- `0x20000300`: 배열 시작
- `r4`: 인덱스 (0부터 시작)
- `ldrb [r5, r4]`: 바이트 배열 접근

**추정:**
```c
uint8_t buffer[256];  // @ 0x20000300
```

### 배열 타입 적용

1. `0x20000300`으로 이동
2. 우클릭 → **Data → byte[256]**
3. 라벨: `rx_buffer`

## 상태 변수 찾기

IAP 부트로더는 **상태 머신**으로 동작. 상태 변수가 있을 것.

### 상태 체크 패턴

```c
void iap_process(void) {
    switch (*(uint32_t *)0x20000010) {  // state
    case 0:  // IDLE
        break;
    case 1:  // CONNECTED
        break;
    case 2:  // RECEIVING
        break;
    case 3:  // VERIFYING
        break;
    }
}
```

### 상태 변수 정의

```c
typedef enum {
    IAP_STATE_IDLE = 0,
    IAP_STATE_CONNECTED = 1,
    IAP_STATE_RECEIVING = 2,
    IAP_STATE_VERIFYING = 3,
    IAP_STATE_COMPLETE = 4
} iap_state_t;

iap_state_t g_iap_state;  // @ 0x20000010
```

## 발견한 전역 변수 목록

| 주소 | 타입 | 이름 (추정) | 용도 |
|------|------|-------------|------|
| 0x20000000 | uint32_t | g_system_tick | 시스템 틱 |
| 0x20000004 | uint32_t | g_timeout_cnt | 타임아웃 카운터 |
| 0x20000010 | uint32_t | g_iap_state | IAP 상태 |
| 0x20000014 | uint32_t | g_page_num | 현재 페이지 번호 |
| 0x20000018 | uint32_t | g_total_size | 펌웨어 크기 |
| 0x20000100 | uint32_t | g_fw_checksum | 펌웨어 체크섬 |
| 0x20000200 | CAN_Message_t | g_rx_msg | CAN 수신 메시지 |
| 0x20000300 | uint8_t[2048] | g_page_buffer | 페이지 버퍼 |

## 삽질: 정렬 문제

ARM은 **4바이트 정렬**을 선호한다.

```c
// 잘못된 추정
struct Wrong {
    uint8_t a;      // 0x00
    uint32_t b;     // 0x01 ← 정렬 오류!
};

// 올바른 추정
struct Correct {
    uint8_t a;      // 0x00
    uint8_t pad[3]; // 0x01 ~ 0x03 (패딩)
    uint32_t b;     // 0x04
};
```

**팁**: `uint32_t` 필드는 항상 4의 배수 주소에 있다.

## 삽질: volatile 변수

인터럽트에서 수정하는 변수는 `volatile`.

```c
// 메인 루프
while (*(uint32_t *)0x20000000 < 1000);  // 무한 루프?

// 인터럽트
void SysTick_Handler(void) {
    (*(uint32_t *)0x20000000)++;  // 여기서 증가
}
```

Ghidra는 `volatile`을 자동 인식 못 한다. 수동으로 판단해야.

**단서**: 인터럽트 핸들러와 메인 루프에서 **동시에 접근**하는 변수.

## 정적 분석 한계

컴파일러 최적화로 변수가 사라지거나 합쳐질 수 있다:

```c
// 원본
int a = 1;
int b = 2;
int c = a + b;

// 최적화 후
int c = 3;  // a, b 사라짐
```

**해결**: 동적 분석 (디버거) 병행 추천.

## RAM 맵 최종 정리

```
┌─────────────────────┐ 0x20010000
│                     │
│       Stack         │
│     (44KB 여유)      │
│                     │
├─────────────────────┤ 0x20005218 (Initial SP)
│                     │
│    Heap / 여유      │
│                     │
├─────────────────────┤ 0x20001000
│                     │
│       .bss          │
│  (전역 변수, 버퍼)    │
│     (약 4KB)        │
│                     │
├─────────────────────┤ 0x20000100
│       .data         │
│   (초기화된 변수)    │
│     (256 bytes)     │
└─────────────────────┘ 0x20000000
```

## 정리

| 항목 | 방법 |
|------|------|
| .data/.bss 경계 | Reset_Handler 초기화 코드 분석 |
| 구조체 추정 | 연속된 필드 접근 패턴 |
| 배열 발견 | 반복문 + 인덱스 접근 |
| 상태 변수 | switch-case 패턴 |
| 정렬 주의 | 4바이트 정렬 필수 |

**Part 2 완료!** 다음은 주변장치 역분석으로 넘어간다.

---

## 시리즈 목차

**Part 1: 리버스 엔지니어링 입문편** ✅
1. [왜 역분석을 하게 됐나](/posts/ghidra/ghidra-stm32-re-01/)
2. [HEX 파일 구조 이해하기](/posts/ghidra/ghidra-stm32-re-02/)
3. [Ghidra 설치와 첫 로드](/posts/ghidra/ghidra-stm32-re-03/)
4. [디스어셈블리 vs 디컴파일](/posts/ghidra/ghidra-stm32-re-04/)

**Part 2: STM32 구조 분석편** ✅
5. [Vector Table 분석](/posts/ghidra/ghidra-stm32-re-05/)
6. [메모리 맵 추정하기](/posts/ghidra/ghidra-stm32-re-06/)
7. [부트로더 경계 찾기](/posts/ghidra/ghidra-stm32-re-07/)
8. **전역 변수 영역 분석** ← 현재 글

**Part 3: 주변장치 역분석편**
9. RCC 설정 복원하기
10. GPIO 초기화 분석
11. CAN 초기화 역분석
12. Flash 접근 함수 찾기

---

## 참고 자료

- [ARM Cortex-M3 Memory Model](https://developer.arm.com/documentation/dui0552/a/the-cortex-m3-processor/memory-model)
- [GCC Linker Script (.ld)](https://sourceware.org/binutils/docs/ld/Scripts.html)
- [STM32 Startup Code](https://www.st.com/resource/en/application_note/an4325.pdf)
