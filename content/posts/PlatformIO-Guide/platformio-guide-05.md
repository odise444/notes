---
title: "PlatformIO 입문 #5 - 프로젝트 구조"
date: 2024-12-22
tags: ["PlatformIO", "프로젝트", "폴더구조"]
categories: ["가이드"]
series: ["PlatformIO 입문"]
summary: "각 폴더가 무슨 역할인지 알아야 제대로 쓴다."
---

프로젝트 만들었더니 폴더가 여러 개다.

각각 뭐 하는 건지 알아보자.

---

## 전체 구조

```
my_project/
├── .pio/              ← 빌드 결과물 (자동 생성)
├── .vscode/           ← VSCode 설정 (자동 생성)
├── include/           ← 헤더 파일 (.h)
├── lib/               ← 프로젝트 전용 라이브러리
├── src/               ← 소스 코드 (.cpp)
├── test/              ← 유닛 테스트
└── platformio.ini     ← 프로젝트 설정 파일
```

---

## src/ 폴더

**메인 소스 코드** 넣는 곳.

```
src/
└── main.cpp
```

### 여러 파일 가능

```
src/
├── main.cpp
├── sensor.cpp
├── motor.cpp
└── display.cpp
```

자동으로 전부 컴파일됨.

### 하위 폴더도 가능

```
src/
├── main.cpp
├── drivers/
│   ├── sensor.cpp
│   └── motor.cpp
└── ui/
    └── display.cpp
```

---

## include/ 폴더

**헤더 파일** 넣는 곳.

```
include/
├── config.h
├── sensor.h
└── motor.h
```

### 사용법

```cpp
// main.cpp
#include "config.h"    // include 폴더에서 찾음
#include "sensor.h"
```

큰따옴표 `""` 쓰면 include 폴더 먼저 검색.

### 왜 분리하나?

```
헤더 (.h):    선언 (이런 함수 있어요)
소스 (.cpp):  정의 (함수 내용은 이거예요)
```

코드가 커지면 분리하는 게 관리하기 좋음.

---

## lib/ 폴더

**프로젝트 전용 라이브러리**.

### 구조

```
lib/
└── MyLibrary/
    ├── MyLibrary.h
    └── MyLibrary.cpp
```

또는

```
lib/
└── MyLibrary/
    ├── src/
    │   ├── MyLibrary.h
    │   └── MyLibrary.cpp
    └── library.json     ← 선택사항
```

### 사용법

```cpp
// main.cpp
#include <MyLibrary.h>   // 꺾쇠로도 됨
```

자동으로 lib 폴더 검색.

### 언제 쓰나?

- 직접 만든 라이브러리
- 외부 라이브러리 수정해서 쓸 때
- 공개 안 된 라이브러리

---

## test/ 폴더

**유닛 테스트** 코드.

```
test/
└── test_main.cpp
```

`pio test` 명령으로 실행.

나중에 자세히 다룸.

---

## .pio/ 폴더

**빌드 결과물**. 자동 생성.

```
.pio/
├── build/
│   └── esp32dev/
│       ├── firmware.bin
│       ├── firmware.elf
│       └── ...
└── libdeps/
    └── esp32dev/
        └── (설치된 라이브러리들)
```

### firmware.bin

업로드되는 바이너리 파일.

### libdeps/

platformio.ini에서 지정한 라이브러리가 여기 설치됨.

### 주의

이 폴더는:
- Git에 올리지 않음 (`.gitignore`에 포함)
- 삭제해도 됨 (다시 빌드하면 생성)
- 직접 수정하지 않음

---

## .vscode/ 폴더

**VSCode 설정**. 자동 생성.

```
.vscode/
├── c_cpp_properties.json   ← IntelliSense 설정
├── extensions.json         ← 추천 확장 프로그램
└── launch.json             ← 디버그 설정
```

PlatformIO가 알아서 관리.

보통 건드릴 일 없음.

---

## platformio.ini

**핵심 설정 파일**.

```ini
[env:esp32dev]
platform = espressif32
board = esp32dev
framework = arduino
```

다음 글에서 자세히 다룸.

---

## 정리

| 폴더/파일 | 용도 | 직접 수정 |
|----------|------|----------|
| `src/` | 메인 소스 코드 | ⭕ |
| `include/` | 헤더 파일 | ⭕ |
| `lib/` | 프로젝트 전용 라이브러리 | ⭕ |
| `test/` | 유닛 테스트 | ⭕ |
| `platformio.ini` | 프로젝트 설정 | ⭕ |
| `.pio/` | 빌드 결과물 | ❌ |
| `.vscode/` | VSCode 설정 | 보통 ❌ |

---

## 실제 프로젝트 예시

BMS 프로젝트라면:

```
bms_firmware/
├── include/
│   ├── config.h           ← 설정값
│   ├── battery.h          ← 배터리 관련 선언
│   └── can_protocol.h     ← CAN 프로토콜 선언
├── lib/
│   └── AD7280A/           ← AFE 드라이버 (직접 작성)
│       ├── AD7280A.h
│       └── AD7280A.cpp
├── src/
│   ├── main.cpp           ← 메인 루프
│   ├── battery.cpp        ← 배터리 관리
│   ├── can.cpp            ← CAN 통신
│   └── protection.cpp     ← 보호 로직
├── test/
│   └── test_battery.cpp   ← 배터리 로직 테스트
└── platformio.ini
```

---

다음 글에서 platformio.ini 파헤치기.

[#6 - platformio.ini 해부](/posts/platformio-guide/platformio-guide-06/)
