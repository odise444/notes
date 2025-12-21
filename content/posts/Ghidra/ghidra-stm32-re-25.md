---
title: "Ghidra STM32 역분석 #25 - 인라인 함수 vs 매크로"
date: 2024-12-14
draft: false
tags: ["Ghidra", "STM32", "리버스엔지니어링", "역분석", "최적화"]
categories: ["역분석"]
series: ["Ghidra 역분석"]
summary: "같은 코드가 여기저기 복붙된 것 같은데?"
---

분석하다 보니 똑같은 코드 패턴이 여러 곳에 있었다.

```c
// 함수 A에서
*(uint32_t *)0x40021000 |= 0x01000000;
while ((*(uint32_t *)0x40021000 & 0x02000000) == 0);

// 함수 B에서
*(uint32_t *)0x40021000 |= 0x01000000;
while ((*(uint32_t *)0x40021000 & 0x02000000) == 0);

// 함수 C에서도...
```

복붙? 인라인? 매크로?

---

## 인라인 함수

컴파일러가 함수 호출 대신 코드를 삽입한다.

```c
static inline void HSE_Enable(void) {
    RCC->CR |= RCC_CR_HSEON;
    while (!(RCC->CR & RCC_CR_HSERDY));
}
```

`-O2` 옵션이면 `bl HSE_Enable` 대신 코드가 직접 들어간다.

---

## 매크로

전처리기가 텍스트 치환한다.

```c
#define HSE_ENABLE() do { \
    RCC->CR |= RCC_CR_HSEON; \
    while (!(RCC->CR & RCC_CR_HSERDY)); \
} while(0)
```

결과는 인라인이랑 똑같다.

---

## 구분할 수 있나?

바이너리만 보면 구분 못한다. 둘 다 같은 기계어.

힌트가 될 수 있는 것:
- 디버그 심볼 있으면 인라인 함수명 나옴
- 매크로는 행 번호가 호출 위치로 나옴

근데 릴리즈 빌드면 둘 다 없다.

---

## 분석할 때 어떻게?

그냥 "공통 패턴"으로 인식하고 라벨 달아두면 된다.

```
// Ghidra 주석
// Pattern: HSE Enable sequence
*(uint32_t *)0x40021000 |= 0x01000000;
while ((*(uint32_t *)0x40021000 & 0x02000000) == 0);
```

원본이 인라인이든 매크로든 상관없다. 의미만 파악하면 됨.

---

## HAL 라이브러리 패턴

ST HAL 쓰면 이런 패턴 많다:

```c
__HAL_RCC_GPIOA_CLK_ENABLE();
__HAL_RCC_CAN1_CLK_ENABLE();
```

매크로라서 전부 인라인된다. 비슷한 코드가 반복되면 HAL 매크로일 가능성 높음.

---

다음 글에서 컴파일러 최적화 패턴.

[#26 - 컴파일러 최적화 패턴](/posts/ghidra/ghidra-stm32-re-26/)
