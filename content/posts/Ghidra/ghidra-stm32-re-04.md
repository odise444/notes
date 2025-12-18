---
title: "Ghidra STM32 역분석 #4 - 디스어셈블리 vs 디컴파일"
date: 2024-12-10
draft: false
tags: ["Ghidra", "STM32", "리버스엔지니어링", "역분석", "ARM", "어셈블리"]
categories: ["역분석"]
series: ["Ghidra 역분석"]
summary: "어셈블리를 봐야 하나, C 코드를 봐야 하나? 둘 다 필요하다."
---

Ghidra 화면 왼쪽엔 어셈블리, 오른쪽엔 C 코드. 뭘 봐야 하나?

---

## 디스어셈블리 (Listing)

바이너리를 어셈블리 명령어로 변환한 것.

```asm
08002800 10 b5           push       { r4, lr }
08002802 04 4c           ldr        r4, [DAT_08002814]
08002804 20 46           mov        r0, r4
08002806 00 f0 4d f8     bl         FUN_080028a4
08002810 10 bd           pop        { r4, pc }
```

장점: 정확하다. CPU가 실행하는 그대로.
단점: 읽기 어렵다. C보다 10배 길다.

---

## 디컴파일 (Decompile)

어셈블리를 C 유사 코드로 변환한 것. Ghidra의 킬러 기능.

```c
void FUN_08002800(void)
{
  FUN_080028a4(DAT_08002814);
  FUN_080028f4(DAT_08002814);
  return;
}
```

장점: 읽기 쉽다.
단점: 100% 정확하지 않다. 최적화된 코드는 이상하게 나온다.

---

## 뭘 써야 하나

**시작은 디컴파일**, 이상하면 디스어셈블리 확인.

디컴파일 결과가 이상할 때:
- 변수 타입이 틀림
- 반복문 구조가 깨짐
- 조건문이 뒤집힘

그럴 때 어셈블리 보면 진짜 동작이 보인다.

---

## ARM 기본 명령어

자주 보이는 것들:

```asm
push { r4, lr }    ; 레지스터 저장
pop  { r4, pc }    ; 복원 후 리턴
bl   FUN_xxx       ; 함수 호출
ldr  r0, [r1]      ; 메모리 읽기
str  r0, [r1]      ; 메모리 쓰기
mov  r0, r1        ; 복사
cmp  r0, #0        ; 비교
beq  label         ; 같으면 점프
```

---

## 실전 예시

디컴파일:

```c
if (*(int *)0x40006400 == 0) {
    // ...
}
```

뭔지 모르겠다. 어셈블리 보면:

```asm
ldr  r0, =0x40006400    ; CAN1 base address
ldr  r1, [r0, #0x0]     ; CAN_MCR 읽기
cmp  r1, #0
beq  somewhere
```

`0x40006400`이 CAN1 베이스 주소라는 걸 알면 이해된다.

---

다음 글에서 Vector Table 분석.

[#5 - Vector Table 분석](/posts/ghidra/ghidra-stm32-re-05/)
