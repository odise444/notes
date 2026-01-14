---
title: "PlatformIO 입문 #9 - 라이브러리 관리"
date: 2024-12-22
tags: ["PlatformIO", "라이브러리", "lib_deps"]
categories: ["가이드"]
series: ["PlatformIO 입문"]
summary: "라이브러리 설치가 Arduino IDE보다 훨씬 편하다."
---

외부 라이브러리 써야 할 때.

Arduino IDE는 전역 설치인데, PlatformIO는 **프로젝트별 관리**.

---

## 라이브러리 검색

### 방법 1: PIO Home

1. PlatformIO 아이콘 → PIO Home
2. **Libraries** 탭
3. 검색창에 라이브러리 이름

예: `ArduinoJson` 검색.

### 방법 2: 웹사이트

https://registry.platformio.org/

더 상세한 정보 있음.

### 방법 3: 터미널

```bash
pio pkg search "json"
```

---

## 라이브러리 설치

### 방법 1: platformio.ini

**가장 추천하는 방법!**

```ini
[env:esp32dev]
platform = espressif32
board = esp32dev
framework = arduino

lib_deps = 
    bblanchon/ArduinoJson @ ^6.21.0
```

저장하면 자동 설치.

### 방법 2: PIO Home에서

1. Libraries → 검색
2. 라이브러리 클릭
3. **Add to Project** 버튼
4. 프로젝트 선택

자동으로 `platformio.ini`에 추가됨.

### 방법 3: 터미널

```bash
pio pkg install --library "bblanchon/ArduinoJson@^6.21.0"
```

---

## lib_deps 문법

### 기본 형식

```ini
lib_deps = 
    라이브러리이름
```

### 버전 지정

```ini
lib_deps = 
    ArduinoJson @ 6.21.0      ; 정확히 이 버전
    ArduinoJson @ ^6.21.0     ; 6.21.0 이상, 7.0.0 미만
    ArduinoJson @ ~6.21.0     ; 6.21.0 이상, 6.22.0 미만
    ArduinoJson @ >=6.0.0     ; 6.0.0 이상
```

### 소유자/이름 형식

```ini
lib_deps = 
    bblanchon/ArduinoJson     ; 정확한 라이브러리 지정
```

이름만 쓰면 동명의 다른 라이브러리 설치될 수 있음.

### 여러 라이브러리

```ini
lib_deps = 
    bblanchon/ArduinoJson @ ^6.21.0
    adafruit/Adafruit SSD1306 @ ^2.5.0
    adafruit/Adafruit GFX Library @ ^1.11.0
    Wire                                       ; Arduino 내장
```

---

## GitHub에서 직접

공개 라이브러리가 아니거나, 최신 버전 필요할 때:

```ini
lib_deps = 
    ; GitHub 저장소
    https://github.com/me-no-dev/ESPAsyncWebServer.git
    
    ; 특정 브랜치
    https://github.com/user/repo.git#develop
    
    ; 특정 태그
    https://github.com/user/repo.git#v1.2.3
    
    ; 특정 커밋
    https://github.com/user/repo.git#abc1234
```

---

## 설치 위치

```
.pio/
└── libdeps/
    └── esp32dev/
        ├── ArduinoJson/
        ├── Adafruit SSD1306/
        └── ...
```

프로젝트 폴더 안에 설치됨.

**프로젝트별로 독립적!**

A 프로젝트: ArduinoJson 6.21.0
B 프로젝트: ArduinoJson 7.0.0

충돌 없음.

---

## 라이브러리 사용

설치했으면 바로 include:

```cpp
#include <ArduinoJson.h>

void setup() {
  Serial.begin(115200);
  
  StaticJsonDocument<200> doc;
  doc["sensor"] = "temperature";
  doc["value"] = 25.5;
  
  serializeJson(doc, Serial);
}

void loop() {
}
```

빌드하면 자동으로 링크됨.

---

## 의존성 자동 해결

Adafruit SSD1306 설치하면:

```ini
lib_deps = 
    adafruit/Adafruit SSD1306
```

의존하는 라이브러리 자동 설치:
- Adafruit GFX Library
- Adafruit BusIO

직접 안 써도 됨.

---

## 프로젝트 전용 라이브러리

직접 만들거나 수정한 라이브러리:

```
lib/
└── MyDriver/
    ├── MyDriver.h
    └── MyDriver.cpp
```

`lib/` 폴더에 넣으면 자동 인식.

```cpp
#include <MyDriver.h>
```

---

## 라이브러리 업데이트

### 확인

```bash
pio pkg outdated
```

### 업데이트

```bash
pio pkg update
```

### 특정 라이브러리만

```bash
pio pkg update --library "ArduinoJson"
```

---

## 라이브러리 삭제

### platformio.ini에서 제거

`lib_deps`에서 해당 줄 삭제 → 저장.

다음 빌드 시 안 쓰는 건 남아있긴 함.

### 완전 삭제

```bash
pio pkg uninstall --library "ArduinoJson"
```

또는 `.pio/libdeps/` 폴더 삭제.

---

## Arduino IDE 라이브러리와 호환

대부분 호환됨.

Arduino Library Manager에 있는 건 PlatformIO Registry에도 있음.

```
Arduino:     라이브러리 관리자 → DHT sensor library
PlatformIO:  adafruit/DHT sensor library
```

---

## 주의사항

### 라이브러리 이름 정확히

```ini
; ❌ 틀림
lib_deps = ArduinoJSON

; ✅ 맞음
lib_deps = ArduinoJson
```

대소문자 주의.

### 버전 호환성

```ini
; 너무 오래된 버전
lib_deps = ArduinoJson @ 5.0.0

; 호환 안 될 수 있음
```

가능하면 최신 버전 사용.

---

다음 글에서 멀티 환경 설정.

[#10 - 멀티 환경 설정](/posts/platformio-guide/platformio-guide-10/)
