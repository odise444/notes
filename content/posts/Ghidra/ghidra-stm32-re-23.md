---
title: "Ghidra STM32 역분석 #23 - Thumb 모드 함정"
date: 2024-12-14
draft: false
tags: ["Ghidra", "STM32", "리버스엔지니어링", "역분석", "ARM", "Thumb"]
categories: ["역분석"]
series: ["Ghidra 역분석"]
summary: "주소 LSB가 1인 이유. 처음엔 버그인 줄 알았다."
---

Vector Table 분석할 때 이상한 걸 발견했다.

```
Reset_Handler: 0x08002801
```

주소가 홀수다. Flash는 2바이트 정렬인데?

---

## Thumb 비트

Cortex-M은 Thumb 모드만 지원한다. ARM 모드 없음.

그래서 분기할 때 주소 LSB로 모드를 표시한다:
- LSB = 0: ARM 모드
- LSB = 1: Thumb 모드

실제 주소는 `0x08002800`이고, `+1`은 "Thumb으로 실행해라"라는 의미.

---

## Ghidra에서 헷갈리는 부분

```c
void (*func)(void) = (void (*)(void))0x08002801;
func();
```

디컴파일러가 이렇게 보여주면 실제 함수 위치는 `0x08002800`이다.

라벨 달 때 주의:
- `0x08002801`에 라벨 달면 안 됨
- `0x08002800`에 달아야 함

---

## BX LR

함수 끝에 자주 보이는 명령:

```asm
bx lr
```

`LR` (Link Register)로 분기하면서 모드 전환. LR의 LSB 보고 Thumb/ARM 결정.

Cortex-M은 항상 Thumb이라 사실 의미 없지만, 호환성 때문에 남아있다.

---

## 삽질 사례

처음에 Vector Table 파싱할 때:

```python
reset_handler = struct.unpack('<I', data[4:8])[0]
# 0x08002801
```

이 주소로 Ghidra에서 Go to 하면 엉뚱한 데 간다.

```python
reset_handler = struct.unpack('<I', data[4:8])[0] & ~1
# 0x08002800
```

`& ~1`로 LSB 지워야 실제 주소.

---

다음 글에서 구조체 크기 추정 삽질.

[#24 - 구조체 크기 추정](/posts/ghidra/ghidra-stm32-re-24/)
