---
title: "PlatformIO 입문 #7 - 빌드와 업로드"
date: 2026-01-07
tags:
  - PlatformIO
  - 빌드
  - 업로드
  - LED
categories:
  - 가이드
series:
  - PlatformIO 입문
summary: 드디어 보드에 코드를 올려본다.
---

설정 끝났으면 이제 진짜 돌려보자.

LED 깜빡이기로 테스트.

---

## 준비물

- 보드 (ESP32, Arduino 등)
- USB 케이블
- 드라이버 (필요시)

---

## 드라이버 설치

### ESP32/ESP8266

CP210x 또는 CH340 칩 사용.

- **CP210x**: https://www.silabs.com/developers/usb-to-uart-bridge-vcp-drivers
- **CH340**: http://www.wch-ic.com/downloads/CH341SER_ZIP.html

Windows에서 자동 설치 안 되면 수동 설치.

### Arduino Uno/Mega

보통 Windows에서 자동 인식.

안 되면 Arduino IDE 설치하면 드라이버도 같이 설치됨.

### STM32 (ST-Link)

ST-Link 드라이버 필요.

https://www.st.com/en/development-tools/stsw-link009.html

---

## 보드 연결

USB 케이블로 컴퓨터에 연결.

### 포트 확인

**Windows:**
1. 장치 관리자 열기
2. 포트(COM & LPT) 확인
3. `USB-SERIAL CH340 (COM3)` 같은 거 보임

**macOS:**
```bash
ls /dev/cu.*
# /dev/cu.usbserial-0001
```

**Linux:**
```bash
ls /dev/ttyUSB*
# /dev/ttyUSB0
```

### PlatformIO에서 확인

하단 상태바 → 플러그 아이콘 클릭.

또는 PIO Home → Devices.

연결된 포트 목록 나옴.

---

## 코드 작성

`src/main.cpp`:

```cpp
#include <Arduino.h>

// ESP32는 보통 GPIO 2
// Arduino Uno는 13
#define LED_PIN 2

void setup() {
  Serial.begin(115200);
  pinMode(LED_PIN, OUTPUT);
  Serial.println("Blink Start!");
}

void loop() {
  digitalWrite(LED_PIN, HIGH);
  Serial.println("ON");
  delay(1000);
  
  digitalWrite(LED_PIN, LOW);
  Serial.println("OFF");
  delay(1000);
}
```

---

## 빌드

### 방법 1: 상태바 버튼

하단 상태바 → **✓ (체크 아이콘)** 클릭.

### 방법 2: 단축키

`Ctrl + Alt + B`

### 방법 3: 명령 팔레트

`Ctrl + Shift + P` → `PlatformIO: Build`

### 방법 4: 터미널

```bash
pio run
```

---

## 빌드 결과

터미널에 출력:

```
Processing esp32dev (platform: espressif32; board: esp32dev; framework: arduino)
--------------------------------------------------------------------------------
Verbose mode can be enabled via `-v, --verbose` option
CONFIGURATION: https://docs.platformio.org/page/boards/espressif32/esp32dev.html
PLATFORM: Espressif 32 (6.4.0) > Espressif ESP32 Dev Module
HARDWARE: ESP32 240MHz, 320KB RAM, 4MB Flash
...
Compiling .pio/build/esp32dev/src/main.cpp.o
Linking .pio/build/esp32dev/firmware.elf
Building .pio/build/esp32dev/firmware.bin
...
RAM:   [=         ]   5.0% (used 16432 bytes from 327680 bytes)
Flash: [==        ]  15.2% (used 199024 bytes from 1310720 bytes)
========================= [SUCCESS] Took 5.67 seconds =========================
```

**SUCCESS** 나오면 성공!

### 메모리 사용량

```
RAM:   [=         ]   5.0%
Flash: [==        ]  15.2%
```

- RAM: 변수, 힙, 스택
- Flash: 코드, 상수

---

## 업로드

### 방법 1: 상태바 버튼

하단 상태바 → **→ (화살표 아이콘)** 클릭.

### 방법 2: 단축키

`Ctrl + Alt + U`

### 방법 3: 명령 팔레트

`Ctrl + Shift + P` → `PlatformIO: Upload`

### 방법 4: 터미널

```bash
pio run -t upload
```

---

## 업로드 과정

```
Configuring upload protocol...
AVAILABLE: cmsis-dap, esp-bridge, esp-builtin, esp-prog, espota, esptool, iot-bus-jtag, jlink, minimodule, olimex-arm-usb-ocd, olimex-arm-usb-ocd-h, olimex-arm-usb-tiny-h, olimex-jtag-tiny, tumpa
CURRENT: upload_protocol = esptool
...
Uploading .pio/build/esp32dev/firmware.bin
esptool.py v4.6.2
Serial port COM3
Connecting....
Chip is ESP32-D0WDQ6 (revision v1.0)
...
Writing at 0x00010000... (100 %)
Wrote 199024 bytes (105748 compressed) at 0x00010000 in 2.5 seconds...
Hash of data verified.

Leaving...
Hard resetting via RTS pin...
========================= [SUCCESS] Took 12.34 seconds ========================
```

**SUCCESS** 나오면 업로드 완료!

---

## ESP32 업로드 팁

### 자동 업로드 모드 실패

```
Connecting........_____....._____
A fatal error occurred: Failed to connect to ESP32
```

**BOOT 버튼 누르기:**

1. 업로드 시작
2. `Connecting...` 나올 때
3. 보드의 **BOOT 버튼** 꾹 누름
4. 연결되면 손 뗌

### 업로드 속도 높이기

```ini
upload_speed = 921600
```

기본 460800보다 빠름.

---

## 업로드 후 확인

LED가 1초 간격으로 깜빡이면 성공!

---

## 빌드 + 업로드 한 번에

### 명령

```bash
pio run -t upload
```

빌드하고 바로 업로드.

### 상태바

보통 Upload 버튼이 빌드도 같이 함.

변경사항 있으면 자동 빌드 후 업로드.

---

## 클린 빌드

이상하면 한 번 지우고 다시:

```bash
pio run -t clean
pio run
```

또는 `.pio/build` 폴더 삭제.

---

## 빌드 에러 해결

### 에러: 포트 접근 불가

```
could not open port 'COM3': PermissionError
```

- 다른 프로그램이 포트 사용 중 (시리얼 모니터 등)
- 시리얼 모니터 닫고 다시 시도

### 에러: 보드 응답 없음

```
A fatal error occurred: Failed to connect
```

- USB 케이블 확인 (데이터 케이블인지)
- 드라이버 설치 확인
- 다른 USB 포트 시도
- ESP32: BOOT 버튼 누르기

---

축하! 첫 업로드 성공!

다음 글에서 시리얼 모니터.

[#8 - 시리얼 모니터](/posts/platformio-guide/platformio-guide-08/)
