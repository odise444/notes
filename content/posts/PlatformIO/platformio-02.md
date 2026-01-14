---
title: "PlatformIO 완벽 가이드 #2 - VSCode 설치"
date: 2024-12-22
tags: ["PlatformIO", "VSCode", "설치"]
categories: ["개발환경"]
series: ["PlatformIO 완벽 가이드"]
summary: "Visual Studio Code 설치하기. 5분이면 끝."
---

PlatformIO는 VSCode 위에서 돌아간다.

먼저 VSCode를 설치하자.

---

## Visual Studio Code란

Microsoft가 만든 **무료** 코드 에디터.

- 가볍다 (Visual Studio랑 다름)
- 확장 기능으로 뭐든 됨
- Windows, Mac, Linux 다 됨
- 전 세계 개발자가 제일 많이 씀

---

## 다운로드

공식 사이트: https://code.visualstudio.com

{{< figure src="/imgs/pio-vscode-download.png" caption="VSCode 다운로드 페이지" >}}

운영체제 자동 감지해서 맞는 버전 보여줌.

파란 버튼 클릭.

---

## Windows 설치

1. 다운로드된 `VSCodeUserSetup-x64-x.xx.x.exe` 실행

2. 라이센스 동의

3. **설치 옵션** (중요!)

```
☑ "Code(으)로 열기" 작업을 Windows 탐색기 파일의 상황에 맞는 메뉴에 추가
☑ "Code(으)로 열기" 작업을 Windows 탐색기 디렉터리의 상황에 맞는 메뉴에 추가
☑ PATH에 추가 (다시 시작한 후 사용 가능)
```

이거 다 체크. 나중에 편함.

4. 설치 완료

---

## Mac 설치

1. 다운로드된 `VSCode-darwin-xxx.zip` 압축 해제

2. `Visual Studio Code.app`을 응용 프로그램 폴더로 이동

3. 처음 실행 시 "인터넷에서 다운로드한 앱입니다" 경고 → 열기

---

## Linux 설치

### Ubuntu/Debian

```bash
# .deb 파일 다운로드 후
sudo dpkg -i code_x.xx.x_amd64.deb
```

또는 snap:

```bash
sudo snap install code --classic
```

### Fedora/RHEL

```bash
sudo rpm -i code-x.xx.x.x86_64.rpm
```

---

## 첫 실행

{{< figure src="/imgs/pio-vscode-first.png" caption="VSCode 첫 화면" >}}

처음 열면 Welcome 탭이 보인다.

왼쪽에 아이콘들:
- 📁 탐색기 (Explorer)
- 🔍 검색 (Search)
- 🔀 소스 제어 (Source Control)
- ▶️ 실행 및 디버그 (Run and Debug)
- 🧩 확장 (Extensions)

---

## 한국어 설정 (선택)

영어가 불편하면 한국어로 바꿀 수 있다.

1. `Ctrl + Shift + P` (Mac: `Cmd + Shift + P`)
2. "Configure Display Language" 입력
3. "한국어" 선택
4. 재시작

근데 영어 추천. 검색할 때 영어가 자료 많음.

---

## 테마 변경 (선택)

기본 테마가 어두우면:

1. `Ctrl + K`, `Ctrl + T`
2. 원하는 테마 선택

밝은 테마: Light+, Light Modern
어두운 테마: Dark+, One Dark Pro

---

## 폰트 설정 (선택)

코딩용 폰트 추천:
- **Fira Code** - 합자(ligature) 지원
- **JetBrains Mono** - 가독성 좋음
- **D2Coding** - 한글 지원

설정 방법:
1. `Ctrl + ,` (설정 열기)
2. "font family" 검색
3. 원하는 폰트 입력

---

## 확인

터미널 열어보기:

`Ctrl + `` (백틱, 숫자 1 왼쪽)

```
Windows PowerShell
Copyright (C) Microsoft Corporation.
PS C:\Users\YourName>
```

터미널 잘 열리면 OK.

---

## 다음 단계

VSCode 준비 완료.

다음 글에서 PlatformIO 확장 설치.

---

다음 글: [#3 - PlatformIO 확장 설치](/posts/platformio/platformio-03/)
