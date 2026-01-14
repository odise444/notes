---
title: "PlatformIO 완벽 가이드 #10 - 라이브러리 설치"
date: 2024-12-22
tags: ["PlatformIO", "라이브러리", "lib_deps"]
categories: ["개발환경"]
series: ["PlatformIO 완벽 가이드"]
summary: "센서 라이브러리 설치하고 사용하기."
---

DHT 센서 쓰려면 라이브러리가 필요하다.

PlatformIO에서 라이브러리 설치하는 방법.

---

## 방법 1: PIO Home에서 검색

1. PIO Home → Libraries
2. Registry 탭
3. 검색: "DHT sensor"

{{< figure src="/imgs/pio-library-search.png" caption="라이브러리 검색" >}}

4. "DHT sensor library" 클릭
5. 상세 페이지에서 **Add to Project** 클릭
6. 프로젝트 선택
7. Add

자동으로 `platformio.ini`에 추가됨.

---

## 방법 2: platformio.ini 직접 수정

```ini
[env:uno]
platform = atmelavr
board = uno
framework = arduino

lib_deps =
    adafruit/DHT sensor library@^1.4.4
```

저장하면 자동으로 다운로드.

**이 방법 추천.** 버전 관리가 명확함.

---

## 라이브러리 이름 찾기

### 공식 레지스트리

https://registry.platformio.org/

### PIO Home

Libraries → Registry에서 검색.

라이브러리 페이지에 정확한 이름 있음:

```
adafruit/DHT sensor library
```

---

## 버전 지정

```ini
lib_deps =
    adafruit/DHT sensor library          ; 최신 버전
    adafruit/DHT sensor library@1.4.4    ; 정확히 1.4.4
    adafruit/DHT sensor library@^1.4.4   ; 1.4.4 이상 1.x.x
    adafruit/DHT sensor library@~1.4.4   ; 1.4.4 이상 1.4.x
```

| 기호 | 의미 |
|------|------|
| (없음) | 최신 |
| `@1.4.4` | 정확히 그 버전 |
| `@^1.4.4` | 1.x.x 중 1.4.4 이상 |
| `@~1.4.4` | 1.4.x 중 1.4.4 이상 |

**팁**: `^`로 시작하면 마이너 업데이트 자동 적용.

---

## 의존성 라이브러리

DHT 라이브러리는 Adafruit Unified Sensor가 필요함.

```ini
lib_deps =
    adafruit/DHT sensor library@^1.4.4
    adafruit/Adafruit Unified Sensor@^1.1.9
```

안 넣으면 빌드할 때 에러:

```
fatal error: Adafruit_Sensor.h: No such file or directory
```

최신 PlatformIO는 의존성 자동 설치해주기도 함.

---

## 설치 확인

빌드하면 라이브러리 자동 다운로드:

```
Library Manager: Installing adafruit/DHT sensor library @ ^1.4.4
Downloading...
Unpacking...
```

설치된 라이브러리는 `.pio/libdeps/` 에 저장.

---

## 사용 예시

```cpp
#include <Arduino.h>
#include <DHT.h>

#define DHTPIN 2
#define DHTTYPE DHT22

DHT dht(DHTPIN, DHTTYPE);

void setup() {
  Serial.begin(115200);
  dht.begin();
}

void loop() {
  float h = dht.readHumidity();
  float t = dht.readTemperature();

  Serial.print("Humidity: ");
  Serial.print(h);
  Serial.print("%, Temp: ");
  Serial.print(t);
  Serial.println("°C");

  delay(2000);
}
```

---

## GitHub에서 직접 설치

공식 레지스트리에 없는 라이브러리:

```ini
lib_deps =
    https://github.com/username/repo.git
    https://github.com/username/repo.git#v1.0.0   ; 태그
    https://github.com/username/repo.git#develop  ; 브랜치
```

---

## 로컬 라이브러리

`lib/` 폴더에 직접 넣기:

```
lib/
└── MyLib/
    ├── MyLib.h
    └── MyLib.cpp
```

`lib_deps` 없이도 자동 인식.

---

## 라이브러리 업데이트

### PIO Home

Libraries → Installed → Update

### CLI

```bash
pio pkg update
```

### 특정 라이브러리

```bash
pio pkg update -l "adafruit/DHT sensor library"
```

---

## 팀 프로젝트

`platformio.ini`에 라이브러리 다 적어두면:

1. Git으로 프로젝트 공유
2. 팀원이 clone
3. 빌드하면 라이브러리 자동 설치

**Arduino IDE 문제였던 "라이브러리 없어서 에러" 해결!**

---

## 다음 단계

라이브러리 설치 완료!

다음 글에서 IntelliSense 자동완성.

---

다음 글: [#11 - IntelliSense 자동완성](/posts/platformio/platformio-11/)
