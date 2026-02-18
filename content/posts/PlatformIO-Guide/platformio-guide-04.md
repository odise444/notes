---
title: "PlatformIO 입문 #4 - 프로젝트 생성"
date: 2026-01-04
tags:
  - PlatformIO
  - 프로젝트
  - 보드
categories:
  - 가이드
series:
  - PlatformIO 입문
summary: 드디어 첫 프로젝트. 보드 선택이 제일 중요하다.
---

설치 끝났으면 첫 프로젝트 만들어보자.

---

## 프로젝트 생성하기

### 방법 1: PIO Home에서

1. PlatformIO 아이콘 클릭 (외계인 머리)
2. **PIO Home** → **Open**
3. **+ New Project** 클릭

### 방법 2: 명령 팔레트에서

1. `Ctrl + Shift + P`
2. `PlatformIO: New Project` 검색
3. 엔터

---

## 프로젝트 설정

팝업 창이 뜬다:

### Name (프로젝트 이름)

```
blink_led
```

영문, 숫자, 밑줄만. 한글 안 됨. 공백 안 됨.

### Board (보드 선택)

**가장 중요!**

검색창에 보드 이름 입력:

| 가지고 있는 보드 | 검색어 |
|-----------------|--------|
| Arduino Uno | `uno` |
| Arduino Nano | `nano` |
| Arduino Mega | `mega` |
| ESP32 DevKit | `esp32dev` |
| ESP8266 NodeMCU | `nodemcuv2` |
| STM32 Bluepill | `bluepill` |
| Raspberry Pi Pico | `pico` |

예: `esp32` 검색하면 ESP32 관련 보드 목록 나옴.

**Espressif ESP32 Dev Module** 선택.

### Framework (프레임워크)

보드에 따라 선택지 다름:

| 보드 | 프레임워크 옵션 |
|-----|----------------|
| Arduino 계열 | Arduino |
| ESP32 | Arduino, ESP-IDF |
| ESP8266 | Arduino, ESP8266 RTOS SDK |
| STM32 | Arduino, STM32Cube, Zephyr |
| Pico | Arduino, Pico SDK |

**초보자는 Arduino 선택.** 익숙한 문법 그대로 사용 가능.

### Location (저장 위치)

기본값 사용하거나, 원하는 폴더 지정.

기본: `C:\Users\사용자\Documents\PlatformIO\Projects\`

**Use default location** 체크하면 위 경로에 생성.

---

## 생성 버튼

**Finish** 클릭.

### 처음 생성 시 시간 걸림

해당 플랫폼 처음이면:

```
- 플랫폼 다운로드 (espressif32, atmelavr 등)
- 툴체인 다운로드 (컴파일러)
- 프레임워크 다운로드 (Arduino Core)
```

**5~10분 걸릴 수 있음.**

오른쪽 아래 상태바 확인:

```
PlatformIO: Installing platform espressif32...
```

두 번째부터는 금방 됨.

---

## 프로젝트 열림

완료되면 자동으로 프로젝트 폴더 열림.

왼쪽 탐색기에 파일 구조 보임:

```
blink_led/
├── .pio/
├── .vscode/
├── include/
├── lib/
├── src/
│   └── main.cpp
├── test/
└── platformio.ini
```

---

## main.cpp 열기

`src/main.cpp` 더블클릭.

기본 템플릿:

```cpp
#include <Arduino.h>

void setup() {
  // put your setup code here, to run once:
}

void loop() {
  // put your main code here, to run repeatedly:
}
```

Arduino 문법 그대로!

단, **`#include <Arduino.h>`가 필수**. Arduino IDE에선 자동 추가됐는데 여기선 직접 써야 함.

---

## 첫 코드 작성

LED 깜빡이기:

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

---

## 빌드해보기

하단 상태바에 버튼들 있음:

```
🏠  ✓  →  🔌  📺  ...
```

| 아이콘 | 기능 |
|--------|------|
| 🏠 | PIO Home |
| ✓ (체크) | Build (컴파일) |
| → (화살표) | Upload (업로드) |
| 🔌 | Serial Monitor |
| 📺 | Terminal |

**✓ (Build)** 클릭.

### 빌드 결과

터미널에 출력:

```
Processing esp32dev (platform: espressif32; board: esp32dev; framework: arduino)
...
Building...
...
RAM:   [=         ]  5.0% (used 16384 bytes from 327680 bytes)
Flash: [==        ] 15.1% (used 197632 bytes from 1310720 bytes)
========================= [SUCCESS] Took 5.42 seconds =========================
```

**SUCCESS** 나오면 성공!

---

## 에러 났을 때

### 에러 예시 1: Arduino.h 없음

```
fatal error: Arduino.h: No such file or directory
```

원인: `#include <Arduino.h>` 빼먹음.

### 에러 예시 2: LED_BUILTIN 없음

```
error: 'LED_BUILTIN' was not declared in this scope
```

원인: 해당 보드에 LED_BUILTIN 정의 안 됨.

해결: 직접 핀 번호 지정.

```cpp
#define LED_PIN 2  // ESP32는 보통 GPIO 2
pinMode(LED_PIN, OUTPUT);
```

---

## 다음 단계

프로젝트 생성 완료!

다음 글에서 프로젝트 구조 자세히 살펴보기.

---

[#5 - 프로젝트 구조](/posts/platformio-guide/platformio-guide-05/)
