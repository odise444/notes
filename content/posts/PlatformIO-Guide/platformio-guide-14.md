---
title: "PlatformIO 입문 #14 - CI/CD"
date: 2024-12-22
tags: ["PlatformIO", "CI", "GitHub Actions", "자동화"]
categories: ["가이드"]
series: ["PlatformIO 입문"]
summary: "푸시하면 자동으로 빌드하고 테스트한다."
---

코드 푸시할 때마다 수동으로 빌드?

GitHub Actions로 자동화하자.

---

## CI/CD란

- **CI (Continuous Integration)**: 코드 푸시할 때마다 자동 빌드/테스트
- **CD (Continuous Deployment)**: 자동 배포

임베디드에서 CD는 OTA 업데이트 정도.

일단 CI부터.

---

## GitHub Actions

GitHub 저장소에 워크플로우 파일 추가하면 끝.

```
.github/
└── workflows/
    └── build.yml
```

---

## 기본 워크플로우

```yaml
# .github/workflows/build.yml
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
      - uses: actions/checkout@v4
      
      - name: Cache PlatformIO
        uses: actions/cache@v3
        with:
          path: |
            ~/.cache/pip
            ~/.platformio
          key: ${{ runner.os }}-pio-${{ hashFiles('**/platformio.ini') }}
      
      - name: Set up Python
        uses: actions/setup-python@v4
        with:
          python-version: '3.x'
      
      - name: Install PlatformIO
        run: |
          python -m pip install --upgrade pip
          pip install platformio
      
      - name: Build
        run: pio run
```

---

## 단계별 설명

### 트리거

```yaml
on:
  push:
    branches: [ main, develop ]
  pull_request:
    branches: [ main ]
```

- main, develop 브랜치에 푸시할 때
- main으로 PR 올릴 때

### 캐시

```yaml
- name: Cache PlatformIO
  uses: actions/cache@v3
  with:
    path: |
      ~/.cache/pip
      ~/.platformio
    key: ${{ runner.os }}-pio-${{ hashFiles('**/platformio.ini') }}
```

플랫폼, 툴체인 캐시해서 빌드 시간 단축.

첫 빌드: 5분
캐시 후: 1분

### 빌드

```yaml
- name: Build
  run: pio run
```

모든 환경 빌드.

---

## 멀티 환경 빌드

```yaml
jobs:
  build:
    runs-on: ubuntu-latest
    strategy:
      matrix:
        environment: [esp32, uno, bluepill]

    steps:
      # ... 설치 단계 ...
      
      - name: Build ${{ matrix.environment }}
        run: pio run -e ${{ matrix.environment }}
```

병렬로 여러 환경 빌드.

---

## 테스트 추가

```yaml
jobs:
  build:
    runs-on: ubuntu-latest
    steps:
      # ... 설치 단계 ...
      
      - name: Build
        run: pio run
      
      - name: Run Tests
        run: pio test -e native
```

네이티브 환경에서 유닛 테스트 실행.

---

## 빌드 아티팩트 저장

```yaml
- name: Build
  run: pio run

- name: Upload Firmware
  uses: actions/upload-artifact@v3
  with:
    name: firmware
    path: |
      .pio/build/*/firmware.bin
      .pio/build/*/firmware.elf
```

빌드된 바이너리 다운로드 가능.

---

## 릴리즈 자동화

태그 푸시하면 릴리즈 생성:

```yaml
name: Release

on:
  push:
    tags:
      - 'v*'

jobs:
  release:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      
      # ... PlatformIO 설치 ...
      
      - name: Build
        run: pio run
      
      - name: Create Release
        uses: softprops/action-gh-release@v1
        with:
          files: |
            .pio/build/esp32/firmware.bin
            .pio/build/uno/firmware.hex
        env:
          GITHUB_TOKEN: ${{ secrets.GITHUB_TOKEN }}
```

`git tag v1.0.0 && git push --tags` 하면 자동 릴리즈.

---

## 빌드 상태 뱃지

README.md에:

```markdown
![Build Status](https://github.com/사용자/저장소/workflows/PlatformIO%20CI/badge.svg)
```

---

## 전체 예시

```yaml
# .github/workflows/ci.yml
name: PlatformIO CI

on: [push, pull_request]

jobs:
  build:
    runs-on: ubuntu-latest
    strategy:
      matrix:
        environment: [esp32, uno]

    steps:
      - uses: actions/checkout@v4

      - name: Cache PlatformIO
        uses: actions/cache@v3
        with:
          path: |
            ~/.cache/pip
            ~/.platformio
          key: ${{ runner.os }}-pio

      - name: Set up Python
        uses: actions/setup-python@v4
        with:
          python-version: '3.11'

      - name: Install PlatformIO
        run: pip install platformio

      - name: Build ${{ matrix.environment }}
        run: pio run -e ${{ matrix.environment }}

  test:
    runs-on: ubuntu-latest
    needs: build
    
    steps:
      - uses: actions/checkout@v4

      - name: Set up Python
        uses: actions/setup-python@v4
        with:
          python-version: '3.11'

      - name: Install PlatformIO
        run: pip install platformio

      - name: Run Tests
        run: pio test -e native
```

---

## 로컬에서 테스트

GitHub Actions 워크플로우 로컬 테스트:

```bash
# act 설치 (https://github.com/nektos/act)
brew install act  # macOS

# 실행
act
```

---

팀 프로젝트에서 CI 없으면 혼란.
개인 프로젝트에서도 있으면 편함.

다음 글에서 트러블슈팅.

[#15 - 트러블슈팅](/posts/platformio-guide/platformio-guide-15/)
