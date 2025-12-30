---
title: "STM32 부트로더 개발기 #4 - 링커 스크립트 설정"
date: 2024-12-17
tags: ["STM32", "Bootloader", "링커", "ld"]
categories: ["개발기"]
series: ["STM32 부트로더 개발기"]
summary: "링커 스크립트가 뭔지 몰라도 됐는데, 부트로더 만들려니 알아야 한다."
---

CubeMX가 알아서 만들어주는 링커 스크립트. 그동안 건드릴 일 없었다.

근데 부트로더 만들려면 이해해야 한다.

---

## 링커 스크립트란

`.c` 파일 컴파일하면 `.o` (오브젝트) 파일 생김.

링커가 `.o` 파일들을 묶어서 최종 바이너리 만듦.

**링커 스크립트**: 뭘 어디에 배치할지 알려주는 설정 파일.

---

## 기본 구조

```ld
/* 메모리 영역 정의 */
MEMORY
{
  RAM    (xrw) : ORIGIN = 0x20000000, LENGTH = 64K
  FLASH  (rx)  : ORIGIN = 0x08000000, LENGTH = 512K
}

/* 섹션 배치 */
SECTIONS
{
  .isr_vector : { ... } > FLASH
  .text       : { ... } > FLASH
  .data       : { ... } > RAM AT> FLASH
  .bss        : { ... } > RAM
}
```

---

## MEMORY 블록

```ld
MEMORY
{
  RAM    (xrw) : ORIGIN = 0x20000000, LENGTH = 64K
  FLASH  (rx)  : ORIGIN = 0x08000000, LENGTH = 512K
}
```

- `xrw`: 실행(x), 읽기(r), 쓰기(w) 가능
- `rx`: 실행, 읽기만 (Flash는 런타임에 쓰기 불가)
- `ORIGIN`: 시작 주소
- `LENGTH`: 크기

---

## 부트로더용 수정

```ld
MEMORY
{
  RAM    (xrw) : ORIGIN = 0x20000000, LENGTH = 64K
  FLASH  (rx)  : ORIGIN = 0x08000000, LENGTH = 16K  /* 16KB만 */
}
```

16KB 넘으면 빌드 에러:

```
region `FLASH' overflowed by 1234 bytes
```

---

## 앱용 수정

```ld
MEMORY
{
  RAM    (xrw) : ORIGIN = 0x20000000, LENGTH = 64K
  FLASH  (rx)  : ORIGIN = 0x08042800, LENGTH = 246K
}
```

시작 주소가 0x08042800.

---

## SECTIONS 블록

```ld
SECTIONS
{
  /* Vector Table - 제일 먼저 */
  .isr_vector :
  {
    . = ALIGN(4);
    KEEP(*(.isr_vector))
    . = ALIGN(4);
  } >FLASH

  /* 코드 */
  .text :
  {
    . = ALIGN(4);
    *(.text)
    *(.text*)
    . = ALIGN(4);
  } >FLASH

  /* 상수 데이터 */
  .rodata :
  {
    . = ALIGN(4);
    *(.rodata)
    *(.rodata*)
    . = ALIGN(4);
  } >FLASH
}
```

- `.isr_vector`: 인터럽트 벡터 테이블
- `.text`: 실행 코드
- `.rodata`: const 데이터

---

## 초기화 데이터

```ld
_sidata = LOADADDR(.data);

.data :
{
  . = ALIGN(4);
  _sdata = .;
  *(.data)
  *(.data*)
  . = ALIGN(4);
  _edata = .;
} >RAM AT> FLASH
```

`>RAM AT> FLASH`: RAM에 배치하지만 초기값은 Flash에 저장.

시작 시 startup 코드가 Flash에서 RAM으로 복사함.

---

## BSS 섹션

```ld
.bss :
{
  . = ALIGN(4);
  _sbss = .;
  *(.bss)
  *(.bss*)
  *(COMMON)
  . = ALIGN(4);
  _ebss = .;
} >RAM
```

초기화 안 된 전역 변수. startup에서 0으로 채움.

---

## 스택/힙

```ld
_Min_Heap_Size = 0x200;
_Min_Stack_Size = 0x400;

.heap :
{
  . = ALIGN(8);
  PROVIDE ( end = . );
  . = . + _Min_Heap_Size;
} >RAM

.stack :
{
  . = ALIGN(8);
  . = . + _Min_Stack_Size;
  _estack = .;
} >RAM
```

부트로더는 힙 거의 안 쓰니까 작게 잡아도 됨.

---

## 확인

빌드 후 `.map` 파일:

```
Memory Configuration

Name             Origin             Length
RAM              0x20000000         0x00010000
FLASH            0x08000000         0x00004000

.isr_vector     0x08000000       0x10c
.text           0x0800010c      0x1a34
```

---

다음 글에서 앱 유효성 검사.

[#5 - 앱 유효성 검사](/posts/stm32-bootloader/stm32-bootloader-05/)
