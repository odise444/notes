---
title: "PlatformIO 완벽 가이드 #11 - IntelliSense 자동완성"
date: 2024-12-22
tags: ["PlatformIO", "IntelliSense", "자동완성", "VSCode"]
categories: ["개발환경"]
series: ["PlatformIO 완벽 가이드"]
summary: "타이핑 줄이고, 오타 줄이고."
---

Arduino IDE에서 제일 아쉬웠던 것.

**자동완성이 안 됨.**

PlatformIO는 된다.

---

## IntelliSense란

코드 작성 도우미:
- **자동완성** - 함수, 변수 이름 제안
- **문법 체크** - 에러 실시간 표시
- **정의 이동** - 함수 정의로 점프
- **호버 정보** - 마우스 올리면 설명

---

## 자동완성 사용

`digital` 까지 입력:

{{< figure src="/imgs/pio-autocomplete.png" caption="자동완성 목록" >}}

- `digitalWrite`
- `digitalRead`
- `digitalPinToInterrupt`

**Tab** 또는 **Enter**로 선택.

---

## 트리거

자동완성이 안 뜨면:

`Ctrl + Space` (수동 트리거)

---

## 파라미터 힌트

함수 괄호 열면 파라미터 표시:

```cpp
digitalWrite(
```

{{< figure src="/imgs/pio-param-hint.png" caption="파라미터 힌트" >}}

```
digitalWrite(uint8_t pin, uint8_t val): void
```

어떤 값 넣어야 하는지 바로 보임.

---

## 호버 정보

함수에 마우스 올리면:

{{< figure src="/imgs/pio-hover.png" caption="호버 정보" >}}

- 함수 시그니처
- 파라미터 설명
- 리턴 타입

---

## 정의로 이동

함수나 변수에서 `F12`:

해당 정의가 있는 파일/라인으로 점프.

`Ctrl + 클릭`도 됨.

예: `LED_BUILTIN`에서 F12

```cpp
// pins_arduino.h로 이동
#define LED_BUILTIN 13
```

---

## 참조 찾기

함수/변수가 어디서 쓰이는지:

`Shift + F12`

모든 참조 목록 표시.

---

## 에러 표시

틀린 코드:

```cpp
digitalWrite(LED_BUILTIN, HIHG);  // HIGH 오타
```

{{< figure src="/imgs/pio-error-squiggle.png" caption="에러 밑줄" >}}

빨간 밑줄 + 물결선.

마우스 올리면 에러 메시지:

```
identifier "HIHG" is undefined
```

**컴파일 전에 알 수 있음!**

---

## 경고 표시

경고는 노란색:

```cpp
int value;  // 초기화 안 됨
Serial.println(value);
```

```
variable 'value' is uninitialized when used here
```

---

## Problems 패널

모든 에러/경고 모아보기:

`Ctrl + Shift + M`

또는 하단 상태바에서 ⚠️ 클릭.

{{< figure src="/imgs/pio-problems.png" caption="Problems 패널" >}}

---

## IntelliSense 안 될 때

### 증상: 빨간 밑줄 많음

```cpp
#include <Arduino.h>  // 빨간 밑줄
```

### 해결 1: Rebuild IntelliSense

`Ctrl + Shift + P` → "PlatformIO: Rebuild IntelliSense Index"

### 해결 2: c_cpp_properties.json 확인

`.vscode/c_cpp_properties.json` 파일 확인.

없으면:
1. 프로젝트 닫기
2. `.vscode/` 폴더 삭제
3. 다시 열기 → 자동 생성

### 해결 3: 빌드 먼저

한 번 빌드하면 IntelliSense 경로 설정됨.

---

## 유용한 단축키

| 단축키 | 기능 |
|--------|------|
| `Ctrl + Space` | 자동완성 트리거 |
| `F12` | 정의로 이동 |
| `Alt + F12` | 정의 미리보기 |
| `Shift + F12` | 모든 참조 찾기 |
| `F2` | 이름 변경 (리팩토링) |
| `Ctrl + Shift + M` | Problems 패널 |

---

## 다음 단계

IntelliSense 활용법 완료!

다음 글에서 코드 포맷팅.

---

다음 글: [#12 - 코드 포맷팅](/posts/platformio/platformio-12/)
