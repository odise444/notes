---
title: "Ghidra STM32 역분석 번외 #4 - 난독화 분석"
date: 2024-12-14
draft: false
tags: ["Ghidra", "STM32", "리버스엔지니어링", "역분석", "난독화"]
categories: ["역분석"]
series: ["Ghidra 역분석"]
summary: "난독화된 펌웨어는 어떻게 분석할까?"
---

이번 부트로더는 난독화가 없었다. 근데 가끔 난독화된 펌웨어도 만난다.

---

## 흔한 난독화 기법

**1. 문자열 암호화**

```c
// 평문 대신
char *msg = decrypt("x#$@!*&");
```

**2. 제어 흐름 평탄화**

```c
// switch로 감싸서 흐름 숨기기
while (1) {
    switch (state) {
        case 0: step1(); state = 3; break;
        case 1: step4(); state = 2; break;
        case 2: step2(); state = 1; break;
        case 3: step3(); state = 4; break;
        ...
    }
}
```

**3. 가짜 코드 삽입**

```c
if (always_false) {
    // 절대 실행 안 되는 코드
    do_nothing();
}
```

**4. 간접 호출**

```c
// 직접 호출 대신
void (*func)(void) = get_func_ptr(0x1234);
func();
```

---

## 대응 방법

**문자열 암호화:**
- 복호화 함수 찾기
- 동적 분석 (에뮬레이터)
- 복호화 함수 스크립트로 실행

**제어 흐름 평탄화:**
- state 변수 추적
- 실제 실행 순서 재구성
- 스크립트로 자동화

**가짜 코드:**
- 조건문이 상수인지 확인
- 데드 코드 제거

**간접 호출:**
- 함수 포인터 테이블 찾기
- 동적 분석으로 실제 호출 확인

---

## 동적 분석

정적 분석만으론 한계. 실제로 실행해보자.

**QEMU:**
```bash
qemu-arm -cpu cortex-m3 firmware.elf
```

**Unicorn Engine:**
```python
from unicorn import *
from unicorn.arm_const import *

mu = Uc(UC_ARCH_ARM, UC_MODE_THUMB)
mu.mem_map(0x08000000, 0x80000)
mu.mem_write(0x08000000, firmware_data)
mu.emu_start(0x08000000, 0x08000100)
```

---

## 실전 팁

1. 먼저 정적 분석으로 전체 구조 파악
2. 난독화된 부분 식별
3. 복호화/디코딩 함수 찾기
4. 스크립트나 에뮬레이터로 실행
5. 결과로 정적 분석 보완

---

## 포기할 때

- 하드웨어 보안 (Secure Boot, TrustZone)
- 전체 암호화 펌웨어
- 디버그 포트 완전 비활성화

이런 경우는 다른 방법이 필요. 하드웨어 공격이나 사이드 채널 분석.

---

## 시리즈 완료

본편 34개 + 번외 4개 = 총 38개

역분석 입문부터 실전 활용까지 다뤘다. 

---

처음 글로 돌아가기: [#1 - 왜 역분석을 하게 됐나](/posts/ghidra/ghidra-stm32-re-01/)
