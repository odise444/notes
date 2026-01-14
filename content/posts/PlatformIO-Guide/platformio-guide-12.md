---
title: "PlatformIO 입문 #12 - 디버깅"
date: 2026-01-12
tags:
  - PlatformIO
  - 디버깅
  - ST-Link
  - J-Link
categories:
  - 가이드
series:
  - PlatformIO 입문
summary: Serial.println() 졸업하고 진짜 디버깅하자.
---

`Serial.println("여기 옴")` 만으로 디버깅하던 시절.

이제 진짜 디버거 쓰자.

---

## 디버깅이란

- **브레이크포인트**: 특정 줄에서 멈춤
- **스텝 실행**: 한 줄씩 실행
- **변수 확인**: 현재 값 보기
- **콜스택**: 어떤 함수에서 왔는지

---

## 필요한 것

### 1. 디버그 프로브

| 프로브 | 지원 MCU | 가격 |
|--------|---------|------|
| ST-Link V2 | STM32 | $5~15 |
| J-Link | 대부분 | $300+ (EDU $60) |
| ESP-Prog | ESP32 | $15 |
| CMSIS-DAP | ARM Cortex | $10~30 |

ST-Link V2 클론이 제일 저렴.

### 2. 연결

**SWD 연결 (STM32):**

| 프로브 | 보드 |
|--------|------|
| SWDIO | SWDIO (PA13) |
| SWCLK | SWCLK (PA14) |
| GND | GND |
| 3.3V | 3.3V (선택) |

---

## 설정 (STM32 + ST-Link)

```ini
[env:bluepill]
platform = ststm32
board = bluepill_f103c8
framework = arduino

; 업로드도 ST-Link로
upload_protocol = stlink

; 디버그 설정
debug_tool = stlink
debug_build_flags = -O0 -g    ; 최적화 끄고 디버그 심볼 추가
```

### 중요: 최적화 끄기

```ini
debug_build_flags = -O0 -g
```

- `-O0`: 최적화 없음 (코드 순서 유지)
- `-g`: 디버그 심볼 포함

최적화 켜면 변수가 사라지거나 줄이 뒤섞임.

---

## 설정 (ESP32)

```ini
[env:esp32]
platform = espressif32
board = esp32dev
framework = arduino

debug_tool = esp-prog         ; 또는 esp-builtin
debug_init_break = tbreak app_main
```

ESP32는 JTAG 필요. ESP-Prog 또는 FT2232H 기반 프로브.

---

## 디버깅 시작

### 방법 1: 상태바

하단 상태바 → **🐞 (벌레 아이콘)** 클릭.

### 방법 2: 단축키

`F5`

### 방법 3: 명령 팔레트

`Ctrl + Shift + P` → `PlatformIO: Start Debugging`

---

## 디버그 화면

디버깅 시작하면 VSCode 레이아웃 변경:

### 왼쪽 패널

- **VARIABLES**: 현재 변수 값
- **WATCH**: 감시할 표현식
- **CALL STACK**: 함수 호출 스택
- **BREAKPOINTS**: 브레이크포인트 목록

### 상단 툴바

| 버튼 | 기능 | 단축키 |
|------|------|--------|
| ▶️ | Continue (계속) | F5 |
| ⏭️ | Step Over (다음 줄) | F10 |
| ⬇️ | Step Into (함수 안으로) | F11 |
| ⬆️ | Step Out (함수 밖으로) | Shift+F11 |
| 🔄 | Restart | Ctrl+Shift+F5 |
| ⏹️ | Stop | Shift+F5 |

---

## 브레이크포인트

### 설정

코드 줄 번호 왼쪽 클릭 → 빨간 점.

```cpp
void loop() {
  int value = analogRead(A0);   // ← 여기 브레이크포인트
  Serial.println(value);
  delay(1000);
}
```

### 조건부 브레이크포인트

빨간 점 우클릭 → "Edit Breakpoint"

```
조건: value > 500
```

value가 500 초과일 때만 멈춤.

### 로그포인트

멈추지 않고 로그만 출력:

빨간 점 우클릭 → "Edit Breakpoint" → "Log Message"

```
Value is {value}
```

---

## 변수 확인

### VARIABLES 패널

현재 스코프의 모든 변수 표시.

```
Locals
  value: 512
  i: 5
  buffer: char[32] "Hello"
```

### WATCH 패널

특정 표현식 감시:

```
buffer[0]       → 'H'
value * 2       → 1024
millis() / 1000 → 45
```

### 호버

코드에서 변수 위에 마우스 올리면 값 표시.

---

## 메모리 확인

### 레지스터

VARIABLES → Registers 섹션.

```
r0: 0x00000200
r1: 0x20001234
pc: 0x08001234
sp: 0x20010000
```

### 메모리 뷰

명령 팔레트 → "Debug: Memory"

특정 주소 영역 16진수로 보기.

---

## 하드폴트 디버깅

STM32에서 HardFault 나면:

1. 디버거로 실행
2. HardFault에서 멈춤
3. Call Stack 확인
4. 어디서 문제 났는지 추적

```
Call Stack:
  HardFault_Handler
  BusFault_Handler  
  myFunction         ← 여기서 문제
  main
```

---

## 주의사항

### 1. 디버그 빌드 크기

디버그 심볼 포함하면 바이너리 커짐.

```
Release: 50KB
Debug:   150KB
```

Flash 작은 MCU에선 안 들어갈 수 있음.

### 2. 타이밍

브레이크포인트에서 멈추면 시간 흐름 멈춤.

타이밍 크리티컬한 코드 (통신 프로토콜 등) 디버깅 어려움.

### 3. 인터럽트

인터럽트 핸들러에서 멈추면 시스템 먹통 될 수 있음.

---

## 팁

### 1. printf 디버깅 병행

```cpp
#ifdef DEBUG_MODE
  Serial.printf("Value: %d at line %d\n", value, __LINE__);
#endif
```

### 2. assert 활용

```cpp
#include <assert.h>

void process(int* ptr) {
  assert(ptr != NULL);  // NULL이면 여기서 멈춤
  *ptr = 100;
}
```

### 3. 조건부 컴파일

```cpp
#ifdef DEBUG
  // 디버깅용 코드
#endif
```

---

다음 글에서 유닛 테스트.

[#13 - 유닛 테스트](/posts/platformio-guide/platformio-guide-13/)
