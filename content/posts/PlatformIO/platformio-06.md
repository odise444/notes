---
title: "PlatformIO 완벽 가이드 #6 - 폴더 구조 이해하기"
date: 2024-12-22
tags: ["PlatformIO", "프로젝트", "폴더구조"]
categories: ["개발환경"]
series: ["PlatformIO 완벽 가이드"]
summary: "각 폴더가 무슨 역할을 하는지."
---

PlatformIO 프로젝트 폴더 구조를 이해하자.

어디에 뭘 넣어야 하는지 알아야 정리가 된다.

---

## 전체 구조

```
MyProject/
├── .pio/              ← 빌드 결과물 (건드리지 마)
├── .vscode/           ← VSCode 설정
├── include/           ← 헤더 파일 (.h)
├── lib/               ← 프로젝트 전용 라이브러리
├── src/               ← 소스 코드 (.cpp)
├── test/              ← 유닛 테스트
└── platformio.ini     ← 프로젝트 설정
```

---

## src/ 폴더

**메인 소스 코드가 들어가는 곳.**

```
src/
├── main.cpp           ← 진입점
├── sensors.cpp        ← 센서 관련
└── motors.cpp         ← 모터 관련
```

`main.cpp`에 `setup()`과 `loop()`가 있어야 함.

파일 여러 개로 분리해도 됨.

---

## include/ 폴더

**헤더 파일(.h) 넣는 곳.**

```
include/
├── sensors.h
├── motors.h
└── config.h
```

`src/`의 `.cpp` 파일에서:

```cpp
#include "sensors.h"  // include/ 폴더에서 찾음
```

---

## lib/ 폴더

**프로젝트 전용 라이브러리.**

공식 라이브러리가 아닌, 직접 만든 라이브러리.

```
lib/
└── MyCustomLib/
    ├── MyCustomLib.h
    └── MyCustomLib.cpp
```

사용:

```cpp
#include <MyCustomLib.h>
```

자동으로 인식됨.

---

## .pio/ 폴더

**빌드 결과물과 다운로드한 패키지.**

```
.pio/
├── build/
│   └── uno/
│       ├── firmware.elf
│       ├── firmware.hex
│       └── ...
└── libdeps/
    └── uno/
        └── (설치된 라이브러리들)
```

**절대 직접 수정하지 마.**

용량 커지면 삭제해도 됨. 다시 빌드하면 생성됨.

---

## .vscode/ 폴더

**VSCode 설정.**

```
.vscode/
├── c_cpp_properties.json   ← IntelliSense 설정
├── extensions.json         ← 추천 확장
└── launch.json             ← 디버그 설정
```

PlatformIO가 자동 생성.

IntelliSense 안 되면 이 파일 확인.

---

## test/ 폴더

**유닛 테스트 코드.**

```
test/
└── test_main.cpp
```

나중에 다룰 예정.

---

## platformio.ini

**프로젝트 설정 파일. 제일 중요!**

```ini
[env:uno]
platform = atmelavr
board = uno
framework = arduino
```

다음 글에서 자세히.

---

## .gitignore

Git 사용한다면, 이건 제외:

```
.pio/
.vscode/
```

이미 포함되어 있을 수 있음.

---

## Arduino IDE와 비교

| Arduino IDE | PlatformIO |
|-------------|------------|
| MyProject.ino | src/main.cpp |
| 같은 폴더에 .h | include/ 폴더 |
| 라이브러리 폴더 전역 | lib/ 폴더 (프로젝트별) |
| 빌드 결과 임시폴더 | .pio/build/ |

---

## 폴더 정리 팁

### 파일 많아지면

```
src/
├── main.cpp
├── sensors/
│   ├── temperature.cpp
│   └── humidity.cpp
└── actuators/
    ├── motor.cpp
    └── servo.cpp
```

하위 폴더로 분류 가능.

### 헤더와 소스 같이

```
src/
├── main.cpp
├── sensors.h
└── sensors.cpp
```

`include/` 안 쓰고 `src/`에 다 넣어도 됨.

---

## 다음 단계

폴더 구조 파악 완료.

다음 글에서 platformio.ini 설정.

---

다음 글: [#7 - platformio.ini 설정](/posts/platformio/platformio-07/)
