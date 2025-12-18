---
title: "Ghidra STM32 역분석 #5 - Vector Table 분석"
date: 2024-12-10
draft: false
tags: ["Ghidra", "STM32", "리버스엔지니어링", "역분석", "ARM", "Vector Table"]
categories: ["역분석"]
series: ["Ghidra 역분석"]
summary: "STM32 펌웨어의 출발점. Vector Table을 읽으면 전체 구조가 보인다."
---

ARM Cortex-M이 부팅할 때 가장 먼저 읽는 테이블.

```
0x08000000: Initial Stack Pointer
0x08000004: Reset_Handler 주소
0x08000008: NMI_Handler
0x0800000C: HardFault_Handler
```

CPU 전원이 들어오면 `0x08000000`에서 SP 값 로드하고, `0x08000004`에서 Reset_Handler 주소 읽어서 점프. 펌웨어 시작.

---

## 200429.hex Vector Table

Ghidra에서 `0x08000000`으로 이동:

```
08000000 18 52 00 20     addr       DAT_20005218      ; SP
08000004 01 28 00 08     addr       FUN_08002800+1    ; Reset
08000008 1f 2b 00 08     addr       FUN_08002b1e+1    ; NMI
0800000c ab 20 00 08     addr       FUN_080020aa+1    ; HardFault
```

---

## 리틀 엔디안

바이트 순서가 뒤집혀 있다:

```
08000000: 18 52 00 20 → 0x20005218 (뒤에서부터)
```

---

## Thumb 비트

Reset_Handler 주소가 `0x08002801`인데 실제 주소는 `0x08002800`.

마지막 비트 1은 Thumb 모드 표시. ARM Cortex-M은 Thumb 모드만 쓴다.

---

## 핵심 발견: CAN 인터럽트

외부 인터럽트는 `0x08000040`부터:

```
08000090 dd 22 00 08     addr       FUN_080022dc+1    ; CAN1_RX0
```

IAP 부트로더니까 CAN 수신 핸들러가 핵심이다. `0x080022DC`로 가보면:

```c
void CAN1_RX0_IRQHandler(void)
{
  // FIFO에서 메시지 읽기
  // IAP 프로토콜 처리
}
```

---

## Vector Table로 알 수 있는 것

1. **사용 중인 인터럽트**: 0이 아닌 핸들러
2. **코드 영역 범위**: 모든 핸들러가 `0x08002xxx`~`0x08003xxx`
3. **개발 수준**: 간단한 에러 처리만 구현

---

## 라벨 지정

분석 편의를 위해 이름 지정 (L 키):

| 주소 | 변경 후 |
|------|---------|
| 0x08002800 | Reset_Handler |
| 0x080022DC | CAN1_RX0_IRQHandler |

---

다음 글에서 메모리 맵 추정.

[#6 - 메모리 맵 추정](/posts/ghidra/ghidra-stm32-re-06/)
