---
title: "PlatformIO 완벽 가이드 #4 - PIO Home 둘러보기"
date: 2024-12-22
tags: ["PlatformIO", "VSCode", "PIO Home"]
categories: ["개발환경"]
series: ["PlatformIO 완벽 가이드"]
summary: "PlatformIO의 대시보드, PIO Home 살펴보기."
---

PIO Home은 PlatformIO의 대시보드다.

프로젝트 만들고, 라이브러리 찾고, 보드 확인하고.

---

## PIO Home 열기

방법 1: 왼쪽 사이드바 🏠 아이콘 클릭

방법 2: 하단 상태바 🏠 클릭

방법 3: `Ctrl + Shift + P` → "PlatformIO: Home" 입력

{{< figure src="/imgs/pio-home-overview.png" caption="PIO Home 메인 화면" >}}

---

## 메인 화면

### Quick Access

자주 쓰는 기능 바로가기:
- **New Project** - 새 프로젝트
- **Open Project** - 기존 프로젝트
- **Project Examples** - 예제
- **Import Arduino Project** - Arduino 프로젝트 가져오기

### Recent Projects

최근 열었던 프로젝트 목록.

클릭하면 바로 열림.

### PlatformIO News

업데이트 소식, 팁 등.

---

## Projects 탭

{{< figure src="/imgs/pio-home-projects.png" caption="프로젝트 목록" >}}

내 프로젝트들 모아보기.

- 프로젝트 이름
- 경로
- 보드 정보
- 마지막 수정 시간

**Add Existing** 버튼으로 기존 프로젝트 추가 가능.

---

## Libraries 탭

{{< figure src="/imgs/pio-home-libraries.png" caption="라이브러리 검색" >}}

**Registry** - 공식 라이브러리 검색

```
검색 예: "DHT sensor"
```

라이브러리 클릭하면:
- 설명
- 예제 코드
- 호환 보드
- 의존성
- 버전 목록

**Installed** - 설치된 라이브러리 확인

---

## Boards 탭

{{< figure src="/imgs/pio-home-boards.png" caption="보드 검색" >}}

900개 이상 보드 검색.

```
검색 예: "ESP32"
```

보드 클릭하면:
- 스펙 (CPU, Flash, RAM)
- 지원 프레임워크
- 업로드 방법
- 디버그 지원 여부

**platformio.ini에 쓸 보드 이름 확인할 때 유용.**

---

## Platforms 탭

{{< figure src="/imgs/pio-home-platforms.png" caption="플랫폼 관리" >}}

플랫폼 = 보드 계열 + 툴체인.

**Embedded** 탭:
- Atmel AVR (Arduino Uno 등)
- Espressif 32 (ESP32)
- ST STM32 (STM32 시리즈)
- Raspberry Pi RP2040 (Pico)

**Installed** 탭:
- 설치된 플랫폼 버전 확인
- 업데이트, 제거

---

## Devices 탭

{{< figure src="/imgs/pio-home-devices.png" caption="연결된 장치" >}}

USB로 연결된 보드 목록.

- 포트 (COM3, /dev/ttyUSB0 등)
- 설명
- 하드웨어 ID (VID/PID)

보드 연결 확인할 때 유용.

**Refresh** 버튼으로 새로고침.

---

## Inspect 탭

프로젝트 분석 도구.

- 코드 크기 분석
- 메모리 사용량
- 의존성 트리

빌드 후 확인 가능.

---

## 설정

오른쪽 상단 ⚙️ 아이콘.

### PIO Account

- 회원가입하면 설정 동기화
- 필수 아님

### Settings

- 기본 프로젝트 경로
- 텔레메트리 (사용 통계)
- 업데이트 알림

---

## 자주 쓰는 기능

| 할 일 | 메뉴 |
|-------|------|
| 새 프로젝트 | Home → New Project |
| 라이브러리 찾기 | Libraries → Registry |
| 보드 이름 찾기 | Boards → 검색 |
| 포트 확인 | Devices |
| 플랫폼 업데이트 | Platforms → Updates |

---

## 다음 단계

PIO Home 파악 완료.

다음 글에서 실제로 프로젝트 만들어보자.

---

다음 글: [#5 - 새 프로젝트 만들기](/posts/platformio/platformio-05/)
