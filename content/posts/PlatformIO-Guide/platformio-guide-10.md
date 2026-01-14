---
title: "PlatformIO 입문 #10 - 멀티 환경 설정"
date: 2026-01-10
tags:
  - PlatformIO
  - 멀티환경
  - 크로스플랫폼
categories:
  - 가이드
series:
  - PlatformIO 입문
summary: 하나의 코드로 ESP32, Arduino, STM32 다 돌린다.
---

같은 코드를 여러 보드에서 돌리고 싶을 때.

PlatformIO의 **멀티 환경** 기능이 빛난다.

---

## 왜 필요한가

센서 읽어서 시리얼로 출력하는 코드.

- 개발: ESP32 (WiFi 디버깅 편함)
- 양산: STM32 (가격 저렴)
- 테스트: Arduino Uno (간편)

코드는 같은데 보드만 다름.

매번 프로젝트 새로 만들기? 귀찮다.

---

## 멀티 환경 설정

```ini
; 공통 설정
[env]
framework = arduino
monitor_speed = 115200

; ESP32 환경
[env:esp32]
platform = espressif32
board = esp32dev

; Arduino Uno 환경
[env:uno]
platform = atmelavr
board = uno

; STM32 Bluepill 환경
[env:bluepill]
platform = ststm32
board = bluepill_f103c8
upload_protocol = stlink
```

---

## 환경 선택하기

### 방법 1: 상태바

VSCode 하단 상태바:

```
env:esp32 (blink_led)
```

클릭하면 환경 목록 나옴. 원하는 거 선택.

### 방법 2: 명령 팔레트

`Ctrl + Shift + P` → `PlatformIO: Switch Project Environment`

### 방법 3: 터미널

```bash
pio run -e esp32      # esp32 환경으로 빌드
pio run -e uno        # uno 환경으로 빌드
pio run -e bluepill   # bluepill 환경으로 빌드
```

---

## 환경별 설정 분리

```ini
[env]
framework = arduino
monitor_speed = 115200

[env:esp32]
platform = espressif32
board = esp32dev
build_flags = -D ESP32_BOARD

[env:uno]
platform = atmelavr
board = uno
build_flags = -D ARDUINO_BOARD

[env:bluepill]
platform = ststm32
board = bluepill_f103c8
build_flags = -D STM32_BOARD
upload_protocol = stlink
```

---

## 코드에서 보드 구분

```cpp
#include <Arduino.h>

void setup() {
  Serial.begin(115200);
  
  #ifdef ESP32_BOARD
    Serial.println("Running on ESP32");
    // ESP32 전용 코드
  #endif
  
  #ifdef ARDUINO_BOARD
    Serial.println("Running on Arduino");
    // Arduino 전용 코드
  #endif
  
  #ifdef STM32_BOARD
    Serial.println("Running on STM32");
    // STM32 전용 코드
  #endif
}

void loop() {
  // 공통 코드
}
```

---

## 핀 정의 분리

```cpp
// config.h
#pragma once

#ifdef ESP32_BOARD
  #define LED_PIN 2
  #define BUTTON_PIN 0
  #define ADC_PIN 34
#endif

#ifdef ARDUINO_BOARD
  #define LED_PIN 13
  #define BUTTON_PIN 2
  #define ADC_PIN A0
#endif

#ifdef STM32_BOARD
  #define LED_PIN PC13
  #define BUTTON_PIN PA0
  #define ADC_PIN PA1
#endif
```

```cpp
// main.cpp
#include <Arduino.h>
#include "config.h"

void setup() {
  pinMode(LED_PIN, OUTPUT);
  pinMode(BUTTON_PIN, INPUT_PULLUP);
}
```

---

## 환경별 라이브러리

```ini
[env]
framework = arduino
lib_deps = 
    ArduinoJson          ; 공통

[env:esp32]
platform = espressif32
board = esp32dev
lib_deps = 
    ${env.lib_deps}      ; 공통 라이브러리 상속
    ESP Async WebServer   ; ESP32 전용

[env:uno]
platform = atmelavr
board = uno
; lib_deps 없음 = 공통만 사용

[env:bluepill]
platform = ststm32
board = bluepill_f103c8
lib_deps = 
    ${env.lib_deps}
    STM32duino FreeRTOS   ; STM32 전용
```

---

## 기본 환경 지정

```ini
[platformio]
default_envs = esp32     ; 기본값
```

환경 선택 안 하면 이걸로 빌드.

---

## 모든 환경 한 번에 빌드

```bash
pio run
```

모든 환경 순차 빌드.

```
Processing esp32 (platform: espressif32; board: esp32dev)
...
[SUCCESS]

Processing uno (platform: atmelavr; board: uno)
...
[SUCCESS]

Processing bluepill (platform: ststm32; board: bluepill_f103c8)
...
[SUCCESS]
```

CI/CD에서 유용.

---

## 실제 예시: 온도 센서

```ini
[platformio]
default_envs = esp32

[env]
framework = arduino
monitor_speed = 115200
lib_deps = 
    adafruit/DHT sensor library

[env:esp32]
platform = espressif32
board = esp32dev
build_flags = 
    -D DHT_PIN=4
    -D LED_PIN=2

[env:uno]
platform = atmelavr
board = uno
build_flags = 
    -D DHT_PIN=2
    -D LED_PIN=13

[env:bluepill]
platform = ststm32
board = bluepill_f103c8
upload_protocol = stlink
build_flags = 
    -D DHT_PIN=PA0
    -D LED_PIN=PC13
```

```cpp
#include <Arduino.h>
#include <DHT.h>

DHT dht(DHT_PIN, DHT22);

void setup() {
  Serial.begin(115200);
  pinMode(LED_PIN, OUTPUT);
  dht.begin();
}

void loop() {
  float temp = dht.readTemperature();
  Serial.print("Temperature: ");
  Serial.println(temp);
  
  digitalWrite(LED_PIN, !digitalRead(LED_PIN));
  delay(2000);
}
```

코드는 하나, 보드는 세 개!

---

다음 글에서 빌드 플래그 자세히.

[#11 - 빌드 플래그](/posts/platformio-guide/platformio-guide-11/)
