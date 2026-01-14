---
title: "PlatformIO 완벽 가이드 #7 - platformio.ini 설정"
date: 2024-12-22
tags: ["PlatformIO", "설정", "platformio.ini"]
categories: ["개발환경"]
series: ["PlatformIO 완벽 가이드"]
summary: "프로젝트의 모든 설정이 이 파일에."
---

`platformio.ini`가 프로젝트의 핵심이다.

보드, 라이브러리, 빌드 옵션 다 여기서 설정.

---

## 기본 구조

```ini
[env:uno]
platform = atmelavr
board = uno
framework = arduino
```

`[env:이름]`으로 환경 정의.

---

## 필수 항목

### platform

플랫폼 (툴체인).

```ini
platform = atmelavr      ; Arduino Uno, Mega
platform = espressif32   ; ESP32
platform = ststm32       ; STM32
platform = raspberrypi   ; Raspberry Pi Pico
```

### board

보드 이름.

```ini
board = uno              ; Arduino Uno
board = esp32dev         ; ESP32 DevKit
board = bluepill_f103c8  ; STM32 Blue Pill
board = pico             ; Raspberry Pi Pico
```

PIO Home → Boards에서 검색.

### framework

프레임워크.

```ini
framework = arduino      ; Arduino 프레임워크
framework = espidf       ; ESP-IDF (ESP32 네이티브)
framework = stm32cube    ; STM32Cube HAL
framework = zephyr       ; Zephyr RTOS
```

Arduino로 시작하는 게 쉬움.

---

## 시리얼 모니터 설정

```ini
[env:uno]
platform = atmelavr
board = uno
framework = arduino

monitor_speed = 115200   ; 보드레이트
```

`Serial.begin(115200)`과 맞춰야 함.

---

## 라이브러리 의존성

```ini
lib_deps =
    adafruit/DHT sensor library@^1.4.4
    adafruit/Adafruit Unified Sensor@^1.1.9
```

- 라이브러리 이름
- `@버전` 으로 버전 지정
- `^1.4.4`는 1.4.4 이상 1.x.x 중 최신

---

## 라이브러리 찾는 법

### 방법 1: PIO Home

Libraries → Registry에서 검색.

라이브러리 페이지에 `lib_deps` 복사 버튼 있음.

### 방법 2: 검색

https://registry.platformio.org/

### 방법 3: CLI

```bash
pio pkg search "DHT"
```

---

## 업로드 포트 지정

자동 감지가 기본이지만, 직접 지정도 가능.

```ini
upload_port = COM3          ; Windows
upload_port = /dev/ttyUSB0  ; Linux
upload_port = /dev/cu.*     ; Mac (와일드카드)
```

여러 보드 연결했을 때 유용.

---

## 빌드 플래그

컴파일 옵션 추가.

```ini
build_flags =
    -D DEBUG_MODE           ; #define DEBUG_MODE
    -D VERSION="1.0.0"      ; #define VERSION "1.0.0"
    -O2                     ; 최적화 레벨
```

코드에서:

```cpp
#ifdef DEBUG_MODE
  Serial.println("Debug mode");
#endif
```

---

## 전체 예시

```ini
; ESP32 프로젝트 예시
[env:esp32]
platform = espressif32
board = esp32dev
framework = arduino

; 시리얼 모니터
monitor_speed = 115200

; 라이브러리
lib_deps =
    adafruit/DHT sensor library@^1.4.4
    bblanchon/ArduinoJson@^6.21.0

; 빌드 옵션
build_flags =
    -D DEBUG_MODE
    -D WIFI_SSID=\"MyWifi\"
    -D WIFI_PASS=\"MyPassword\"

; 업로드 속도
upload_speed = 921600
```

---

## 공통 설정

여러 환경에서 공유하는 설정:

```ini
[platformio]
default_envs = esp32    ; 기본 환경

[env]
; 모든 환경에 적용
monitor_speed = 115200
lib_deps =
    bblanchon/ArduinoJson@^6.21.0

[env:esp32]
platform = espressif32
board = esp32dev
framework = arduino

[env:uno]
platform = atmelavr
board = uno
framework = arduino
```

`[env]` 섹션은 모든 환경에 적용.

---

## 자주 쓰는 옵션

| 옵션 | 설명 | 예시 |
|------|------|------|
| `platform` | 플랫폼 | `espressif32` |
| `board` | 보드 | `esp32dev` |
| `framework` | 프레임워크 | `arduino` |
| `monitor_speed` | 시리얼 속도 | `115200` |
| `lib_deps` | 라이브러리 | (여러 줄) |
| `build_flags` | 컴파일 옵션 | `-D DEBUG` |
| `upload_port` | 업로드 포트 | `COM3` |
| `upload_speed` | 업로드 속도 | `921600` |

---

## 다음 단계

platformio.ini 파악 완료.

다음 글에서 빌드와 업로드.

---

다음 글: [#8 - 첫 빌드와 업로드](/posts/platformio/platformio-08/)
