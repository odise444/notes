---
title: "PlatformIO 입문 #6 - platformio.ini 해부"
date: 2026-01-06
tags:
  - PlatformIO
  - 설정
  - ini
categories:
  - 가이드
series:
  - PlatformIO 입문
summary: 이 파일 하나에 모든 설정이 들어있다.
---

`platformio.ini`는 프로젝트의 **모든 설정**을 담고 있다.

처음 보면 복잡해 보이는데, 하나씩 뜯어보면 간단하다.

---

## 기본 구조

```ini
; 주석은 세미콜론으로

[env:esp32dev]           ; 환경 이름
platform = espressif32   ; 플랫폼
board = esp32dev         ; 보드
framework = arduino      ; 프레임워크
```

이게 최소 설정.

---

## 섹션 이해하기

### [env:이름]

**환경(Environment)** 정의.

```ini
[env:esp32dev]
```

- `env:` 접두사 필수
- `esp32dev`는 내가 정한 이름
- 여러 환경 만들 수 있음

### platform

**플랫폼** = 칩 제조사/계열.

```ini
platform = espressif32    ; ESP32 계열
platform = atmelavr       ; AVR (Arduino Uno 등)
platform = ststm32        ; STM32 계열
platform = raspberrypi    ; Raspberry Pi Pico
```

### board

**정확한 보드 모델**.

```ini
board = esp32dev          ; ESP32 DevKit
board = uno               ; Arduino Uno
board = bluepill_f103c8   ; STM32 Bluepill
board = pico              ; Raspberry Pi Pico
```

보드마다 핀맵, 클럭, 플래시 크기 등이 다름.

### framework

**프레임워크** = 개발 방식.

```ini
framework = arduino       ; Arduino 스타일
framework = espidf        ; ESP-IDF (네이티브)
framework = stm32cube     ; STM32 HAL
framework = zephyr        ; Zephyr RTOS
```

초보자는 `arduino` 추천.

---

## 자주 쓰는 옵션들

### 시리얼 모니터 속도

```ini
monitor_speed = 115200
```

기본값: 9600. 요즘은 115200 많이 씀.

### 업로드 속도

```ini
upload_speed = 921600
```

ESP32는 높여도 됨. 업로드 빨라짐.

### 업로드 포트 지정

```ini
upload_port = COM3           ; Windows
upload_port = /dev/ttyUSB0   ; Linux
upload_port = /dev/cu.usbserial-*  ; macOS
```

자동 감지되니까 보통 안 써도 됨.

### 라이브러리 의존성

```ini
lib_deps = 
    adafruit/Adafruit SSD1306 @ ^2.5.0
    bblanchon/ArduinoJson @ ^6.21.0
```

**핵심 기능!** 다음 글에서 자세히.

---

## 빌드 플래그

### 매크로 정의

```ini
build_flags = 
    -D DEBUG_MODE
    -D VERSION=123
    -D BOARD_NAME=\"ESP32\"
```

코드에서:

```cpp
#ifdef DEBUG_MODE
  Serial.println("Debug mode!");
#endif
```

### 최적화 레벨

```ini
build_flags = -O2    ; 속도 최적화
build_flags = -Os    ; 크기 최적화 (기본값)
build_flags = -O0    ; 최적화 없음 (디버깅용)
```

### 경고 설정

```ini
build_flags = 
    -Wall            ; 모든 경고 켜기
    -Wextra          ; 추가 경고
    -Werror          ; 경고를 에러로 처리
```

---

## 멀티 환경

**하나의 코드로 여러 보드 지원**.

```ini
[env:esp32]
platform = espressif32
board = esp32dev
framework = arduino

[env:uno]
platform = atmelavr
board = uno
framework = arduino

[env:bluepill]
platform = ststm32
board = bluepill_f103c8
framework = arduino
```

빌드할 때 환경 선택:

- 상태바에서 클릭
- 또는 `pio run -e esp32`

---

## 공통 설정

`[env]` 섹션 (콜론 없음):

```ini
[env]
; 모든 환경에 공통 적용
framework = arduino
monitor_speed = 115200
lib_deps = 
    ArduinoJson

[env:esp32]
platform = espressif32
board = esp32dev

[env:uno]
platform = atmelavr
board = uno
```

`[env]`에 넣으면 모든 환경에 적용.

---

## 플랫폼 버전 고정

```ini
platform = espressif32 @ 6.4.0
```

특정 버전 고정. 팀 프로젝트에서 유용.

버전 안 쓰면 최신 버전 사용.

---

## 보드 설정 오버라이드

```ini
[env:esp32_custom]
platform = espressif32
board = esp32dev
framework = arduino

; 보드 기본값 덮어쓰기
board_build.f_cpu = 240000000L    ; 클럭 240MHz
board_build.flash_mode = qio       ; 플래시 모드
board_upload.flash_size = 4MB      ; 플래시 크기
```

---

## 실제 예시

### ESP32 프로젝트

```ini
[env:esp32dev]
platform = espressif32
board = esp32dev
framework = arduino

monitor_speed = 115200
upload_speed = 921600

build_flags = 
    -D CORE_DEBUG_LEVEL=0
    -D CONFIG_ASYNC_TCP_QUEUE_SIZE=128

lib_deps = 
    ESP Async WebServer
    ArduinoJson @ ^6.21.0
```

### Arduino Uno 프로젝트

```ini
[env:uno]
platform = atmelavr
board = uno
framework = arduino

monitor_speed = 9600

lib_deps = 
    Servo
```

### STM32 Bluepill 프로젝트

```ini
[env:bluepill]
platform = ststm32
board = bluepill_f103c8
framework = arduino

upload_protocol = stlink    ; ST-Link 사용
debug_tool = stlink         ; 디버깅도 ST-Link

monitor_speed = 115200
```

---

## 설정 찾기

모든 옵션 외우기 불가능.

### 방법 1: 공식 문서

https://docs.platformio.org/en/latest/projectconf/

### 방법 2: 보드 페이지

PIO Home → Boards → 보드 선택 → 설정 예시 나옴.

### 방법 3: 검색

`platformio esp32 upload speed` 검색.

---

다음 글에서 빌드와 업로드 자세히.

[#7 - 빌드와 업로드](/posts/platformio-guide/platformio-guide-07/)
