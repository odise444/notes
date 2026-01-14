---
title: "PlatformIO 입문 #3 - PlatformIO 확장 설치"
date: 2024-12-22
tags: ["PlatformIO", "VSCode", "설치"]
categories: ["가이드"]
series: ["PlatformIO 입문"]
summary: "확장 프로그램 하나 설치하면 알아서 다 깔린다."
---

VSCode 준비됐으면 이제 PlatformIO.

확장 프로그램 하나만 설치하면 된다.

---

## 설치 방법

### 1. VSCode 실행

### 2. 확장 아이콘 클릭

왼쪽 사이드바에서 **네모 4개** 아이콘.

또는 단축키:
- Windows/Linux: `Ctrl + Shift + X`
- macOS: `Cmd + Shift + X`

### 3. 검색

검색창에 `PlatformIO` 입력.

### 4. 설치

**PlatformIO IDE** 선택 → **Install** 버튼.

제작자: PlatformIO

---

## 설치 중 (중요!)

**처음 설치할 때 시간이 오래 걸린다.**

PlatformIO가 백그라운드에서 이것들을 설치:

```
- Python (PlatformIO Core 실행용)
- PlatformIO Core (CLI)
- 기본 툴체인
```

### 상태 확인

VSCode 오른쪽 아래 보면:

```
PlatformIO: Installing PlatformIO Core...
```

이런 메시지 뜬다.

**5~15분 정도 기다려야 함.** 인터넷 속도에 따라 다름.

### 절대 하지 말 것

설치 중에:
- VSCode 끄지 않기
- 컴퓨터 끄지 않기
- 인터넷 끊지 않기

중간에 끊기면 꼬인다.

---

## 설치 완료 확인

### 1. PlatformIO 아이콘

왼쪽 사이드바에 **외계인 머리** 같은 아이콘 생김.

이게 PlatformIO 아이콘.

### 2. PlatformIO Home

아이콘 클릭하면 **PIO Home** 탭 열림.

또는:
1. 명령 팔레트 (`Ctrl + Shift + P`)
2. `PlatformIO: Home` 검색
3. 엔터

환영 화면 나오면 설치 성공!

### 3. 버전 확인

터미널에서:

```bash
pio --version
```

```
PlatformIO Core, version 6.x.x
```

---

## PIO Home 둘러보기

### Welcome 탭

- **New Project**: 새 프로젝트 생성
- **Import Arduino Project**: 기존 아두이노 프로젝트 변환
- **Open Project**: 기존 PlatformIO 프로젝트 열기

### Platforms 탭

설치된 플랫폼 목록.

처음엔 비어있음. 프로젝트 만들면 자동 설치됨.

### Boards 탭

지원하는 보드 검색.

1000개 넘는 보드 목록.

### Libraries 탭

라이브러리 검색, 설치.

Arduino Library Manager보다 강력함.

### Devices 탭

연결된 장치 (시리얼 포트) 확인.

---

## 설치 안 될 때

### 증상 1: Python 관련 에러

```
Python is not installed or not in PATH
```

해결:
1. Python 직접 설치 (python.org)
2. 설치 시 **Add to PATH** 체크
3. VSCode 재시작
4. PlatformIO 재설치

### 증상 2: 설치가 멈춤

30분 넘게 진행 안 되면:

1. VSCode 완전 종료
2. PlatformIO 폴더 삭제:
   - Windows: `C:\Users\사용자\.platformio`
   - macOS/Linux: `~/.platformio`
3. VSCode 다시 열기
4. PlatformIO 재설치

### 증상 3: 방화벽/프록시

회사나 학교 네트워크에서:

```
Connection refused
SSL Error
```

IT 부서에 문의하거나, 개인 네트워크에서 설치.

---

## 확장 프로그램 같이 설치되는 것들

PlatformIO IDE 설치하면 자동으로:

- **C/C++** (Microsoft): 이미 설치했으면 패스
- **PlatformIO IDE Terminal**: 통합 터미널

---

## 설정 (선택)

`Ctrl + ,` (설정) → `platformio` 검색.

### 자주 바꾸는 설정

```json
{
  // Home 자동 열기 끄기
  "platformio-ide.showHomeOnStartup": false,
  
  // 업로드 후 시리얼 모니터 자동 열기
  "platformio-ide.autoOpenSerialMonitor": true,
  
  // 빌드 전 저장
  "platformio-ide.autoSaveOnBuild": true
}
```

나중에 필요할 때 바꾸면 됨. 일단 기본값으로.

---

## 정리

설치 순서:

1. VSCode 설치 ✅
2. PlatformIO 확장 설치 ✅
3. 기다림 (Python, Core 설치) ✅
4. PIO Home 열리면 성공 ✅

이제 프로젝트 만들 준비 끝!

---

다음 글에서 첫 프로젝트 생성.

[#4 - 프로젝트 생성](/posts/platformio-guide/platformio-guide-04/)
