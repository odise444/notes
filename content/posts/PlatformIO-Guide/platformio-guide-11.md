---
title: "PlatformIO 입문 #11 - 빌드 플래그"
date: 2024-12-22
tags: ["PlatformIO", "빌드플래그", "매크로", "컴파일"]
categories: ["가이드"]
series: ["PlatformIO 입문"]
summary: "컴파일 옵션으로 코드 동작을 바꾼다."
---

빌드 플래그는 **컴파일러에 전달하는 옵션**.

코드 수정 없이 동작을 바꿀 수 있다.

---

## 기본 사용법

```ini
[env:esp32]
platform = espressif32
board = esp32dev
framework = arduino

build_flags = 
    -D DEBUG_MODE
    -D VERSION=123
```

`-D`는 매크로 정의.

---

## 코드에서 사용

```cpp
#include <Arduino.h>

void setup() {
  Serial.begin(115200);
  
  #ifdef DEBUG_MODE
    Serial.println("Debug mode enabled");
  #endif
  
  Serial.print("Version: ");
  Serial.println(VERSION);  // 123 출력
}

void loop() {
}
```

---

## 문자열 전달

따옴표 이스케이프 필요:

```ini
build_flags = 
    -D DEVICE_NAME=\"ESP32-Dev\"
    -D WIFI_SSID=\"MyNetwork\"
```

```cpp
Serial.println(DEVICE_NAME);  // "ESP32-Dev" 출력
```

---

## 자주 쓰는 플래그

### 디버그 레벨 (ESP32)

```ini
build_flags = 
    -D CORE_DEBUG_LEVEL=5    ; 모든 로그
    ; 0=None, 1=Error, 2=Warn, 3=Info, 4=Debug, 5=Verbose
```

### 최적화 레벨

```ini
build_flags = 
    -O0    ; 최적화 없음 (디버깅용)
    -O2    ; 속도 최적화
    -Os    ; 크기 최적화 (기본값)
    -O3    ; 최대 속도 최적화
```

### 경고 설정

```ini
build_flags = 
    -Wall           ; 모든 경고 켜기
    -Wextra         ; 추가 경고
    -Werror         ; 경고를 에러로
    -Wno-unused-variable  ; 특정 경고 끄기
```

---

## 조건부 컴파일 패턴

### 기능 토글

```ini
; 개발용
[env:dev]
build_flags = 
    -D DEBUG_MODE
    -D FEATURE_WIFI
    -D FEATURE_BLUETOOTH

; 릴리즈용
[env:release]
build_flags = 
    -D FEATURE_WIFI
    ; 블루투스 제외
```

```cpp
void setup() {
  #ifdef FEATURE_WIFI
    initWiFi();
  #endif
  
  #ifdef FEATURE_BLUETOOTH
    initBluetooth();
  #endif
  
  #ifdef DEBUG_MODE
    Serial.println("Debug build");
  #endif
}
```

### 하드웨어 버전

```ini
[env:v1]
build_flags = -D HARDWARE_VERSION=1

[env:v2]
build_flags = -D HARDWARE_VERSION=2
```

```cpp
#if HARDWARE_VERSION == 1
  #define LED_PIN 13
#elif HARDWARE_VERSION == 2
  #define LED_PIN 2
#endif
```

---

## build_unflags

기존 플래그 제거:

```ini
build_flags = -Os          ; 기본 최적화
build_unflags = -Os        ; 제거
build_flags = -O0          ; 다른 걸로 교체
```

---

## 환경별 플래그

```ini
[env]
build_flags = 
    -D COMMON_FLAG

[env:esp32]
build_flags = 
    ${env.build_flags}     ; 공통 상속
    -D ESP32_FLAG

[env:uno]
build_flags = 
    ${env.build_flags}
    -D ARDUINO_FLAG
```

---

## 인클루드 경로 추가

```ini
build_flags = 
    -I include/mylib
    -I lib/external/include
```

헤더 파일 검색 경로 추가.

---

## 소스 정의 추가

```ini
build_src_flags = 
    -D SRC_ONLY_FLAG
```

`src/` 폴더 파일에만 적용. 라이브러리엔 적용 안 됨.

---

## 실제 예시

### BMS 프로젝트

```ini
[env:bms_dev]
platform = ststm32
board = genericSTM32F103VE
framework = arduino
build_flags = 
    -D DEBUG_MODE
    -D CELL_COUNT=24
    -D BALANCE_THRESHOLD=50
    -D CAN_SPEED=500000
    -D SERIAL_DEBUG

[env:bms_release]
platform = ststm32
board = genericSTM32F103VE
framework = arduino
build_flags = 
    -D CELL_COUNT=24
    -D BALANCE_THRESHOLD=30
    -D CAN_SPEED=500000
    ; DEBUG 없음
```

### ESP32 웹서버

```ini
[env:webserver]
platform = espressif32
board = esp32dev
framework = arduino
build_flags = 
    -D WIFI_SSID=\"MyNetwork\"
    -D WIFI_PASS=\"MyPassword\"
    -D WEB_PORT=80
    -D ASYNC_TCP_SSL_ENABLED=0
    -D CONFIG_ASYNC_TCP_QUEUE_SIZE=128
```

---

## 외부 파일에서 읽기

```ini
build_flags = 
    !python scripts/get_version.py
```

스크립트 실행 결과를 플래그로.

```python
# scripts/get_version.py
import subprocess
version = subprocess.check_output(['git', 'describe', '--tags']).strip()
print(f"-D GIT_VERSION=\\\"{version.decode()}\\\"")
```

---

## 정리

| 용도 | 플래그 |
|------|--------|
| 매크로 정의 | `-D NAME` |
| 값 있는 매크로 | `-D NAME=VALUE` |
| 문자열 매크로 | `-D NAME=\"string\"` |
| 최적화 | `-O0`, `-Os`, `-O2` |
| 경고 | `-Wall`, `-Werror` |
| 인클루드 경로 | `-I path` |

---

다음 글에서 디버깅.

[#12 - 디버깅](/posts/platformio-guide/platformio-guide-12/)
