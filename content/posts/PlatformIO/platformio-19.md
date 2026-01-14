---
title: "PlatformIO 완벽 가이드 #19 - 유닛 테스트"
date: 2024-12-22
tags: ["PlatformIO", "테스트", "Unity", "TDD"]
categories: ["개발환경"]
series: ["PlatformIO 완벽 가이드"]
summary: "코드가 제대로 동작하는지 자동으로 확인."
---

"이 함수 바꿨는데 다른 데 영향 없겠지?"

유닛 테스트로 확인하자.

---

## PlatformIO 테스트 프레임워크

PlatformIO에 **Unity** 테스트 프레임워크 내장.

C/C++ 임베디드용 경량 테스트 프레임워크.

---

## 테스트 폴더 구조

```
MyProject/
├── lib/
├── src/
│   └── main.cpp
├── test/                    ← 테스트 코드
│   └── test_main.cpp
└── platformio.ini
```

`test/` 폴더에 테스트 코드 작성.

---

## 첫 번째 테스트

`test/test_main.cpp`:

```cpp
#include <unity.h>

void test_addition(void) {
    TEST_ASSERT_EQUAL(4, 2 + 2);
}

void test_string(void) {
    TEST_ASSERT_EQUAL_STRING("hello", "hello");
}

void setup() {
    UNITY_BEGIN();
    RUN_TEST(test_addition);
    RUN_TEST(test_string);
    UNITY_END();
}

void loop() {
    // 빈 루프
}
```

---

## 테스트 실행

### 방법 1: 아이콘

하단 상태바에서 🧪 (테스트) 클릭.

### 방법 2: 명령 팔레트

`Ctrl + Shift + P` → "PlatformIO: Test"

### 방법 3: CLI

```bash
pio test
```

---

## 테스트 결과

```
test/test_main.cpp:5: test_addition  [PASSED]
test/test_main.cpp:9: test_string    [PASSED]
-----------------------
2 Tests 0 Failures 0 Ignored
```

모든 테스트 통과!

---

## Unity 어서션

### 기본 비교

```cpp
TEST_ASSERT_EQUAL(expected, actual);           // int
TEST_ASSERT_EQUAL_STRING("exp", actual);       // 문자열
TEST_ASSERT_EQUAL_FLOAT(1.5, actual, 0.01);    // float (오차 허용)
TEST_ASSERT_EQUAL_MEMORY(exp, act, size);      // 메모리
```

### 참/거짓

```cpp
TEST_ASSERT_TRUE(condition);
TEST_ASSERT_FALSE(condition);
TEST_ASSERT_NULL(pointer);
TEST_ASSERT_NOT_NULL(pointer);
```

### 배열

```cpp
int expected[] = {1, 2, 3};
int actual[] = {1, 2, 3};
TEST_ASSERT_EQUAL_INT_ARRAY(expected, actual, 3);
```

---

## 실제 코드 테스트

테스트할 코드:

`lib/Calculator/Calculator.h`:
```cpp
#ifndef CALCULATOR_H
#define CALCULATOR_H

class Calculator {
public:
    int add(int a, int b);
    int subtract(int a, int b);
    int multiply(int a, int b);
    float divide(int a, int b);
};

#endif
```

`lib/Calculator/Calculator.cpp`:
```cpp
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

float Calculator::divide(int a, int b) {
    if (b == 0) return 0;
    return (float)a / b;
}
```

---

## 테스트 코드

`test/test_calculator.cpp`:

```cpp
#include <unity.h>
#include <Calculator.h>

Calculator calc;

void setUp(void) {
    // 각 테스트 전 실행
}

void tearDown(void) {
    // 각 테스트 후 실행
}

void test_add(void) {
    TEST_ASSERT_EQUAL(5, calc.add(2, 3));
    TEST_ASSERT_EQUAL(0, calc.add(-1, 1));
    TEST_ASSERT_EQUAL(-5, calc.add(-2, -3));
}

void test_subtract(void) {
    TEST_ASSERT_EQUAL(1, calc.subtract(3, 2));
    TEST_ASSERT_EQUAL(-2, calc.subtract(-1, 1));
}

void test_multiply(void) {
    TEST_ASSERT_EQUAL(6, calc.multiply(2, 3));
    TEST_ASSERT_EQUAL(0, calc.multiply(0, 100));
}

void test_divide(void) {
    TEST_ASSERT_EQUAL_FLOAT(2.0, calc.divide(6, 3), 0.01);
    TEST_ASSERT_EQUAL_FLOAT(0.0, calc.divide(5, 0), 0.01);  // 0으로 나누기
}

void setup() {
    UNITY_BEGIN();
    RUN_TEST(test_add);
    RUN_TEST(test_subtract);
    RUN_TEST(test_multiply);
    RUN_TEST(test_divide);
    UNITY_END();
}

void loop() {}
```

---

## 네이티브 테스트

보드 없이 PC에서 테스트:

`platformio.ini`:

```ini
[env:native]
platform = native
test_framework = unity
```

```bash
pio test -e native
```

하드웨어 독립적인 로직은 PC에서 빠르게 테스트.

---

## 테스트 필터

특정 테스트만 실행:

```bash
pio test -f test_calculator   # 파일명으로 필터
pio test -i test_add          # 테스트 이름으로 필터
```

---

## test_ignore

특정 테스트 제외:

```ini
test_ignore = test_hardware
```

하드웨어 의존 테스트 제외.

---

## 다음 단계

유닛 테스트 완료!

다음 글에서 CI/CD 연동.

---

다음 글: [#20 - CI/CD 연동](/posts/platformio/platformio-20/)
