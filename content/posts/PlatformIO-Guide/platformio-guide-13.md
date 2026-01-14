---
title: "PlatformIO 입문 #13 - 유닛 테스트"
date: 2024-12-22
tags: ["PlatformIO", "유닛테스트", "Unity", "TDD"]
categories: ["가이드"]
series: ["PlatformIO 입문"]
summary: "임베디드에서도 테스트 코드 짤 수 있다."
---

"임베디드는 테스트 못 해"

틀렸다. PlatformIO에서 유닛 테스트 지원한다.

---

## 왜 테스트?

```cpp
int calculateCRC(uint8_t* data, int len) {
  // 복잡한 로직
}
```

이 함수가 맞게 동작하는지 어떻게 확인?

1. 보드에 올려서 확인? → 느림, 귀찮음
2. 눈으로 코드 검토? → 실수 많음
3. **테스트 코드 작성** → 자동화, 확실

---

## Unity 테스트 프레임워크

PlatformIO 기본 테스트 프레임워크: **Unity**.

C용 경량 테스트 프레임워크.

---

## 프로젝트 구조

```
my_project/
├── src/
│   └── main.cpp
├── lib/
│   └── Calculator/
│       ├── Calculator.h
│       └── Calculator.cpp
├── test/
│   └── test_calculator/
│       └── test_main.cpp      ← 테스트 코드
└── platformio.ini
```

---

## 테스트 대상 코드

```cpp
// lib/Calculator/Calculator.h
#pragma once

class Calculator {
public:
  int add(int a, int b);
  int subtract(int a, int b);
  int multiply(int a, int b);
  int divide(int a, int b);
};
```

```cpp
// lib/Calculator/Calculator.cpp
#include "Calculator.h"

int Calculator::add(int a, int b) {
  return a + b;
}

int Calculator::subtract(int a, int b) {
  return a - b;
}

int Calculator::multiply(int a, int b) {
  return a * b;
}

int Calculator::divide(int a, int b) {
  if (b == 0) return 0;  // 간단한 처리
  return a / b;
}
```

---

## 테스트 코드 작성

```cpp
// test/test_calculator/test_main.cpp
#include <unity.h>
#include <Calculator.h>

Calculator calc;

void setUp(void) {
  // 각 테스트 전 실행
}

void tearDown(void) {
  // 각 테스트 후 실행
}

// 테스트 함수들
void test_add_positive_numbers(void) {
  TEST_ASSERT_EQUAL(5, calc.add(2, 3));
}

void test_add_negative_numbers(void) {
  TEST_ASSERT_EQUAL(-5, calc.add(-2, -3));
}

void test_add_mixed_numbers(void) {
  TEST_ASSERT_EQUAL(1, calc.add(-2, 3));
}

void test_subtract(void) {
  TEST_ASSERT_EQUAL(2, calc.subtract(5, 3));
}

void test_multiply(void) {
  TEST_ASSERT_EQUAL(15, calc.multiply(3, 5));
}

void test_divide(void) {
  TEST_ASSERT_EQUAL(3, calc.divide(15, 5));
}

void test_divide_by_zero(void) {
  TEST_ASSERT_EQUAL(0, calc.divide(10, 0));
}

int main(int argc, char **argv) {
  UNITY_BEGIN();
  
  RUN_TEST(test_add_positive_numbers);
  RUN_TEST(test_add_negative_numbers);
  RUN_TEST(test_add_mixed_numbers);
  RUN_TEST(test_subtract);
  RUN_TEST(test_multiply);
  RUN_TEST(test_divide);
  RUN_TEST(test_divide_by_zero);
  
  UNITY_END();
}
```

---

## 테스트 실행

### 네이티브 (PC에서)

```ini
; platformio.ini
[env:native]
platform = native
```

```bash
pio test -e native
```

보드 없이 PC에서 테스트!

### 보드에서

```bash
pio test -e esp32
```

보드에 업로드해서 실행, 결과를 시리얼로 받음.

---

## 테스트 결과

```
test/test_calculator/test_main.cpp:15: test_add_positive_numbers    [PASSED]
test/test_calculator/test_main.cpp:19: test_add_negative_numbers    [PASSED]
test/test_calculator/test_main.cpp:23: test_add_mixed_numbers       [PASSED]
test/test_calculator/test_main.cpp:27: test_subtract                [PASSED]
test/test_calculator/test_main.cpp:31: test_multiply                [PASSED]
test/test_calculator/test_main.cpp:35: test_divide                  [PASSED]
test/test_calculator/test_main.cpp:39: test_divide_by_zero          [PASSED]

-----------------------
7 Tests 0 Failures 0 Ignored
OK
```

---

## Unity 어설션

### 값 비교

```cpp
TEST_ASSERT_EQUAL(expected, actual);           // int
TEST_ASSERT_EQUAL_FLOAT(expected, actual);     // float
TEST_ASSERT_EQUAL_STRING("hello", str);        // 문자열
TEST_ASSERT_EQUAL_MEMORY(expected, actual, len); // 메모리
```

### 조건

```cpp
TEST_ASSERT_TRUE(condition);
TEST_ASSERT_FALSE(condition);
TEST_ASSERT_NULL(pointer);
TEST_ASSERT_NOT_NULL(pointer);
```

### 범위

```cpp
TEST_ASSERT_INT_WITHIN(delta, expected, actual);
TEST_ASSERT_FLOAT_WITHIN(delta, expected, actual);
```

---

## 여러 테스트 파일

```
test/
├── test_calculator/
│   └── test_main.cpp
├── test_parser/
│   └── test_main.cpp
└── test_protocol/
    └── test_main.cpp
```

각 폴더가 독립적인 테스트 세트.

```bash
pio test                              # 전부 실행
pio test -f test_calculator           # 특정 테스트만
```

---

## 모킹 (Mocking)

하드웨어 의존 코드 테스트할 때:

```cpp
// 실제 코드
int readSensor() {
  return analogRead(A0);
}

// 테스트용 목 (mock)
int mock_sensor_value = 0;

int readSensor() {
  return mock_sensor_value;
}

void test_sensor_high(void) {
  mock_sensor_value = 900;
  TEST_ASSERT_TRUE(isSensorHigh());
}
```

조건부 컴파일로 분리:

```cpp
#ifdef UNIT_TEST
  // 목 구현
#else
  // 실제 구현
#endif
```

---

## 테스트 설정

```ini
[env:native]
platform = native
test_build_src = true              ; src/ 폴더도 빌드
test_ignore = test_hardware/*      ; 하드웨어 테스트 제외

[env:esp32]
platform = espressif32
board = esp32dev
framework = arduino
test_port = COM3
test_speed = 115200
```

---

## TDD 워크플로우

1. **테스트 먼저 작성** (실패)
2. **코드 구현** (성공)
3. **리팩토링** (여전히 성공)

```bash
# 1. 테스트 작성
pio test -e native    # FAIL

# 2. 코드 구현
pio test -e native    # PASS

# 3. 리팩토링 후 확인
pio test -e native    # PASS
```

---

다음 글에서 CI/CD.

[#14 - CI/CD](/posts/platformio-guide/platformio-guide-14/)
