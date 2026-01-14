---
title: "PlatformIO 완벽 가이드 #9 - 시리얼 모니터"
date: 2024-12-22
tags: ["PlatformIO", "시리얼", "모니터", "디버깅"]
categories: ["개발환경"]
series: ["PlatformIO 완벽 가이드"]
summary: "Serial.println() 출력 보기."
---

임베디드 디버깅의 기본, 시리얼 모니터.

`Serial.println()`으로 출력한 거 여기서 본다.

---

## 코드 준비

```cpp
#include <Arduino.h>

void setup() {
  Serial.begin(115200);  // 보드레이트 설정
  Serial.println("Hello, PlatformIO!");
}

void loop() {
  Serial.println(millis());
  delay(1000);
}
```

---

## 보드레이트 설정

`platformio.ini`에 추가:

```ini
[env:uno]
platform = atmelavr
board = uno
framework = arduino

monitor_speed = 115200
```

코드의 `Serial.begin()`과 같은 값으로.

---

## 시리얼 모니터 열기

### 방법 1: 하단 툴바

🔌 아이콘 클릭.

### 방법 2: 단축키

`Ctrl + Alt + S`

### 방법 3: 명령 팔레트

`Ctrl + Shift + P` → "PlatformIO: Serial Monitor"

---

## 시리얼 모니터 화면

{{< figure src="/imgs/pio-serial-monitor.png" caption="시리얼 모니터" >}}

```
--- Terminal on COM3 | 115200 8-N-1
--- Available filters and text transformations...
--- Quit: Ctrl+C | Menu: Ctrl+T | Help: Ctrl+T followed by Ctrl+H
Hello, PlatformIO!
1000
2000
3000
...
```

---

## 시리얼 모니터 옵션

### 보드레이트 변경

`platformio.ini`:

```ini
monitor_speed = 9600
```

### 필터 추가

```ini
monitor_filters = 
    colorize        ; 색상 추가
    time            ; 타임스탬프 추가
    printable       ; 출력 가능한 문자만
```

타임스탬프 예시:

```
[00:00:01.234] Hello, PlatformIO!
[00:00:02.234] 1000
```

### 라인 엔딩

```ini
monitor_eol = LF     ; \n만 (기본값)
monitor_eol = CRLF   ; \r\n (Windows 스타일)
```

---

## 데이터 보내기

시리얼 모니터에서 타이핑하면 보드로 전송됨.

```cpp
void loop() {
  if (Serial.available()) {
    char c = Serial.read();
    Serial.print("Received: ");
    Serial.println(c);
  }
}
```

모니터에서 `A` 입력 → Enter:

```
Received: A
```

---

## 단축키

모니터 실행 중:

| 키 | 동작 |
|-----|------|
| `Ctrl + C` | 종료 |
| `Ctrl + T` | 메뉴 |
| `Ctrl + T`, `Ctrl + H` | 도움말 |
| `Ctrl + T`, `Ctrl + R` | DTR 토글 (리셋) |

---

## 업로드하면서 모니터

시리얼 모니터 열려있으면 업로드 실패함.

왜? 포트 충돌.

해결:

### 방법 1: 수동

1. 모니터 닫기 (`Ctrl + C`)
2. 업로드
3. 모니터 다시 열기

### 방법 2: Upload and Monitor

```bash
pio run -t upload --target monitor
```

또는 `platformio.ini`:

```ini
targets = upload, monitor
```

업로드 후 자동으로 모니터 열림.

---

## 문제 해결

### 글자가 깨짐

```
ÿÿÿÿ???ÿÿÿ
```

→ 보드레이트 불일치. `monitor_speed` 확인.

### 모니터 안 열림

→ 포트 확인. Devices 탭에서 연결 확인.

### 아무것도 안 나옴

→ `Serial.begin()` 호출했는지 확인.
→ 코드에 `Serial.println()` 있는지 확인.

---

## 다음 단계

시리얼 모니터 완료!

다음 글에서 라이브러리 설치.

---

다음 글: [#10 - 라이브러리 설치](/posts/platformio/platformio-10/)
