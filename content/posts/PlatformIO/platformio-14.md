---
title: "PlatformIO 완벽 가이드 #14 - 환경별 빌드"
date: 2024-12-22
tags: ["PlatformIO", "빌드", "조건부컴파일"]
categories: ["개발환경"]
series: ["PlatformIO 완벽 가이드"]
summary: "Uno에서는 이 코드, ESP32에서는 저 코드."
---

같은 프로젝트인데 보드마다 코드가 다를 때.

조건부 컴파일로 해결.

---

## 문제 상황

```cpp
#include <WiFi.h>  // ESP32만 있음

void setup() {
    WiFi.begin("SSID", "PASS");  // Uno에서 에러
}
```

Uno에는 WiFi 없어서 컴파일 에러.

---

## 해결: 조건부 컴파일

```cpp
#ifdef ESP32
  #include <WiFi.h>
#endif

void setup() {
    Serial.begin(115200);
    
    #ifdef ESP32
    WiFi.begin("SSID", "PASS");
    Serial.println("WiFi connecting...");
    #else
    Serial.println("No WiFi on this board");
    #endif
}
```

`ESP32`가 정의되어 있으면 그 코드 포함.

---

## 보드별 매크로

PlatformIO가 자동 정의하는 매크로:

| 보드 | 매크로 |
|------|--------|
| Arduino Uno | `ARDUINO_AVR_UNO` |
| Arduino Mega | `ARDUINO_AVR_MEGA2560` |
| ESP32 | `ESP32` |
| ESP8266 | `ESP8266` |
| STM32 | `STM32F1xx` 등 |

---

## 플랫폼별 분기

```cpp
#if defined(ESP32)
  // ESP32 전용 코드
  #include <WiFi.h>
#elif defined(ESP8266)
  // ESP8266 전용 코드
  #include <ESP8266WiFi.h>
#elif defined(ARDUINO_AVR_UNO)
  // Arduino Uno 전용 코드
#else
  // 기타
#endif
```

---

## build_flags로 매크로 추가

`platformio.ini`:

```ini
[env:uno]
platform = atmelavr
board = uno
framework = arduino
build_flags = 
    -D BOARD_UNO
    -D HAS_SERIAL_ONLY

[env:esp32]
platform = espressif32
board = esp32dev
framework = arduino
build_flags = 
    -D BOARD_ESP32
    -D HAS_WIFI
    -D HAS_BLUETOOTH
```

코드에서:

```cpp
#ifdef HAS_WIFI
  WiFi.begin(ssid, pass);
#endif

#ifdef HAS_BLUETOOTH
  BLEDevice::init("MyDevice");
#endif
```

---

## 버전 정보 전달

```ini
build_flags =
    -D VERSION=\"1.0.0\"
    -D BUILD_TIME=\"${UNIX_TIME}\"
```

코드에서:

```cpp
Serial.print("Version: ");
Serial.println(VERSION);
```

**주의**: 문자열은 `\"` 로 이스케이프.

---

## 디버그 모드

```ini
[env:esp32_debug]
platform = espressif32
board = esp32dev
framework = arduino
build_flags = 
    -D DEBUG_MODE
    -DCORE_DEBUG_LEVEL=5

[env:esp32_release]
platform = espressif32
board = esp32dev
framework = arduino
build_flags = 
    -D RELEASE_MODE
    -O2
```

코드에서:

```cpp
#ifdef DEBUG_MODE
  #define DEBUG_PRINT(x) Serial.println(x)
#else
  #define DEBUG_PRINT(x)
#endif

void loop() {
    DEBUG_PRINT("Loop running");  // 릴리즈에선 제거됨
}
```

---

## 파일 레벨 분리

보드별로 아예 다른 파일:

```
src/
├── main.cpp
├── wifi_esp32.cpp      ; ESP32 전용
├── wifi_esp8266.cpp    ; ESP8266 전용
└── wifi_dummy.cpp      ; WiFi 없는 보드
```

`platformio.ini`:

```ini
[env:esp32]
platform = espressif32
board = esp32dev
framework = arduino
build_src_filter = 
    +<*>
    -<wifi_esp8266.cpp>
    -<wifi_dummy.cpp>

[env:uno]
platform = atmelavr
board = uno
framework = arduino
build_src_filter = 
    +<*>
    -<wifi_esp32.cpp>
    -<wifi_esp8266.cpp>
```

`build_src_filter`로 포함/제외 파일 지정.

---

## 추상화 패턴

인터페이스로 추상화:

```cpp
// wifi_interface.h
class IWiFi {
public:
    virtual void begin(const char* ssid, const char* pass) = 0;
    virtual bool isConnected() = 0;
};

// wifi_esp32.cpp
#ifdef ESP32
class WiFiESP32 : public IWiFi {
    void begin(const char* ssid, const char* pass) override {
        WiFi.begin(ssid, pass);
    }
    bool isConnected() override {
        return WiFi.status() == WL_CONNECTED;
    }
};
#endif
```

---

## 다음 단계

환경별 빌드 완료!

다음 글에서 공통 코드 분리.

---

다음 글: [#15 - 공통 코드 분리](/posts/platformio/platformio-15/)
