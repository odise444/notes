---
title: "PlatformIO 완벽 가이드 #1 - Arduino IDE의 한계"
date: 2024-12-22
tags: ["PlatformIO", "Arduino", "VSCode", "임베디드"]
categories: ["개발환경"]
series: ["PlatformIO 완벽 가이드"]
summary: "Arduino IDE로 시작했지만, 이제 졸업할 때가 됐다."
---

Arduino로 LED 깜빡이기 성공했다.

센서도 붙여보고, 모터도 돌려봤다.

근데 코드가 길어지니까 뭔가 불편하다.

---

## Arduino IDE의 좋은 점

처음엔 최고다.

- **설치 간단** - 다운로드, 설치, 끝
- **보드 선택 쉬움** - 드롭다운에서 고르기
- **업로드 한 방** - 버튼 하나로 끝
- **예제 많음** - 메뉴에서 바로 열기

입문자에게 완벽하다.

---

## 근데 코드가 길어지면

### 1. 자동완성이 없다

```cpp
Serial.prin
```

여기서 Tab 눌러도 아무 일도 안 일어난다.

`println`인지 `print`인지 직접 다 쳐야 함.

함수 이름 길면? 오타 나면? 컴파일해봐야 안다.

---

### 2. 에러 메시지가 불친절

```
error: 'sensors' was not declared in this scope
```

어디서 틀렸는지 찾아야 한다.

라인 번호 클릭해도 점프 안 됨.

---

### 3. 파일 여러 개면 복잡

프로젝트 커지면 파일 분리하고 싶다.

```
MyProject/
├── MyProject.ino
├── sensors.h
├── sensors.cpp
├── motors.h
└── motors.cpp
```

Arduino IDE에서 탭으로 보이긴 하는데... 불편하다.

---

### 4. 라이브러리 관리

라이브러리 매니저로 설치는 되는데:

- 버전 관리 어려움
- 프로젝트마다 다른 버전 쓰기 힘듦
- 어떤 라이브러리 썼는지 기록 안 남음

팀 프로젝트? 다른 사람이 열면 라이브러리 없어서 에러.

---

### 5. 디버깅 불가

```cpp
Serial.println("여기까지 옴");
Serial.println("변수 값: " + String(value));
```

이게 디버깅의 전부다.

브레이크포인트? 변수 워치? 없다.

---

### 6. 여러 보드 동시 개발

ESP32랑 Arduino Uno 동시에 개발하면?

보드 바꿀 때마다:
1. 도구 → 보드 선택
2. 도구 → 포트 선택
3. 빌드
4. 업로드

반복. 귀찮다.

---

## 프로가 되려면

IDE를 바꿔야 한다.

**PlatformIO**

- VSCode 기반
- 자동완성, 문법 체크
- 라이브러리 버전 관리
- 멀티 보드 지원
- 디버깅 가능
- 무료

---

## 걱정되는 것들

### "VSCode 어렵지 않아요?"

처음엔 낯설다. 근데 금방 익숙해진다.

Arduino IDE보다 버튼이 많아 보이지만, 쓰는 건 몇 개 안 됨.

### "Arduino 코드 그대로 되나요?"

된다. `.ino` 파일 그대로 가져와도 됨.

`setup()`, `loop()` 그대로.

### "보드 드라이버는요?"

Arduino IDE에서 쓰던 드라이버 그대로.

CH340, CP2102 다 됨.

---

## 이 시리즈에서 배울 것

1. VSCode 설치
2. PlatformIO 확장 설치
3. 프로젝트 생성
4. 빌드 & 업로드
5. 시리얼 모니터
6. 라이브러리 관리
7. 멀티 보드 설정
8. 디버깅

Arduino IDE 쓰던 사람 기준으로 설명한다.

---

다음 글에서 VSCode 설치.

[#2 - VSCode 설치](/posts/platformio/platformio-02/)
