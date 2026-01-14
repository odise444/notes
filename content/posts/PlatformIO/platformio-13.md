---
title: "PlatformIO 완벽 가이드 #13 - 여러 보드 동시 설정"
date: 2024-12-22
tags: ["PlatformIO", "멀티보드", "환경설정"]
categories: ["개발환경"]
series: ["PlatformIO 완벽 가이드"]
summary: "Arduino Uno랑 ESP32 동시에 개발하기."
---

프로젝트 하나에 보드 여러 개.

Arduino Uno로 프로토타입, ESP32로 실제 배포.

---

## 멀티 환경 설정

`platformio.ini`:

```ini
[env:uno]
platform = atmelavr
board = uno
framework = arduino

[env:esp32]
platform = espressif32
board = esp32dev
framework = arduino
```

`[env:이름]`으로 환경 구분.

---

## 환경 선택

### 방법 1: 하단 상태바

하단에 현재 환경 표시.

{{< figure src="/imgs/pio-env-select.png" caption="환경 선택" >}}

클릭하면 목록 나옴:
- uno
- esp32

### 방법 2: 명령 팔레트

`Ctrl + Shift + P` → "PlatformIO: Switch Project Environment"

---

## 환경별 빌드

### 현재 환경만

```bash
pio run
```

또는 ✓ 버튼.

### 특정 환경

```bash
pio run -e uno
pio run -e esp32
```

### 모든 환경

```bash
pio run
```

환경 지정 안 하면 모든 환경 빌드.

---

## 환경별 업로드

```bash
pio run -e uno -t upload
pio run -e esp32 -t upload
```

또는 환경 선택 후 → 버튼.

---

## 기본 환경 지정

```ini
[platformio]
default_envs = esp32

[env:uno]
...

[env:esp32]
...
```

`default_envs`로 기본 환경 지정.

빌드/업로드 시 이 환경 사용.

---

## 공통 설정

여러 환경에서 같은 설정 반복?

`[env]` 섹션으로 공통화:

```ini
[platformio]
default_envs = esp32

[env]
; 모든 환경에 적용
framework = arduino
monitor_speed = 115200
lib_deps =
    bblanchon/ArduinoJson@^6.21.0

[env:uno]
platform = atmelavr
board = uno

[env:esp32]
platform = espressif32
board = esp32dev
```

`[env]`에 쓴 건 모든 환경에 상속.

---

## 환경별 다른 설정

```ini
[env]
framework = arduino
monitor_speed = 115200

[env:uno]
platform = atmelavr
board = uno
; Uno는 라이브러리 추가
lib_deps = 
    adafruit/DHT sensor library@^1.4.4

[env:esp32]
platform = espressif32
board = esp32dev
; ESP32는 WiFi 라이브러리
lib_deps =
    bblanchon/ArduinoJson@^6.21.0
```

---

## 같은 보드, 다른 설정

디버그 빌드 vs 릴리즈 빌드:

```ini
[env:esp32_debug]
platform = espressif32
board = esp32dev
framework = arduino
build_flags = 
    -D DEBUG_MODE
    -O0                    ; 최적화 없음

[env:esp32_release]
platform = espressif32
board = esp32dev
framework = arduino
build_flags = 
    -D RELEASE_MODE
    -O2                    ; 최적화
```

---

## 포트 지정

보드 여러 개 연결했을 때:

```ini
[env:uno]
platform = atmelavr
board = uno
framework = arduino
upload_port = COM3
monitor_port = COM3

[env:esp32]
platform = espressif32
board = esp32dev
framework = arduino
upload_port = COM5
monitor_port = COM5
```

포트 직접 지정해서 혼동 방지.

---

## 실용 예시

BMS 프로젝트:

```ini
[platformio]
default_envs = stm32

[env]
framework = arduino
monitor_speed = 115200

; 프로토타입용 Arduino Mega
[env:mega]
platform = atmelavr
board = megaatmega2560

; 실제 제품용 STM32
[env:stm32]
platform = ststm32
board = bluepill_f103c8
upload_protocol = stlink

; 게이트웨이용 ESP32
[env:esp32]
platform = espressif32
board = esp32dev
lib_deps =
    bblanchon/ArduinoJson@^6.21.0
```

---

## 다음 단계

멀티보드 설정 완료!

다음 글에서 환경별 빌드.

---

다음 글: [#14 - 환경별 빌드](/posts/platformio/platformio-14/)
