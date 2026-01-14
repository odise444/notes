---
title: "PlatformIO 입문 #1 - PlatformIO란?"
date: 2026-01-01
tags:
  - PlatformIO
  - VSCode
  - 임베디드
  - Arduino
categories:
  - 가이드
series:
  - PlatformIO 입문
summary: Arduino IDE 쓰다가 답답해서 찾아보니 이런 게 있더라.
---

Arduino IDE로 시작했다.

LED 깜빡이기, 센서 읽기... 잘 됐다.

근데 프로젝트가 커지니까 불편해졌다.

---

## Arduino IDE의 한계

### 1. 코드 편집기가 구림

```
- 자동완성 느림
- 함수 정의로 점프 안 됨
- 문법 오류 실시간 표시 안 됨
- 테마 변경 안 됨
```

메모장에 색깔 입힌 수준.

### 2. 프로젝트 관리가 안 됨

```
- 파일 여러 개면 탭 지옥
- 폴더 구조 못 만듦
- 버전 관리 (Git) 연동 어려움
```

### 3. 라이브러리 관리 불편

```
- 전역 설치만 됨
- 프로젝트별 버전 관리 안 됨
- 의존성 충돌
```

A 프로젝트는 라이브러리 v1.0 필요, B 프로젝트는 v2.0 필요.

Arduino IDE에선 답 없음.

### 4. 디버깅 불가

```
Serial.println("여기까지 옴1");
Serial.println("여기까지 옴2");
Serial.println("여기까지 옴3");
```

이게 디버깅의 전부.

브레이크포인트? 변수 값 확인? 꿈도 못 꿈.

---

## PlatformIO란

**Professional IDE for Embedded Development**

한 마디로: **VSCode에서 임베디드 개발하게 해주는 확장 프로그램**.

---

## 뭐가 좋은데?

### 1. VSCode의 모든 기능

```
- IntelliSense (자동완성)
- Go to Definition (함수 정의로 점프)
- 실시간 문법 검사
- Git 통합
- 수천 개의 확장 프로그램
```

코드 편집이 **쾌적**해진다.

### 2. 프로젝트 기반 관리

```
my_project/
├── src/
│   └── main.cpp
├── lib/
│   └── MyLibrary/
├── include/
│   └── config.h
├── test/
└── platformio.ini
```

폴더 구조 자유롭게. Git으로 버전 관리.

### 3. 라이브러리 격리

프로젝트마다 **독립적인 라이브러리**.

```ini
[env:uno]
lib_deps = 
    adafruit/Adafruit SSD1306 @ 2.5.7   ; 이 버전만 사용
```

A 프로젝트 v1.0, B 프로젝트 v2.0 **공존 가능**.

### 4. 멀티 보드 지원

하나의 코드로 여러 보드:

```ini
[env:uno]
platform = atmelavr
board = uno

[env:esp32]
platform = espressif32
board = esp32dev

[env:bluepill]
platform = ststm32
board = bluepill_f103c8
```

빌드할 때 보드만 선택.

### 5. 디버깅 지원

ST-Link, J-Link 연결하면:

```
- 브레이크포인트
- 변수 실시간 확인
- 스텝 실행
- 콜스택 추적
```

진짜 디버깅.

---

## 지원 플랫폼

1000개 이상의 보드 지원:

| 플랫폼 | 대표 보드 |
|--------|-----------|
| Atmel AVR | Arduino Uno, Mega, Nano |
| Espressif | ESP32, ESP8266 |
| ST STM32 | Bluepill, Nucleo, Discovery |
| Raspberry Pi | Pico, Pico W |
| Nordic | nRF52 시리즈 |
| NXP | Teensy, LPC |

Arduino, ESP-IDF, STM32 HAL, Zephyr 등 **프레임워크도 선택 가능**.

---

## Arduino IDE vs PlatformIO

| 항목 | Arduino IDE | PlatformIO |
|------|-------------|------------|
| 설치 | 쉬움 | 약간 복잡 |
| 학습 곡선 | 낮음 | 중간 |
| 코드 편집 | 기본 | 최고 |
| 자동완성 | 느림 | 빠름 |
| 프로젝트 관리 | 없음 | 있음 |
| 라이브러리 관리 | 전역 | 프로젝트별 |
| Git 연동 | 어려움 | 쉬움 |
| 디버깅 | 없음 | 있음 |
| 멀티 보드 | 불편 | 편리 |
| CLI 지원 | 없음 | 있음 |

---

## 언제 Arduino IDE?

- 처음 시작할 때
- 간단한 실험
- 빠르게 뭔가 돌려보고 싶을 때

## 언제 PlatformIO?

- 프로젝트가 커질 때
- 여러 파일로 나눠야 할 때
- 팀 작업
- 제대로 된 개발 환경 원할 때
- 디버깅 필요할 때

---

## 설치 전 알아둘 것

PlatformIO는 **VSCode 확장 프로그램**이다.

그래서:

1. VSCode 먼저 설치
2. PlatformIO 확장 설치
3. (자동으로) Python, 툴체인 등 설치됨

다음 글에서 VSCode 설치부터 시작.

---

[#2 - VSCode 설치](/posts/platformio-guide/platformio-guide-02/)
