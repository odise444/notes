---
title: "Ghidra로 STM32 부트로더 역분석한 썰 #4 - 디스어셈블리 vs 디컴파일"
date: 2024-12-10
draft: false
tags: ["Ghidra", "STM32", "리버스엔지니어링", "역분석", "ARM", "어셈블리", "디컴파일"]
categories: ["역분석"]
summary: "어셈블리를 읽어야 하나, 디컴파일된 C 코드를 봐야 하나? 둘 다 필요하다."
---

## 지난 글 요약

[지난 글](/posts/ghidra/ghidra-stm32-re-03/)에서 Ghidra 설치하고 바이너리를 로드했다. 이제 코드를 분석해야 하는데... 왼쪽엔 어셈블리, 오른쪽엔 C 코드. 뭘 봐야 하나?

## 두 가지 뷰

Ghidra CodeBrowser의 핵심 패널:

```
┌─────────────────────────────┬─────────────────────────────┐
│         Listing             │         Decompile           │
│      (디스어셈블리)          │        (디컴파일)            │
│                             │                             │
│  08002800 10 b5  push {r4,lr}│  void FUN_08002800(void)   │
│  08002802 04 4c  ldr r4,...  │  {                         │
│  08002804 20 46  mov r0,r4   │    FUN_080028a4(0x20000000);│
│  08002806 00 f0  bl ...      │    FUN_080028f4(0x20000000);│
│  ...                        │    return;                  │
│                             │  }                          │
└─────────────────────────────┴─────────────────────────────┘
```

## 디스어셈블리 (Listing)

**바이너리 → 어셈블리 명령어** 변환.

```asm
08002800 10 b5           push       { r4, lr }
08002802 04 4c           ldr        r4, [DAT_08002814]
08002804 20 46           mov        r0, r4
08002806 00 f0 4d f8     bl         FUN_080028a4
0800280a 20 46           mov        r0, r4
0800280c 00 f0 72 f8     bl         FUN_080028f4
08002810 10 bd           pop        { r4, pc }
```

### 구조

```
주소     바이트      명령어     오퍼랜드
08002800 10 b5       push      { r4, lr }
│        │           │         │
│        │           │         └─ 무엇을 (r4, lr 레지스터)
│        │           └─ 뭘 하는지 (스택에 저장)
│        └─ 기계어 코드 (CPU가 실행하는 것)
└─ 메모리 주소
```

### 장점

- **정확함**: 실제 CPU가 실행하는 그대로
- **최적화 코드 분석**: 컴파일러 트릭 파악 가능
- **타이밍 분석**: 명령어 수로 실행 시간 추정

### 단점

- **읽기 어려움**: C보다 10배 길다
- **학습 필요**: ARM 명령어 셋 숙지 필요

## 디컴파일 (Decompile)

**어셈블리 → C 유사 코드** 변환. Ghidra의 킬러 기능.

```c
void FUN_08002800(void)
{
  FUN_080028a4(DAT_08002814);
  FUN_080028f4(DAT_08002814);
  return;
}
```

### 장점

- **읽기 쉬움**: C 개발자면 바로 이해
- **흐름 파악**: if/for/while 구조 복원
- **빠른 분석**: 전체 구조 빨리 파악

### 단점

- **부정확할 수 있음**: 추정 기반
- **변수명 없음**: `param_1`, `local_10` 같은 자동 이름
- **최적화 역변환 한계**: 원본과 다를 수 있음

## ARM Thumb-2 기초

STM32(Cortex-M)는 **Thumb-2** 명령어 셋 사용.

### 자주 보는 명령어

| 명령어 | 의미 | 예시 |
|--------|------|------|
| `mov` | 값 복사 | `mov r0, r4` (r0 = r4) |
| `ldr` | 메모리 → 레지스터 | `ldr r0, [r1]` (r0 = *r1) |
| `str` | 레지스터 → 메모리 | `str r0, [r1]` (*r1 = r0) |
| `push` | 스택에 저장 | `push {r4, lr}` |
| `pop` | 스택에서 복원 | `pop {r4, pc}` |
| `bl` | 함수 호출 | `bl FUN_08001234` |
| `bx` | 분기 (레지스터) | `bx lr` (리턴) |
| `cmp` | 비교 | `cmp r0, #0` |
| `beq` | 같으면 분기 | `beq LAB_08001000` |
| `bne` | 다르면 분기 | `bne LAB_08001000` |

### 레지스터

| 레지스터 | 용도 |
|----------|------|
| r0-r3 | 함수 인자 / 리턴값 |
| r4-r11 | 로컬 변수 (callee-saved) |
| r12 | IP (Intra-Procedure) |
| r13 (sp) | 스택 포인터 |
| r14 (lr) | 링크 레지스터 (리턴 주소) |
| r15 (pc) | 프로그램 카운터 |

### 예시: 함수 호출

```asm
; 호출 전
08002804 20 46           mov        r0, r4      ; 첫 번째 인자
08002806 00 f0 4d f8     bl         FUN_080028a4 ; 함수 호출

; FUN_080028a4(r4)와 동일
```

디컴파일 결과:
```c
FUN_080028a4(DAT_08002814);
```

## 실전 예시: 간단한 함수

### 어셈블리

```asm
                     FUN_08001000
08001000 10 b5           push       { r4, lr }
08001002 04 46           mov        r4, r0          ; r4 = param1
08001004 00 28           cmp        r0, #0x0        ; if (param1 == 0)
08001006 03 d0           beq        LAB_08001010    ;   goto LAB
08001008 20 46           mov        r0, r4
0800100a 00 f0 10 f8     bl         FUN_0800102e    ; 함수 호출
0800100e 01 e0           b          LAB_08001014    ; else 건너뛰기
                     LAB_08001010
08001010 4f f0 ff 30     mov        r0, #-0x1       ; return -1
08001014 10 bd           pop        { r4, pc }      ; return
```

### 디컴파일

```c
int FUN_08001000(int param_1)
{
  int iVar1;
  
  if (param_1 == 0) {
    iVar1 = -1;
  }
  else {
    iVar1 = FUN_0800102e(param_1);
  }
  return iVar1;
}
```

**훨씬 읽기 쉽다!**

## 디컴파일 오류 사례

### 사례 1: switch-case 인식 실패

원본 C:
```c
switch (cmd) {
  case 0x30: handle_connect(); break;
  case 0x31: handle_auth(); break;
  case 0x32: handle_size(); break;
}
```

디컴파일 (잘못된 경우):
```c
if (cmd == 0x30) {
  handle_connect();
}
if (cmd == 0x31) {  // else if 아님!
  handle_auth();
}
// ...
```

**어셈블리로 점프 테이블 확인 필요.**

### 사례 2: 비트 연산 복원 실패

원본:
```c
REG->CR |= (1 << 4);
```

디컴파일:
```c
*(uint *)0x40021000 = *(uint *)0x40021000 | 0x10;
```

의미는 같지만 가독성이 떨어진다. 수동으로 해석 필요.

### 사례 3: 구조체 미인식

원본:
```c
uart->BRR = 0x1D4C;
uart->CR1 = 0x200C;
```

디컴파일:
```c
*(undefined4 *)0x40013808 = 0x1d4c;
*(undefined4 *)0x4001380c = 0x200c;
```

구조체 정의를 추가해야 깔끔해진다.

## 언제 뭘 볼까?

| 상황 | 추천 뷰 |
|------|---------|
| 전체 흐름 파악 | 디컴파일 |
| 함수 역할 추정 | 디컴파일 |
| 정확한 동작 확인 | 디스어셈블리 |
| 레지스터 접근 분석 | 디스어셈블리 |
| 타이밍 크리티컬 코드 | 디스어셈블리 |
| 컴파일러 최적화 분석 | 디스어셈블리 |

**결론**: 디컴파일로 빠르게 파악 → 의심되면 어셈블리 확인.

## Ghidra 팁: 디컴파일 품질 향상

### 1. 함수 시그니처 수정

```c
// 자동 생성
void FUN_08001000(void)

// 수정 후 (함수 선택 → 우클릭 → Edit Function Signature)
int CAN_Init(CAN_TypeDef *canx, uint32_t baudrate)
```

### 2. 변수 타입 지정

디컴파일 창에서 변수 우클릭 → **Retype Variable**

```c
// Before
*(undefined4 *)0x40006400 = 0;

// After (CAN_TypeDef 구조체 정의 후)
CAN1->MCR = 0;
```

### 3. 구조체 정의

**Window → Data Type Manager → 우클릭 → New Structure**

```c
struct CAN_TypeDef {
  uint32_t MCR;   // 0x00
  uint32_t MSR;   // 0x04
  uint32_t TSR;   // 0x08
  // ...
};
```

## 실습: Reset_Handler 분석

### 디스어셈블리

```asm
                     FUN_08002800
08002800 10 b5           push       { r4, lr }
08002802 04 4c           ldr        r4, [DAT_08002814]
08002804 20 46           mov        r0, r4
08002806 00 f0 4d f8     bl         FUN_080028a4
0800280a 20 46           mov        r0, r4
0800280c 00 f0 72 f8     bl         FUN_080028f4
08002810 10 bd           pop        { r4, pc }
08002812 00 bf           nop
                     DAT_08002814
08002814 00 00 00 20     addr       0x20000000
```

### 해석

1. `push {r4, lr}` - 프롤로그 (r4, 리턴주소 저장)
2. `ldr r4, [DAT_08002814]` - r4 = 0x20000000 (SRAM 시작)
3. `mov r0, r4` - 첫 번째 인자로 전달
4. `bl FUN_080028a4` - 함수 호출
5. `mov r0, r4` - 다시 인자 설정
6. `bl FUN_080028f4` - 또 다른 함수 호출
7. `pop {r4, pc}` - 에필로그 (리턴)

### 디컴파일

```c
void Reset_Handler(void)
{
  SystemInit(0x20000000);
  main(0x20000000);
  return;
}
```

**추정**: `FUN_080028a4`는 `SystemInit`, `FUN_080028f4`는 `main` 또는 부트로더 메인 루프.

## 정리

| 항목 | 디스어셈블리 | 디컴파일 |
|------|--------------|----------|
| 정확도 | 100% | 90%+ |
| 가독성 | 낮음 | 높음 |
| 분석 속도 | 느림 | 빠름 |
| 용도 | 상세 분석 | 전체 파악 |

**다음 글에서**: Vector Table 상세 분석, 인터럽트 핸들러 찾기.

---

## 시리즈 목차

1. [왜 역분석을 하게 됐나](/posts/ghidra/ghidra-stm32-re-01/)
2. [HEX 파일 구조 이해하기](/posts/ghidra/ghidra-stm32-re-02/)
3. [Ghidra 설치와 첫 로드](/posts/ghidra/ghidra-stm32-re-03/)
4. **디스어셈블리 vs 디컴파일** ← 현재 글
5. Vector Table 분석
6. ...

---

## ARM 명령어 참고

- [ARM Cortex-M3 Instruction Set](https://developer.arm.com/documentation/ddi0337/e/Instruction-Timing)
- [Thumb-2 Quick Reference](https://developer.arm.com/documentation/qrc0001/m/)
