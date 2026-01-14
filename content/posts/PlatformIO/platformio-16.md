---
title: "PlatformIO 완벽 가이드 #16 - 빌드 플래그"
date: 2024-12-22
tags: ["PlatformIO", "빌드", "컴파일러", "플래그"]
categories: ["개발환경"]
series: ["PlatformIO 완벽 가이드"]
summary: "컴파일러 옵션으로 빌드 제어하기."
---

`build_flags`로 컴파일러에 옵션을 전달할 수 있다.

매크로 정의, 최적화 레벨, 경고 설정 등.

---

## 기본 문법

```ini
[env:esp32]
platform = espressif32
board = esp32dev
framework = arduino

build_flags =
    -D DEBUG_MODE
    -Wall
    -O2
```

---

## 매크로 정의 (-D)

```ini
build_flags =
    -D DEBUG_MODE              ; #define DEBUG_MODE
    -D VERSION=10              ; #define VERSION 10
    -D NAME=\"MyDevice\"       ; #define NAME "MyDevice"
```

코드에서:

```cpp
#ifdef DEBUG_MODE
  Serial.println("Debug mode enabled");
#endif

Serial.print("Version: ");
Serial.println(VERSION);

Serial.print("Name: ");
Serial.println(NAME);
```

**주의**: 문자열은 `\"` 로 이스케이프.

---

## 경고 옵션

```ini
build_flags =
    -Wall           ; 모든 경고 활성화
    -Wextra         ; 추가 경고
    -Werror         ; 경고를 에러로 처리
    -Wno-unused     ; 사용 안 하는 변수 경고 끄기
```

---

## 최적화 레벨

```ini
build_flags =
    -O0   ; 최적화 없음 (디버깅용)
    -O1   ; 기본 최적화
    -O2   ; 권장 최적화
    -O3   ; 최대 최적화 (코드 커질 수 있음)
    -Os   ; 크기 최적화
```

디버깅할 때는 `-O0`, 릴리즈는 `-O2`.

---

## ESP32 전용

```ini
[env:esp32]
platform = espressif32
board = esp32dev
framework = arduino

build_flags =
    -D CORE_DEBUG_LEVEL=5     ; ESP32 로그 레벨
    -D CONFIG_ASYNC_TCP_RUNNING_CORE=1
```

`CORE_DEBUG_LEVEL`:
- 0: None
- 1: Error
- 2: Warn
- 3: Info
- 4: Debug
- 5: Verbose

---

## STM32 전용

```ini
[env:stm32]
platform = ststm32
board = bluepill_f103c8
framework = arduino

build_flags =
    -D PIO_FRAMEWORK_ARDUINO_ENABLE_CDC
    -D USBCON
    -D USB_VID=0x1234
    -D USB_PID=0x5678
```

---

## 인클루드 경로 추가

```ini
build_flags =
    -I include/custom
    -I lib/external/include
```

`-I` 로 헤더 검색 경로 추가.

---

## 환경변수 사용

```ini
build_flags =
    -D WIFI_SSID=\"${sysenv.WIFI_SSID}\"
    -D WIFI_PASS=\"${sysenv.WIFI_PASSWORD}\"
```

시스템 환경변수에서 값 가져오기.

비밀번호 같은 민감한 정보 코드에 안 넣어도 됨.

---

## 파일에서 플래그 읽기

```ini
build_flags =
    !python get_flags.py
```

`!`로 시작하면 명령 실행 결과를 사용.

`get_flags.py`:

```python
import subprocess
import datetime

print("-D BUILD_TIME=" + str(int(datetime.datetime.now().timestamp())))
```

---

## build_unflags

기본 플래그 제거:

```ini
build_unflags =
    -Os        ; 크기 최적화 제거
    -std=gnu++11

build_flags =
    -O2        ; 속도 최적화로 교체
    -std=gnu++17
```

---

## 디버그 vs 릴리즈

```ini
[env:debug]
platform = espressif32
board = esp32dev
framework = arduino
build_type = debug
build_flags =
    -D DEBUG_MODE
    -O0
    -g                  ; 디버그 심볼

[env:release]
platform = espressif32
board = esp32dev
framework = arduino
build_type = release
build_flags =
    -D RELEASE_MODE
    -O2
    -DNDEBUG            ; assert 비활성화
```

---

## 전체 예시

```ini
[env:esp32]
platform = espressif32
board = esp32dev
framework = arduino

build_flags =
    ; 매크로
    -D VERSION=\"1.0.0\"
    -D BUILD_ENV=\"production\"
    
    ; 디버그
    -D CORE_DEBUG_LEVEL=3
    
    ; 최적화
    -O2
    
    ; 경고
    -Wall
    -Wextra
    
    ; 인클루드
    -I include
```

---

## 다음 단계

빌드 플래그 완료!

다음 글에서 디버거 연결.

---

다음 글: [#17 - 디버거 연결 (ST-Link)](/posts/platformio/platformio-17/)
