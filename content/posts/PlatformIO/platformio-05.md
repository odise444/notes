---
title: "PlatformIO 완벽 가이드 #5 - 새 프로젝트 만들기"
date: 2024-12-22
tags: ["PlatformIO", "프로젝트", "Arduino", "ESP32"]
categories: ["개발환경"]
series: ["PlatformIO 완벽 가이드"]
summary: "LED 깜빡이기 프로젝트 만들기."
---

드디어 프로젝트를 만든다.

클래식한 LED 깜빡이기(Blink)로 시작.

---

## 새 프로젝트 생성

1. PIO Home 열기 (🏠 클릭)
2. **New Project** 클릭

{{< figure src="/imgs/pio-new-project-wizard.png" caption="Project Wizard" >}}

---

## 프로젝트 설정

### Name

프로젝트 이름. 영문 추천.

```
Blink
```

### Board

사용할 보드 선택.

검색창에 입력:
- Arduino Uno: `uno`
- ESP32 DevKit: `esp32dev`
- STM32 Blue Pill: `bluepill_f103c8`

{{< figure src="/imgs/pio-board-select.png" caption="보드 선택" >}}

### Framework

Arduino Uno/ESP32: **Arduino** 선택
STM32: Arduino 또는 STM32Cube 선택

### Location

프로젝트 저장 위치.

기본값 사용하거나 원하는 폴더 지정.

---

## 프로젝트 생성

**Finish** 버튼 클릭.

처음엔 시간이 좀 걸린다.

```
PlatformIO: Initializing...
Installing platform...
Installing toolchain...
```

보드에 필요한 도구들 자동 설치.

{{< figure src="/imgs/pio-project-creating.png" caption="프로젝트 생성 중" >}}

---

## 생성 완료

왼쪽 탐색기에 프로젝트 폴더 구조가 보인다.

```
Blink/
├── .pio/
├── .vscode/
├── include/
├── lib/
├── src/
│   └── main.cpp      ← 코드 여기
├── test/
└── platformio.ini    ← 설정 파일
```

---

## main.cpp 열기

`src/main.cpp` 더블클릭.

```cpp
#include <Arduino.h>

void setup() {
  // put your setup code here, to run once:
}

void loop() {
  // put your loop code here, to run repeatedly:
}
```

Arduino IDE의 `.ino` 파일과 거의 같다.

차이점: `#include <Arduino.h>` 필요.

---

## LED 깜빡이기 코드

```cpp
#include <Arduino.h>

void setup() {
  pinMode(LED_BUILTIN, OUTPUT);
}

void loop() {
  digitalWrite(LED_BUILTIN, HIGH);
  delay(1000);
  digitalWrite(LED_BUILTIN, LOW);
  delay(1000);
}
```

Arduino IDE에서 쓰던 코드 그대로.

---

## IntelliSense 체험

`digital` 까지 치면:

{{< figure src="/imgs/pio-intellisense.png" caption="자동완성" >}}

- `digitalWrite`
- `digitalRead`
- `digitalPinToInterrupt`

자동완성 목록이 뜬다!

Tab 또는 Enter로 선택.

**이게 Arduino IDE에 없던 것.**

---

## 에러 표시

일부러 틀려보자:

```cpp
digitalWrite(LED_BUILTIN, HIHG);  // HIGH 오타
```

저장하면 바로 빨간 밑줄.

{{< figure src="/imgs/pio-error-highlight.png" caption="에러 하이라이트" >}}

마우스 올리면 에러 메시지:

```
'HIHG' was not declared in this scope
```

**컴파일하기 전에 에러 확인 가능.**

---

## 함수 정보

`digitalWrite`에 마우스 올리면:

{{< figure src="/imgs/pio-hover-info.png" caption="함수 정보" >}}

```cpp
void digitalWrite(uint8_t pin, uint8_t val)
```

파라미터 타입, 설명 바로 보임.

---

## 정의로 이동

`LED_BUILTIN`에서 `F12` (또는 Ctrl+클릭):

해당 상수가 정의된 파일로 점프.

```cpp
#define LED_BUILTIN 13
```

Arduino 보드 설정 파일 안에 있다.

---

## 다음 단계

프로젝트 생성 완료.

다음 글에서 폴더 구조 자세히 살펴보기.

---

다음 글: [#6 - 폴더 구조 이해하기](/posts/platformio/platformio-06/)
