---
title: "PlatformIO 입문 #15 - 트러블슈팅"
date: 2026-01-14
tags: ["PlatformIO", "트러블슈팅", "에러해결"]
categories: ["가이드"]
series: ["PlatformIO 입문"]
summary: "PlatformIO 쓰다 만나는 흔한 에러들."
---

PlatformIO 쓰다 보면 에러 만난다.

흔한 문제와 해결법 정리.

---

## 설치 관련

### Python 관련 에러

```
Python is not installed or not in PATH
```

**해결:**
1. Python 3.x 설치 (python.org)
2. 설치 시 "Add to PATH" 체크
3. 터미널 재시작
4. `python --version` 확인

### PlatformIO Core 설치 멈춤

30분 넘게 진행 안 됨.

**해결:**
1. VSCode 완전 종료
2. 폴더 삭제:
   - Windows: `C:\Users\사용자\.platformio`
   - macOS/Linux: `~/.platformio`
3. VSCode 다시 시작
4. PlatformIO 재설치

### 방화벽/프록시 문제

```
Connection refused
SSL Error
```

**해결:**
- 개인 네트워크에서 설치
- 프록시 설정:
  ```bash
  export HTTP_PROXY=http://proxy:port
  export HTTPS_PROXY=http://proxy:port
  ```

---

## 빌드 에러

### Arduino.h 없음

```
fatal error: Arduino.h: No such file or directory
```

**원인:** `#include <Arduino.h>` 빠짐.

**해결:**
```cpp
#include <Arduino.h>  // 맨 위에 추가
```

### 라이브러리 못 찾음

```
fatal error: SomeLibrary.h: No such file or directory
```

**해결:**
1. `platformio.ini`에 라이브러리 추가:
   ```ini
   lib_deps = 
       SomeLibrary
   ```
2. 라이브러리 이름 정확히 확인 (대소문자)

### 함수 중복 정의

```
multiple definition of `setup'
```

**원인:** `src/`에 `.ino` 파일 있음.

**해결:**
- `.ino` 파일을 `.cpp`로 변경
- 또는 `.ino` 파일 삭제

### 메모리 초과

```
region `FLASH' overflowed by 1234 bytes
```

**해결:**
- 불필요한 라이브러리 제거
- 코드 최적화
- 더 큰 Flash 보드 사용
- 문자열을 PROGMEM으로:
  ```cpp
  const char msg[] PROGMEM = "Hello";
  ```

---

## 업로드 에러

### 포트 못 찾음

```
Auto-detected: None
Error: Please specify `upload_port`
```

**해결:**
1. USB 케이블 확인 (데이터 케이블인지)
2. 드라이버 설치 확인
3. 다른 USB 포트 시도
4. 직접 포트 지정:
   ```ini
   upload_port = COM3  ; Windows
   upload_port = /dev/ttyUSB0  ; Linux
   ```

### 포트 사용 중

```
could not open port 'COM3': PermissionError
```

**해결:**
- 시리얼 모니터 닫기
- 다른 프로그램 닫기 (Arduino IDE 등)
- Linux: 권한 추가
  ```bash
  sudo usermod -a -G dialout $USER
  # 로그아웃 후 다시 로그인
  ```

### ESP32 연결 실패

```
A fatal error occurred: Failed to connect to ESP32
```

**해결:**
1. BOOT 버튼 누른 상태로 업로드
2. USB 케이블 교체
3. `upload_speed` 낮추기:
   ```ini
   upload_speed = 115200
   ```

### ST-Link 인식 안 됨

```
Error: libusb_open() failed with LIBUSB_ERROR_NOT_SUPPORTED
```

**해결 (Windows):**
1. Zadig 다운로드 (zadig.akeo.ie)
2. Options → List All Devices
3. ST-Link 선택
4. WinUSB 드라이버 설치

---

## IntelliSense 문제

### 빨간 밑줄 투성이

코드는 빌드되는데 에디터에서 에러 표시.

**해결:**
1. 명령 팔레트 → "PlatformIO: Rebuild IntelliSense Index"
2. VSCode 재시작
3. `.vscode/c_cpp_properties.json` 삭제 후 재생성

### 자동완성 안 됨

**해결:**
1. C/C++ 확장 설치 확인
2. 명령 팔레트 → "C/C++: Reset IntelliSense Database"
3. `platformio.ini` 저장 (인덱스 재생성 트리거)

---

## 시리얼 모니터

### 글자 깨짐

```
ÿÿÿ?ÿñ
```

**원인:** 속도 불일치.

**해결:**
```ini
monitor_speed = 115200  ; 코드와 맞추기
```

### 아무것도 안 나옴

**체크리스트:**
1. `Serial.begin()` 있는지
2. `delay(1000)` 추가 (연결 안정화)
3. 올바른 포트인지
4. USB 케이블 확인

### 한글 깨짐

**해결:**
```ini
monitor_filters = 
    direct
    printable
```

또는 터미널 인코딩 UTF-8로 설정.

---

## 기타

### 빌드 느림

**해결:**
1. 바이러스 백신 예외 추가 (프로젝트 폴더, `.platformio` 폴더)
2. SSD 사용
3. 라이브러리 정리

### 클린 빌드

뭔가 이상하면:

```bash
pio run -t clean
pio run
```

또는 `.pio/build` 폴더 삭제.

### 플랫폼 재설치

```bash
pio platform uninstall espressif32
pio platform install espressif32
```

### 전체 초기화

핵옵션:

```bash
# 모든 플랫폼, 툴체인 삭제
rm -rf ~/.platformio

# VSCode에서 PlatformIO 재설치
```

---

## 도움 받기

### 공식 문서

https://docs.platformio.org

### 커뮤니티

https://community.platformio.org

### GitHub Issues

https://github.com/platformio/platformio-core/issues

에러 메시지로 검색하면 대부분 답 나옴.

---

## 시리즈 끝

15개 글에 걸쳐 PlatformIO 입문을 다뤘다.

정리:
- VSCode + PlatformIO 설치
- 프로젝트 생성과 구조
- platformio.ini 설정
- 빌드, 업로드, 시리얼 모니터
- 라이브러리 관리
- 멀티 환경
- 디버깅
- 유닛 테스트
- CI/CD

이제 Arduino IDE 졸업!

---

[#1 - PlatformIO란?](/posts/platformio-guide/platformio-guide-01/)로 돌아가기
