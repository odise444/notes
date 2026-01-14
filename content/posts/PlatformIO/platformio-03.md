---
title: "PlatformIO 완벽 가이드 #3 - PlatformIO 확장 설치"
date: 2024-12-22
tags: ["PlatformIO", "VSCode", "확장"]
categories: ["개발환경"]
series: ["PlatformIO 완벽 가이드"]
summary: "확장 하나로 Arduino, ESP32, STM32 다 된다."
---

VSCode는 에디터일 뿐이다.

임베디드 개발하려면 PlatformIO 확장이 필요하다.

---

## PlatformIO란

임베디드 개발 플랫폼.

- **900개 이상 보드 지원** - Arduino, ESP32, STM32, Raspberry Pi Pico...
- **라이브러리 관리** - 한 줄로 설치
- **빌드 시스템** - 보드마다 자동 설정
- **디버거 통합** - GDB, OpenOCD
- **무료, 오픈소스**

---

## 확장 설치

### 방법 1: 검색으로 설치

1. VSCode 왼쪽 사이드바에서 🧩 (확장) 클릭
   - 또는 `Ctrl + Shift + X`

2. 검색창에 "PlatformIO" 입력

3. "PlatformIO IDE" 찾아서 **Install** 클릭

{{< figure src="/imgs/pio-extension-search.png" caption="PlatformIO IDE 확장" >}}

---

### 방법 2: 마켓플레이스에서 설치

1. https://marketplace.visualstudio.com/items?itemName=platformio.platformio-ide 접속

2. "Install" 클릭

3. VSCode 열기 허용

---

## 설치 중

처음 설치하면 시간이 좀 걸린다.

{{< figure src="/imgs/pio-installing.png" caption="PlatformIO 설치 중" >}}

뒤에서 설치되는 것들:
- Python 환경
- PlatformIO Core (CLI)
- 기본 도구들

하단 상태바에 진행 상황 표시.

**인내심 필요. 5~10분 걸릴 수 있음.**

---

## 설치 완료 확인

설치 끝나면:

1. 왼쪽 사이드바에 🏠 아이콘 (PlatformIO) 추가됨

2. 하단 상태바에 🏠 아이콘 표시

3. 클릭하면 PIO Home 열림

{{< figure src="/imgs/pio-home.png" caption="PIO Home 화면" >}}

---

## PIO Home 메뉴

### New Project
새 프로젝트 만들기

### Open Project
기존 프로젝트 열기

### Project Examples
예제 프로젝트들

### Libraries
라이브러리 검색/설치

### Boards
지원 보드 목록

### Platforms
플랫폼 관리 (Atmel AVR, Espressif 32 등)

### Devices
연결된 디바이스 확인

---

## 보드 지원 확인

ESP32 쓸 거면:

1. PIO Home → Platforms
2. "Embedded" 탭
3. "Espressif 32" 검색
4. Install

{{< figure src="/imgs/pio-platform-esp32.png" caption="ESP32 플랫폼 설치" >}}

처음 프로젝트 만들 때 자동 설치되니까 미리 안 해도 됨.

---

## CLI 확인 (선택)

터미널에서 PlatformIO 명령어 확인:

```bash
pio --version
```

```
PlatformIO Core, version 6.1.11
```

버전 나오면 OK.

안 되면 VSCode 재시작.

---

## 트러블슈팅

### "Installing PlatformIO Core..."가 너무 오래 걸림

- 인터넷 연결 확인
- 방화벽/VPN 확인
- VSCode 재시작 후 다시 시도

### Python 관련 에러

PlatformIO가 자체 Python 환경 쓰므로 보통 문제없음.

시스템 Python이 충돌하면:
1. 시스템 Python 환경변수에서 제거
2. VSCode 재시작

### 설치 후 아이콘 안 보임

1. VSCode 완전 종료 (트레이 아이콘까지)
2. 다시 실행

---

## 다음 단계

PlatformIO 준비 완료.

다음 글에서 PIO Home 둘러보기.

---

다음 글: [#4 - PIO Home 둘러보기](/posts/platformio/platformio-04/)
