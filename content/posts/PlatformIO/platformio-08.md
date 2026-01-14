---
title: "PlatformIO 완벽 가이드 #8 - 첫 빌드와 업로드"
date: 2024-12-22
tags: ["PlatformIO", "빌드", "업로드", "Arduino"]
categories: ["개발환경"]
series: ["PlatformIO 완벽 가이드"]
summary: "코드 컴파일하고 보드에 올려보자."
---

코드 작성 끝. 이제 보드에 올릴 차례.

---

## 보드 연결

USB 케이블로 보드를 PC에 연결.

드라이버는 Arduino IDE 쓸 때 설치했으면 그대로 쓰면 됨.

---

## 연결 확인

PIO Home → Devices 탭.

{{< figure src="/imgs/pio-devices-connected.png" caption="연결된 장치" >}}

```
COM3 - USB-SERIAL CH340 (COM3)
```

포트가 보이면 OK.

안 보이면:
- 케이블 확인 (데이터 케이블인지)
- 드라이버 확인
- USB 포트 바꿔보기

---

## 빌드하기

### 방법 1: 하단 툴바

하단 상태바에서 ✓ 클릭.

{{< figure src="/imgs/pio-toolbar.png" caption="PlatformIO 툴바" >}}

| 아이콘 | 기능 |
|--------|------|
| 🏠 | PIO Home |
| ✓ | Build (빌드) |
| → | Upload (업로드) |
| 🗑️ | Clean (클린) |
| 🔌 | Serial Monitor |
| 🔧 | Terminal |

### 방법 2: 단축키

`Ctrl + Alt + B`

### 방법 3: 명령 팔레트

`Ctrl + Shift + P` → "PlatformIO: Build" 입력

---

## 빌드 결과

터미널에 빌드 로그 출력.

```
Processing uno (platform: atmelavr; board: uno; framework: arduino)
-----------------------------------------------------------------
Verbose mode can be enabled via `-v, --verbose` option
CONFIGURATION: https://docs.platformio.org/...
PLATFORM: Atmel AVR (4.2.0) > Arduino Uno
...
Compiling .pio/build/uno/src/main.cpp.o
Linking .pio/build/uno/firmware.elf
Checking size .pio/build/uno/firmware.elf
Building .pio/build/uno/firmware.hex

RAM:   [          ]   0.4% (used 9 bytes from 2048 bytes)
Flash: [          ]   2.9% (used 924 bytes from 32256 bytes)
======== [SUCCESS] Took 2.34 seconds ========
```

**SUCCESS** 나오면 성공.

---

## 빌드 에러

에러 있으면:

```
src/main.cpp:5:3: error: 'HIHG' was not declared in this scope
     digitalWrite(LED_BUILTIN, HIHG);
     ^~~~
*** [.pio/build/uno/src/main.cpp.o] Error 1
```

- 파일명과 라인 번호 표시
- 클릭하면 해당 위치로 점프
- 에러 수정 후 다시 빌드

---

## 업로드하기

### 방법 1: 하단 툴바

→ (화살표) 클릭.

### 방법 2: 단축키

`Ctrl + Alt + U`

### 방법 3: 명령 팔레트

`Ctrl + Shift + P` → "PlatformIO: Upload"

---

## 업로드 결과

```
Configuring upload protocol...
AVAILABLE: arduino
CURRENT: upload_protocol = arduino
...
Uploading .pio/build/uno/firmware.hex
avrdude: AVR device initialized...
avrdude: Device signature = 0x1e950f (probably m328p)
avrdude: writing flash (924 bytes):
Writing | ################################# | 100% 0.15s
avrdude: 924 bytes of flash written
avrdude: verifying flash memory...
avrdude: 924 bytes of flash verified

======== [SUCCESS] Took 3.21 seconds ========
```

**SUCCESS** 나오면 완료.

보드에서 LED가 깜빡이기 시작!

---

## 업로드 에러

### 포트를 못 찾음

```
Error: Auto-detected: could not find a board connected...
```

해결:
1. 케이블 확인
2. Devices 탭에서 포트 확인
3. `platformio.ini`에 포트 직접 지정

```ini
upload_port = COM3
```

### 포트 사용 중

```
Error: Could not open port COM3
```

해결:
- 시리얼 모니터 닫기
- 다른 프로그램에서 포트 사용 중인지 확인

### 동기화 실패

```
avrdude: stk500_recv(): programmer is not responding
```

해결:
- 보드 리셋 버튼 누르면서 업로드
- 업로드 속도 낮추기

```ini
upload_speed = 57600
```

---

## 빌드만 vs 업로드

| 명령 | 동작 |
|------|------|
| Build (✓) | 컴파일만, 보드에 안 올림 |
| Upload (→) | 컴파일 + 보드에 올림 |

코드 확인만 할 때는 Build.

실제로 테스트할 때는 Upload.

---

## Clean

빌드 결과물 삭제.

```
🗑️ Clean
```

`.pio/build/` 폴더 내용 삭제됨.

이상하게 안 되면 Clean 후 다시 Build.

---

## 다음 단계

빌드와 업로드 완료!

다음 글에서 시리얼 모니터.

---

다음 글: [#9 - 시리얼 모니터](/posts/platformio/platformio-09/)
