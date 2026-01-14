---
title: "PlatformIO 완벽 가이드 #17 - 디버거 연결 (ST-Link)"
date: 2024-12-22
tags: ["PlatformIO", "디버깅", "ST-Link", "GDB"]
categories: ["개발환경"]
series: ["PlatformIO 완벽 가이드"]
summary: "Serial.println() 졸업하고 진짜 디버깅하기."
---

`Serial.println()`으로 디버깅하는 건 한계가 있다.

진짜 디버거로 브레이크포인트 걸고 변수 확인해보자.

---

## 준비물

- **디버거**: ST-Link V2, J-Link, CMSIS-DAP 등
- **보드**: 디버그 지원 보드 (STM32, ESP32 등)

Arduino Uno는 디버깅 안 됨 (디버그 인터페이스 없음).

---

## STM32 + ST-Link 예시

Blue Pill (STM32F103C8) + ST-Link V2.

### 연결

```
ST-Link V2      Blue Pill
---------       ---------
SWCLK     -->   SWCLK
SWDIO     -->   SWDIO
GND       -->   GND
3.3V      -->   3.3V (또는 외부 전원)
```

---

## platformio.ini 설정

```ini
[env:bluepill_debug]
platform = ststm32
board = bluepill_f103c8
framework = arduino

; 업로드 프로토콜
upload_protocol = stlink

; 디버그 설정
debug_tool = stlink
debug_init_break = tbreak main
```

---

## debug_tool 옵션

| 디버거 | 값 |
|--------|-----|
| ST-Link | `stlink` |
| J-Link | `jlink` |
| Black Magic Probe | `blackmagic` |
| CMSIS-DAP | `cmsis-dap` |

---

## 디버깅 시작

### 방법 1: 사이드바

왼쪽 ▶️ (Run and Debug) 클릭 → 초록색 ▶️ 클릭

### 방법 2: 단축키

`F5`

### 방법 3: 명령 팔레트

`Ctrl + Shift + P` → "PlatformIO: Start Debugging"

---

## 디버그 화면

{{< figure src="/imgs/pio-debug-view.png" caption="디버그 화면" >}}

화면 구성:
- **Variables**: 변수 값
- **Watch**: 감시할 변수/표현식
- **Call Stack**: 호출 스택
- **Breakpoints**: 브레이크포인트 목록

상단 툴바:
- ▶️ Continue (F5)
- ⏭️ Step Over (F10)
- ⬇️ Step Into (F11)
- ⬆️ Step Out (Shift+F11)
- 🔄 Restart
- ⏹️ Stop

---

## 첫 디버깅

1. `main.cpp` 열기
2. 라인 번호 왼쪽 클릭 → 빨간 점 (브레이크포인트)
3. F5로 디버깅 시작
4. 브레이크포인트에서 멈춤
5. F10으로 한 줄씩 실행

---

## ESP32 디버깅

ESP32도 JTAG 디버깅 가능.

ESP-PROG 또는 ESP32-S3 내장 USB JTAG.

```ini
[env:esp32_debug]
platform = espressif32
board = esp32dev
framework = arduino

debug_tool = esp-prog
debug_init_break = tbreak setup
```

---

## debug_init_break

디버깅 시작 시 어디서 멈출지:

```ini
debug_init_break = tbreak main      ; main()에서
debug_init_break = tbreak setup     ; setup()에서
debug_init_break = tbreak loop      ; loop()에서
debug_init_break =                  ; 안 멈춤 (빈 값)
```

---

## 디버그 빌드

디버깅하려면 디버그 심볼 필요:

```ini
build_type = debug

; 또는 수동으로
build_flags =
    -Og       ; 디버그 최적화
    -g3       ; 최대 디버그 정보
```

`-O2`나 `-Os` 최적화는 디버깅 어려움 (변수 최적화로 사라짐).

---

## 원격 디버깅 (선택)

```ini
debug_server =
    /path/to/openocd
    -f interface/stlink.cfg
    -f target/stm32f1x.cfg
    
debug_port = localhost:3333
```

GDB 서버 직접 설정 가능.

---

## 트러블슈팅

### "Target not found"

- ST-Link 드라이버 설치 확인
- 연결 상태 확인 (SWDIO, SWCLK)
- ST-Link 펌웨어 업데이트

### "Failed to connect"

- 보드 전원 확인
- BOOT0 핀 확인 (GND여야 함)
- 리셋 후 다시 시도

### 변수 값이 <optimized out>

- `build_type = debug` 설정
- `-O0` 또는 `-Og` 최적화 사용

---

## 다음 단계

디버거 연결 완료!

다음 글에서 브레이크포인트 활용.

---

다음 글: [#18 - 브레이크포인트](/posts/platformio/platformio-18/)
