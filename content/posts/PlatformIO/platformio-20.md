---
title: "PlatformIO 완벽 가이드 #20 - CI/CD 연동"
date: 2024-12-22
tags: ["PlatformIO", "CI/CD", "GitHub Actions", "자동화"]
categories: ["개발환경"]
series: ["PlatformIO 완벽 가이드"]
summary: "푸시하면 자동으로 빌드, 테스트."
---

코드 푸시할 때마다 자동으로 빌드하고 테스트하자.

GitHub Actions로 CI/CD 구축.

---

## CI/CD란

**Continuous Integration / Continuous Deployment**

- **CI**: 코드 변경 시 자동 빌드/테스트
- **CD**: 테스트 통과 시 자동 배포

임베디드에선 "배포" = 펌웨어 아티팩트 생성.

---

## GitHub Actions 설정

프로젝트 루트에 폴더 생성:

```
.github/
└── workflows/
    └── build.yml
```

---

## 기본 워크플로우

`.github/workflows/build.yml`:

```yaml
name: PlatformIO Build

on:
  push:
    branches: [ main, develop ]
  pull_request:
    branches: [ main ]

jobs:
  build:
    runs-on: ubuntu-latest

    steps:
      - name: Checkout
        uses: actions/checkout@v3

      - name: Setup Python
        uses: actions/setup-python@v4
        with:
          python-version: '3.11'

      - name: Install PlatformIO
        run: |
          pip install platformio

      - name: Build
        run: |
          pio run
```

---

## 동작 설명

1. **push/PR 시 트리거**
2. **Ubuntu 환경 준비**
3. **Python 설치**
4. **PlatformIO 설치**
5. **빌드 실행**

---

## 멀티 환경 빌드

```yaml
jobs:
  build:
    runs-on: ubuntu-latest
    strategy:
      matrix:
        environment: [esp32, uno, stm32]

    steps:
      - uses: actions/checkout@v3
      
      - uses: actions/setup-python@v4
        with:
          python-version: '3.11'

      - run: pip install platformio
      
      - name: Build ${{ matrix.environment }}
        run: pio run -e ${{ matrix.environment }}
```

여러 환경 병렬 빌드.

---

## 테스트 추가

```yaml
      - name: Build
        run: pio run

      - name: Test (Native)
        run: pio test -e native
```

네이티브 테스트 추가.

---

## 아티팩트 업로드

빌드 결과물 저장:

```yaml
      - name: Build
        run: pio run

      - name: Upload Firmware
        uses: actions/upload-artifact@v3
        with:
          name: firmware
          path: .pio/build/*/firmware.*
```

GitHub에서 다운로드 가능.

---

## 전체 예시

```yaml
name: PlatformIO CI

on:
  push:
    branches: [ main, develop ]
  pull_request:
    branches: [ main ]

jobs:
  build:
    runs-on: ubuntu-latest

    steps:
      - name: Checkout
        uses: actions/checkout@v3

      - name: Cache PlatformIO
        uses: actions/cache@v3
        with:
          path: ~/.platformio
          key: ${{ runner.os }}-pio-${{ hashFiles('platformio.ini') }}

      - name: Setup Python
        uses: actions/setup-python@v4
        with:
          python-version: '3.11'

      - name: Install PlatformIO
        run: pip install platformio

      - name: Build All Environments
        run: pio run

      - name: Run Tests
        run: pio test -e native

      - name: Upload Artifacts
        uses: actions/upload-artifact@v3
        with:
          name: firmware-${{ github.sha }}
          path: |
            .pio/build/*/firmware.bin
            .pio/build/*/firmware.hex
```

---

## 캐시

PlatformIO 패키지 캐시로 빌드 속도 향상:

```yaml
      - name: Cache PlatformIO
        uses: actions/cache@v3
        with:
          path: ~/.platformio
          key: ${{ runner.os }}-pio-${{ hashFiles('platformio.ini') }}
```

---

## 상태 뱃지

README에 빌드 상태 표시:

```markdown
![Build Status](https://github.com/username/repo/actions/workflows/build.yml/badge.svg)
```

---

## 릴리스 자동화

태그 푸시 시 릴리스 생성:

```yaml
on:
  push:
    tags:
      - 'v*'

jobs:
  release:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v3
      - run: pip install platformio
      - run: pio run
      
      - name: Create Release
        uses: softprops/action-gh-release@v1
        with:
          files: .pio/build/*/firmware.*
```

`git tag v1.0.0 && git push --tags` 하면 자동 릴리스.

---

## 시리즈 끝!

20개 글에 걸쳐 PlatformIO 완벽 가이드를 마쳤다.

요약:
- VSCode + PlatformIO 설치
- 프로젝트 생성과 구조
- 빌드, 업로드, 시리얼 모니터
- 라이브러리 관리
- IntelliSense와 포맷팅
- 멀티보드, 빌드 플래그
- 디버깅, 테스트, CI/CD

Arduino IDE 졸업 완료! 🎓

---

[#1 - Arduino IDE의 한계](/posts/platformio/platformio-01/)로 돌아가기
