---
title: "PlatformIO 입문 #8 - 시리얼 모니터"
date: 2024-12-22
tags: ["PlatformIO", "시리얼", "디버깅", "UART"]
categories: ["가이드"]
series: ["PlatformIO 입문"]
summary: "Serial.println()으로 디버깅하는 기본기."
---

LED 깜빡이기 됐으면 다음은 시리얼 통신.

`Serial.println()`으로 보드가 뭘 하는지 볼 수 있다.

---

## 코드에 시리얼 추가

```cpp
#include <Arduino.h>

void setup() {
  Serial.begin(115200);  // 속도 설정
  delay(1000);           // 연결 대기
  Serial.println("=== Program Start ===");
}

void loop() {
  Serial.print("Uptime: ");
  Serial.print(millis() / 1000);
  Serial.println(" seconds");
  delay(1000);
}
```

---

## 시리얼 모니터 열기

### 방법 1: 상태바

하단 상태바 → **🔌 (플러그 아이콘)** 클릭.

### 방법 2: 단축키

없음. 상태바나 명령 팔레트 사용.

### 방법 3: 명령 팔레트

`Ctrl + Shift + P` → `PlatformIO: Serial Monitor`

### 방법 4: 터미널

```bash
pio device monitor
```

---

## 시리얼 모니터 화면

```
--- Terminal on COM3 | 115200 8-N-1
--- Available filters and text transformations: colorize, debug, default, direct, esp32_exception_decoder, hexlify, log2file, nocontrol, printable, send_on_enter, time
--- More details at https://bit.ly/pio-monitor-filters
--- Quit: Ctrl+C | Menu: Ctrl+T | Help: Ctrl+T followed by Ctrl+H
=== Program Start ===
Uptime: 0 seconds
Uptime: 1 seconds
Uptime: 2 seconds
...
```

---

## 설정

### 속도 (Baud Rate)

**platformio.ini:**

```ini
monitor_speed = 115200
```

코드의 `Serial.begin()`과 **반드시 일치**해야 함.

```cpp
Serial.begin(115200);  // 코드
```

```ini
monitor_speed = 115200  ; 설정
```

불일치하면 깨진 글자 나옴:

```
�����ㄴ쀄ㅇ뢾
```

### 자주 쓰는 속도

| 속도 | 용도 |
|------|------|
| 9600 | Arduino 기본 |
| 115200 | ESP32 기본, 범용 |
| 921600 | 고속 로깅 |

---

## 필터

유용한 필터들:

### 타임스탬프

```ini
monitor_filters = time
```

출력:

```
[00:00:01.234] Uptime: 1 seconds
[00:00:02.235] Uptime: 2 seconds
```

### ESP32 예외 디코더

```ini
monitor_filters = esp32_exception_decoder
```

크래시 났을 때 스택 트레이스 읽기 쉽게 변환.

### 여러 필터 조합

```ini
monitor_filters = 
    time
    colorize
    esp32_exception_decoder
```

---

## 시리얼 입력

### 보내기

모니터 창 아래 입력 가능.

텍스트 입력 → Enter.

### 코드에서 받기

```cpp
void loop() {
  if (Serial.available()) {
    String input = Serial.readStringUntil('\n');
    Serial.print("받은 메시지: ");
    Serial.println(input);
  }
}
```

---

## 모니터 명령어

모니터 실행 중:

| 키 | 기능 |
|----|------|
| `Ctrl + C` | 종료 |
| `Ctrl + T` | 메뉴 |
| `Ctrl + T, Ctrl + H` | 도움말 |
| `Ctrl + T, Ctrl + R` | 재연결 |

---

## 업로드 후 자동 열기

```ini
; 업로드 후 자동으로 모니터 열기
monitor_filters = send_on_enter
```

또는 VSCode 설정:

```json
"platformio-ide.autoOpenSerialMonitor": true
```

---

## 문제 해결

### 글자 깨짐

```
ÿÿÿ?ÿñ
```

원인: 속도 불일치.

해결: `monitor_speed`와 `Serial.begin()` 맞추기.

### 아무것도 안 나옴

1. 코드에 `Serial.begin()` 있는지 확인
2. `delay(1000)` 추가 (연결 안정화)
3. 올바른 포트인지 확인
4. USB 케이블 데이터 지원하는지 확인

### 포트 사용 중

```
could not open port 'COM3'
```

다른 프로그램이 포트 사용 중.

- Arduino IDE 시리얼 모니터 닫기
- 다른 터미널 닫기

---

## 여러 시리얼 포트

### 코드 (ESP32 예시)

```cpp
// 기본 USB 시리얼
Serial.begin(115200);

// 하드웨어 시리얼 2
Serial2.begin(9600, SERIAL_8N1, 16, 17);  // RX=16, TX=17
```

### 모니터링

다른 포트 모니터:

```bash
pio device monitor -p COM4 -b 9600
```

---

## 유용한 디버깅 패턴

### 함수 진입/종료

```cpp
void myFunction() {
  Serial.println(">>> myFunction() 진입");
  
  // ... 코드 ...
  
  Serial.println("<<< myFunction() 종료");
}
```

### 변수 값 출력

```cpp
int sensorValue = analogRead(A0);
Serial.print("센서 값: ");
Serial.println(sensorValue);
```

### 조건부 출력

```cpp
#define DEBUG 1

#if DEBUG
  #define DEBUG_PRINT(x) Serial.println(x)
#else
  #define DEBUG_PRINT(x)
#endif

void loop() {
  DEBUG_PRINT("디버그 모드에서만 출력");
}
```

---

시리얼 모니터는 임베디드 개발의 기본 디버깅 도구!

다음 글에서 라이브러리 관리.

[#9 - 라이브러리 관리](/posts/platformio-guide/platformio-guide-09/)
