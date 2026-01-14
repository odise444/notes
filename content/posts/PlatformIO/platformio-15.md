---
title: "PlatformIO 완벽 가이드 #15 - 공통 코드 분리"
date: 2024-12-22
tags: ["PlatformIO", "모듈화", "라이브러리"]
categories: ["개발환경"]
series: ["PlatformIO 완벽 가이드"]
summary: "여러 프로젝트에서 재사용할 코드 분리하기."
---

센서 드라이버를 여러 프로젝트에서 쓰고 싶다.

복사-붙여넣기? 비효율적.

공통 라이브러리로 분리하자.

---

## lib/ 폴더 활용

프로젝트 내 `lib/` 폴더:

```
MyProject/
├── lib/
│   └── MySensor/
│       ├── MySensor.h
│       └── MySensor.cpp
├── src/
│   └── main.cpp
└── platformio.ini
```

---

## 라이브러리 구조

`lib/MySensor/MySensor.h`:

```cpp
#ifndef MY_SENSOR_H
#define MY_SENSOR_H

#include <Arduino.h>

class MySensor {
public:
    MySensor(uint8_t pin);
    void begin();
    int read();
    
private:
    uint8_t _pin;
};

#endif
```

`lib/MySensor/MySensor.cpp`:

```cpp
#include "MySensor.h"

MySensor::MySensor(uint8_t pin) : _pin(pin) {}

void MySensor::begin() {
    pinMode(_pin, INPUT);
}

int MySensor::read() {
    return analogRead(_pin);
}
```

---

## 사용

`src/main.cpp`:

```cpp
#include <Arduino.h>
#include <MySensor.h>  // lib/ 폴더에서 자동 인식

MySensor sensor(A0);

void setup() {
    Serial.begin(115200);
    sensor.begin();
}

void loop() {
    int value = sensor.read();
    Serial.println(value);
    delay(1000);
}
```

별도 설정 없이 바로 사용.

---

## library.json (선택)

라이브러리 메타데이터:

`lib/MySensor/library.json`:

```json
{
    "name": "MySensor",
    "version": "1.0.0",
    "description": "Custom analog sensor library",
    "keywords": "sensor, analog",
    "authors": [
        {
            "name": "Your Name",
            "email": "you@example.com"
        }
    ],
    "frameworks": "arduino",
    "platforms": "*"
}
```

없어도 되지만 있으면 PIO Home에서 정보 표시.

---

## 여러 프로젝트에서 공유

### 방법 1: 심볼릭 링크

```
SharedLibs/
└── MySensor/
    ├── MySensor.h
    └── MySensor.cpp

Project1/lib/MySensor -> ../../SharedLibs/MySensor
Project2/lib/MySensor -> ../../SharedLibs/MySensor
```

Windows:
```cmd
mklink /D lib\MySensor C:\SharedLibs\MySensor
```

Linux/Mac:
```bash
ln -s /path/to/SharedLibs/MySensor lib/MySensor
```

### 방법 2: lib_extra_dirs

`platformio.ini`:

```ini
[env:esp32]
platform = espressif32
board = esp32dev
framework = arduino

lib_extra_dirs = 
    C:/SharedLibs
    ~/my-arduino-libs
```

`SharedLibs/` 안의 모든 라이브러리 인식.

---

## GitHub에서 공유

라이브러리를 GitHub 저장소로:

```
MySensorLib/
├── src/
│   ├── MySensor.h
│   └── MySensor.cpp
├── library.json
└── README.md
```

사용:

```ini
lib_deps =
    https://github.com/yourname/MySensorLib.git
```

---

## 버전 관리

```ini
lib_deps =
    https://github.com/yourname/MySensorLib.git#v1.0.0  ; 태그
    https://github.com/yourname/MySensorLib.git#main    ; 브랜치
    https://github.com/yourname/MySensorLib.git#abc123  ; 커밋
```

---

## 프로젝트 구조 예시

큰 프로젝트:

```
BMS_Project/
├── lib/
│   ├── BMS_Core/           ; BMS 핵심 로직
│   │   ├── bms.h
│   │   └── bms.cpp
│   ├── CAN_Driver/         ; CAN 통신
│   │   ├── can_driver.h
│   │   └── can_driver.cpp
│   ├── Flash_Driver/       ; Flash 저장
│   │   ├── flash.h
│   │   └── flash.cpp
│   └── UI/                 ; LCD/LED 표시
│       ├── ui.h
│       └── ui.cpp
├── src/
│   └── main.cpp            ; 진입점만
├── include/
│   └── config.h            ; 설정
└── platformio.ini
```

`main.cpp`는 최소화, 로직은 lib/에.

---

## lib_compat_mode

라이브러리 호환성 모드:

```ini
lib_compat_mode = strict  ; 엄격한 의존성 체크 (기본)
lib_compat_mode = soft    ; 느슨한 체크
lib_compat_mode = off     ; 체크 안 함
```

라이브러리 의존성 문제 있을 때 `soft`나 `off` 시도.

---

## 다음 단계

공통 코드 분리 완료!

다음 글에서 빌드 플래그.

---

다음 글: [#16 - 빌드 플래그](/posts/platformio/platformio-16/)
