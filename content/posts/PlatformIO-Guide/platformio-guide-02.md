---
title: "PlatformIO 입문 #2 - VSCode 설치"
date: 2026-01-02
tags:
  - PlatformIO
  - VSCode
  - 설치
categories:
  - 가이드
series:
  - PlatformIO 입문
summary: 일단 VSCode부터 설치하자.
---

PlatformIO는 VSCode 확장 프로그램이다.

먼저 VSCode를 설치해야 한다.

---

## VSCode란

**Visual Studio Code**

마이크로소프트가 만든 **무료** 코드 편집기.

Visual Studio랑 다르다:
- Visual Studio: 무겁고 큰 IDE (수 GB)
- VSCode: 가볍고 빠른 편집기 (수백 MB)

---

## 다운로드

공식 사이트: https://code.visualstudio.com/

운영체제에 맞는 버전 다운로드.

---

## Windows 설치

### 1. 다운로드

`VSCodeUserSetup-x64-x.xx.x.exe` 다운로드.

### 2. 실행

다운받은 파일 더블클릭.

### 3. 라이선스 동의

"동의합니다" 체크 → 다음.

### 4. 설치 위치

기본값 그대로 → 다음.

### 5. 추가 작업 선택

**중요!** 이것들 체크:

```
☑ 바탕 화면에 바로 가기 만들기
☑ "Code(으)로 열기" 작업을 Windows 탐색기 파일 상황에 맞는 메뉴에 추가
☑ "Code(으)로 열기" 작업을 Windows 탐색기 디렉터리 상황에 맞는 메뉴에 추가
☑ PATH에 추가
```

특히 **PATH에 추가**는 꼭 체크.

나중에 터미널에서 `code .` 명령 쓸 수 있다.

### 6. 설치

"설치" 클릭 → 기다림 → 완료.

---

## macOS 설치

### 1. 다운로드

`VSCode-darwin-universal.zip` 또는 `VSCode-darwin-arm64.zip` (M1/M2).

### 2. 압축 해제

다운받은 zip 파일 더블클릭.

### 3. 응용 프로그램으로 이동

`Visual Studio Code.app`을 **응용 프로그램** 폴더로 드래그.

### 4. 처음 실행

응용 프로그램에서 VSCode 실행.

"개발자를 확인할 수 없습니다" 뜨면:

시스템 환경설정 → 보안 및 개인정보 → "확인 없이 열기"

### 5. PATH 설정

VSCode 열고:

1. `Cmd + Shift + P` (명령 팔레트)
2. `shell command` 검색
3. "Shell Command: Install 'code' command in PATH" 선택

이제 터미널에서 `code .` 사용 가능.

---

## Linux 설치

### Ubuntu/Debian

```bash
# 패키지 저장소 추가
wget -qO- https://packages.microsoft.com/keys/microsoft.asc | gpg --dearmor > packages.microsoft.gpg
sudo install -o root -g root -m 644 packages.microsoft.gpg /etc/apt/trusted.gpg.d/
sudo sh -c 'echo "deb [arch=amd64] https://packages.microsoft.com/repos/code stable main" > /etc/apt/sources.list.d/vscode.list'

# 설치
sudo apt update
sudo apt install code
```

### Fedora/RHEL

```bash
sudo rpm --import https://packages.microsoft.com/keys/microsoft.asc
sudo sh -c 'echo -e "[code]\nname=Visual Studio Code\nbaseurl=https://packages.microsoft.com/yumrepos/vscode\nenabled=1\ngpgcheck=1\ngpgkey=https://packages.microsoft.com/keys/microsoft.asc" > /etc/yum.repos.d/vscode.repo'
sudo dnf install code
```

### Snap

```bash
sudo snap install code --classic
```

---

## 첫 실행

VSCode 실행하면 환영 화면 나옴.

### 테마 선택

어두운 테마 좋아하면: **Dark+** (기본)
밝은 테마 좋아하면: **Light+**

나중에 바꿀 수 있으니 일단 넘어가도 됨.

### 언어 설정

한국어로 바꾸고 싶으면:

1. 왼쪽 사이드바 → 확장 아이콘 (네모 4개)
2. `Korean` 검색
3. "Korean Language Pack for Visual Studio Code" 설치
4. VSCode 재시작

---

## 기본 조작

### 명령 팔레트

**가장 중요한 단축키**:

- Windows/Linux: `Ctrl + Shift + P`
- macOS: `Cmd + Shift + P`

모든 명령을 여기서 검색해서 실행.

### 파일 열기

- `Ctrl + O` (Windows/Linux)
- `Cmd + O` (macOS)

### 폴더 열기

- `Ctrl + K, Ctrl + O` (Windows/Linux)
- `Cmd + K, Cmd + O` (macOS)

또는 메뉴 → 파일 → 폴더 열기

### 터미널 열기

- `` Ctrl + ` `` (백틱, 숫자 1 왼쪽 키)

VSCode 안에서 터미널 사용 가능.

---

## 필수 확장 프로그램

PlatformIO 전에 이것들 먼저 설치하면 좋다:

### 1. C/C++ (Microsoft)

C/C++ 지원. IntelliSense, 디버깅.

확장 검색 → `C/C++` → Microsoft 꺼 설치.

### 2. GitLens (선택)

Git 히스토리, blame 보기 편함.

---

## 설치 확인

터미널에서:

```bash
code --version
```

버전 나오면 성공.

```
1.85.0
...
```

---

다음 글에서 PlatformIO 확장 설치.

[#3 - PlatformIO 확장 설치](/posts/platformio-guide/platformio-guide-03/)
