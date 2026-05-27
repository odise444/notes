---
title: "Motor Control #3 - ESCON Data Recorder로 응답 측정하고 튜닝하기"
date: 2026-05-28
lastmod: 2026-05-28
tags: ["모터제어", "ESCON", "Data Recorder", "튜닝", "임베디드", "주파수응답"]
categories: ["Motor-Control"]
series: ["Motor-Control"]
summary: "2편 코드로 명령은 줬는데 진짜 그대로 도는지 어떻게 확인할까. ESCON Studio의 Data Recorder로 스텝 응답·부하 변동을 캡처하고 Expert Tuning까지 연결한다."
ShowToc: true
TocOpen: true
---

## 들어가며

[2편](/posts/motor-control/motor-control-02)에서 STM32 코드로 ESCON에 PWM 명령을 던졌다. ADC로 받은 피드백이 시리얼로 잘 찍히긴 하는데, 한 가지 찜찜한 게 있다.

"진짜 명령대로 도는 게 맞나?"

ADC가 보는 건 ESCON의 Analog Output이고, AO는 Averaged 옵션이 켜져 있는 상태다. 즉 STM32 쪽 측정값은 이미 한 번 평활화된 값이라 진짜 응답 파형은 못 본다. 1500 rpm 명령을 줬을 때 실측이 1500 rpm 부근에서 안정되는 건 보이는데, 거기까지 가는 동안 오버슈트가 있었는지, settling time이 몇 ms였는지, 부하가 갑자기 걸렸을 때 얼마나 떨어졌다 회복하는지 — 이런 시간 영역 특성은 ADC 평균값으론 안 보인다.

ESCON Studio엔 이걸 보라고 있는 도구가 따로 있다. Data Recorder.

## Data Recorder가 하는 일

ESCON Studio 좌측 메뉴 → Tuning → Data Recording.

오실로스코프와 비슷한 도구다. ESCON 내부 변수를 USB로 PC에 실시간 스트리밍해서 시간축 그래프로 보여준다. 외부 측정 장비 없이 "컨트롤러가 실제로 어떤 신호를 만들고 있는지"를 직접 들여다볼 수 있다는 게 가장 큰 강점이다.

핵심 사양은 이렇다.

| 항목 | 사양 |
|---|---|
| 동시 채널 | 최대 4채널 |
| 샘플 주기 | 0.1 ms ~ 100 ms (가변) |
| 최대 길이 | 약 30초 (샘플 주기에 따라 다름) |
| 트리거 | Manual / Rising / Falling / Level |
| 데이터 export | CSV, BMP, PNG |

샘플 주기 0.1 ms면 10 kHz 샘플링이라 BLDC 권선의 PWM 리플까지 다 보인다. 보통 속도 응답 보려면 1 ms (1 kHz)면 충분하다.

## 채널 선택 — 무엇을 보면 되나

채널 4개에 뭘 할당하는지가 분석의 절반이다. 흔히 잡는 조합 세 가지를 정리한다.

### A. 속도 루프 응답 보기

| 채널 | 변수 | 의도 |
|---|---|---|
| CH1 | Set Speed Value | 명령 |
| CH2 | Actual Speed Averaged | 실측 (필터링됨) |
| CH3 | Actual Speed | 실측 (raw) |
| CH4 | Actual Current Averaged | 토크 부담 |

CH2와 CH3을 같이 띄우는 이유는 ESCON 내부 Speed 필터의 효과를 직접 보기 위함이다. 평소엔 둘이 거의 겹치지만 부하가 급변할 때 둘이 벌어진다. 그 차이가 보이면 평활 필터의 시정수가 응답을 늦추고 있다는 신호다.

### B. 전류 루프 응답 보기

| 채널 | 변수 | 의도 |
|---|---|---|
| CH1 | Set Current Value | 내부 명령 (속도 PI 출력) |
| CH2 | Actual Current | raw |
| CH3 | Actual Speed | 결과적 속도 |
| CH4 | Motor Voltage | PWM 듀티 추이 |

전류 루프가 잘 안 도는 경우 — 권선 인덕턴스가 사양서랑 다르거나, Auto Tuning이 헐겁게 잡혔거나 — 여기서 바로 드러난다. CH4의 Motor Voltage가 포화(Saturation)에 자주 닿으면 ESCON이 가용 전압을 다 쓰고도 따라가지 못한다는 뜻이다.

### C. PWM 입력 검증

| 채널 | 변수 | 의도 |
|---|---|---|
| CH1 | PWM Input Duty | MCU가 보낸 듀티 |
| CH2 | Set Speed Value | 듀티 → rpm 환산 결과 |
| CH3 | Actual Speed Averaged | 실측 |
| CH4 | Digital Input State | Enable/Direction 비트맵 |

2편 코드를 처음 돌릴 때 가장 유용한 조합이다. MCU에서 보낸 듀티가 ESCON 내부에서 rpm으로 어떻게 환산되는지, Enable/Direction 핀이 의도대로 떨어지는지를 한 화면에서 본다. 디버깅 단계에선 이걸 먼저 본다.

## 트리거 설정

산업용 데이터 레코더 분석의 첫 단추는 트리거 잘 잡기다. Manual로 잡고 손으로 시작 버튼을 누르면 이벤트 직전 데이터를 놓치기 쉽다. 자동 트리거가 답이다.

ESCON Data Recorder 트리거 옵션은 네 가지.

| 모드 | 동작 | 용도 |
|---|---|---|
| Manual | 사용자가 Start 클릭 | 정상 동작 관찰 |
| Rising Edge | 채널 값이 임계값을 상향 통과 | Step 응답 |
| Falling Edge | 채널 값이 임계값을 하향 통과 | 정지 응답 |
| Level | 채널 값이 임계값 위/아래 진입 | 부하 변동 감지 |

스텝 응답 측정은 Rising Edge가 정석. 트리거 채널을 "Set Speed Value", 임계값을 50 rpm 정도로 잡아두면 STM32가 `motor_set_speed(1500)`을 호출하는 순간 자동으로 캡처가 시작된다.

Pre-trigger를 10% 정도로 잡아 두는 게 좋다. 트리거 시점 이전 데이터도 일부 캡처돼서 "명령 직전에 시스템이 어떤 상태였는지"가 같이 기록된다.

## 첫 측정: 무부하 스텝 응답

배선 그대로, 모터 축에 부하 없이, STM32에서 0 → 1500 rpm 스텝 명령을 던진다.

설정:

```
샘플 주기:   1 ms
캡처 길이:   2 s
Pre-trigger: 10%
트리거:      CH1 (Set Speed) Rising Edge @ 50 rpm

CH1: Set Speed Value
CH2: Actual Speed Averaged
CH3: Actual Current Averaged
CH4: (비움)
```

STM32 main 루프에선 이렇게 했다.

```c
HAL_Delay(500);                  // 안정화 대기
motor_set_speed(1500);           // 스텝
HAL_Delay(2000);
motor_set_speed(0);
HAL_Delay(1000);
```

캡처 결과를 텍스트 그래프로 옮기면 대략 이런 모양이 나온다.

```
rpm
1600 │              ┌─────────────────────
1500 │           ╭──┘ ╭──────────────────
1400 │          ╱    ↑ overshoot ~ 60 rpm
1200 │         ╱
1000 │        ╱
 800 │       ╱
 600 │      ╱
 400 │     ╱       ← Actual Speed
 200 │    ╱
   0 │───┘─ ─ ─ ← Set Speed (1500까지 즉시 점프, Ramp 1000 rpm/s)
     └─────────────────────────────────── time
     0    0.5   1.0   1.5   2.0 s
```

읽어야 할 지표 네 가지.

### Rise Time (10% → 90%)

0 rpm에서 1500 rpm까지의 10~90% 구간을 지나는 데 걸린 시간. ESCON Speed Ramp를 1000 rpm/s로 잡아 뒀으니 0 → 1500은 이론상 1.5초. 측정에서도 1.5초 근방이 나와야 정상이다.

Ramp 설정값과 실측 rise time이 일치한다는 건, ESCON 내부 속도 루프 대역폭이 Ramp 속도보다 훨씬 빠르다는 뜻이다. 즉 현재 시스템에서 속도 응답을 제한하는 건 Ramp 설정이지 PI gain이 아니다.

### Overshoot

목표값을 얼마나 초과했는가. 1편 Auto Tuning 그래프에서 Speed Loop가 5% 이내였으니 무부하 1500 rpm 명령이면 약 75 rpm 이내가 정상이다.

오버슈트가 크게 나오면 Speed Controller P gain이 너무 높다는 신호다. Expert Tuning에서 KP를 낮추거나 KI를 올리는 방향으로 간다 (자세한 건 뒤).

### Settling Time (±2% 밴드 진입 후 유지)

오버슈트 후 1500 rpm ± 30 rpm 범위로 들어와 머무르기 시작하는 시점. 잘 튜닝된 시스템은 200 ms 이내.

Settling time이 길면 KI가 너무 작거나, 댐핑이 부족하다는 뜻.

### Steady-state Current

정상 상태에서 모터가 잡아먹는 전류. 무부하라면 베어링 마찰 + 감속기 효율 손실만 보상하는 수준이다. PGH56S(효율 94%) + 베어링 합치면 보통 0.2~0.4 A 정도 나온다.

이 값이 1 A 넘어가면 어딘가에 마찰이 걸려 있다는 신호다. 축이 휘었거나, 감속기 그리스가 끈적해졌거나, 모터 자체가 데미지 입었거나.

## 두 번째 측정: 부하 변동 응답

스텝 응답이 깨끗하게 나왔다고 끝이 아니다. 진짜 운전 조건에선 부하가 시시각각 변한다. 사람이 타는 장비면 체중 이동, 컨베이어면 적재물 등락, 펌프면 출구 압력 변화 — 외란(disturbance)에 대한 반응이 어떤지가 더 중요한 경우가 많다.

부하 변동 응답을 보는 방법.

1. 모터를 1500 rpm으로 정상 운전 중인 상태로 만든다
2. 모터 축에 갑자기 부하를 건다 (장갑 끼고 손으로 잡거나 브레이크 토크 거는 장비 사용)
3. Data Recorder가 이 순간을 잡도록 트리거를 Actual Current Rising Edge로 설정

설정:

```
샘플 주기:   1 ms
캡처 길이:   1 s
Pre-trigger: 30%
트리거:      CH3 (Actual Current) Rising Edge @ 1.0 A

CH1: Set Speed Value          ← 1500 rpm 고정
CH2: Actual Speed Averaged    ← 부하에 끌려 떨어졌다 회복
CH3: Actual Current Averaged  ← 부하 보상하느라 증가
CH4: Motor Voltage            ← 포화 여부 확인
```

이때 봐야 할 건 두 가지.

### Speed Dip (속도 처짐)

부하가 걸린 순간 actual speed가 얼마나 떨어지는가. 잘 튜닝된 시스템이면 50 rpm 이내, 200 ms 안에 복귀한다.

Speed dip이 크면 Speed Controller가 외란을 늦게 보고 늦게 반응한다는 뜻. KI를 키우면 줄어드는데, 너무 키우면 무부하 응답에서 오버슈트가 커지는 trade-off가 있다.

### Voltage Saturation

부하 보상하느라 ESCON이 출력 전압을 끝까지 쓰는 상황. CH4 Motor Voltage가 ±전원 전압의 95% 이상을 자주 친다면 모터가 토크를 더 못 뽑는 한계에 가까운 거다. 이 경우엔 튜닝으로 해결 못 하고 컨트롤러 등급을 올리거나(50/5 → 70/10) 모터를 키워야 한다.

24V 전원 기준이면 약 22V 이상이 포화 영역. 부하 보상 중에 잠깐 닿는 건 정상이지만 정상 운전 중에도 자주 닿으면 시스템이 너무 빡빡한 거다.

## CSV로 export해서 외부 분석

화면 그래프만으론 정확한 수치 측정에 한계가 있다. Data Recorder 캡처 후 File → Export → CSV로 받아 Python으로 처리하는 게 정석이다.

```python
import numpy as np
import pandas as pd
import matplotlib.pyplot as plt

# ESCON Studio가 export한 CSV 읽기
df = pd.read_csv("step_response.csv", skiprows=2)
df.columns = ["time_ms", "set_speed", "actual_speed", "current"]

# 시간을 초로
df["t"] = df["time_ms"] / 1000.0

# Rise time: 명령값의 10% → 90%
target = df["set_speed"].iloc[-100:].mean()   # 정상상태 명령값
t10 = df.loc[df["actual_speed"] >= 0.1 * target, "t"].iloc[0]
t90 = df.loc[df["actual_speed"] >= 0.9 * target, "t"].iloc[0]
print(f"Rise time (10-90%): {(t90 - t10) * 1000:.1f} ms")

# Overshoot
peak = df["actual_speed"].max()
overshoot_pct = (peak - target) / target * 100
print(f"Overshoot: {overshoot_pct:.2f} % ({peak - target:.1f} rpm)")

# Settling time (±2% 밴드 진입 후 유지)
band = 0.02 * target
settled = (df["actual_speed"] - target).abs() < band
# settled가 True인 가장 늦은 False 다음 시점
last_false = settled[~settled].index.max()
t_settle = df["t"].iloc[last_false + 1] if pd.notna(last_false) else df["t"].iloc[0]
t_step = df.loc[df["set_speed"] >= 50, "t"].iloc[0]
print(f"Settling time (±2%): {(t_settle - t_step) * 1000:.1f} ms")

# Steady-state current
print(f"Steady-state current: {df['current'].iloc[-100:].mean():.3f} A")
```

이런 식으로 자동화해 두면 튜닝 파라미터 바꿀 때마다 매번 같은 지표가 나와서 비교가 깔끔해진다.

내 케이스 실측값 예시.

| 지표 | Auto Tuning 직후 | KP -10% | KP +10% |
|---|---|---|---|
| Rise time | 1.49 s | 1.50 s | 1.48 s |
| Overshoot | 4.2% | 2.8% | 7.1% |
| Settling time | 180 ms | 240 ms | 130 ms |
| Steady-state current | 0.31 A | 0.31 A | 0.32 A |

Rise time이 거의 안 변한 건 위에서 말한 대로 Speed Ramp가 응답을 지배하고 있어서다. KP 변화의 효과는 overshoot/settling에 집중적으로 나타난다.

## Expert Tuning과 연결하기

Auto Tuning이 잡아준 게인은 maxon 표준 모터 + 평균적인 부하 조건의 통계적 평균이다. 국산 모터 + 특수 부하면 미세 조정이 필요할 때가 있다. Expert Tuning은 이걸 손으로 조정하는 화면이다.

ESCON Studio → Tuning → Regulation Tuning → Expert Tuning.

손볼 수 있는 파라미터 네 가지.

| 파라미터 | 기호 | 효과 |
|---|---|---|
| Current Controller P-Gain | I-KP | 전류 응답 속도 |
| Current Controller I-Gain | I-KI | 전류 정상오차 제거 |
| Speed Controller P-Gain | S-KP | 속도 응답 강도 |
| Speed Controller I-Gain | S-KI | 외란 보상력 |

데이터 레코더 지표와 게인의 매핑은 대략 이렇다.

| 증상 | 원인 추정 | 조치 |
|---|---|---|
| Overshoot 크다 | S-KP 너무 큼 | S-KP 5~10% 감소 |
| Settling time 길다 | S-KI 작거나 댐핑 부족 | S-KI 10% 증가 |
| 부하 dip 크다 | S-KI 작음 | S-KI 20% 증가 |
| 정상 속도가 약간 못 미침 | S-KI 너무 작음 | S-KI 증가 |
| 전류 진동 (chattering) | I-KP 너무 큼 | I-KP 감소 |
| Voltage saturation 자주 | 시스템 한계 | 게인 조정으론 안 됨 |

조정은 한 번에 한 파라미터씩, 10% 단위로 움직인다. 이게 가장 중요한 룰이다. 두 개를 동시에 바꾸면 어느 변경이 효과를 냈는지 추적이 안 된다.

조정 후엔 반드시 Data Recorder로 같은 시험을 재실행해서 지표가 의도대로 움직였는지 확인한다. 의도 반대로 움직이면 즉시 되돌리고 다른 가설을 검증.

## 측정 자동화 — 회귀 테스트로 만들기

여기까지 한 번 한 측정은 일회성이지만, 튜닝 작업이 길어지면 같은 조건의 시험을 수십 번 반복하게 된다. 매번 손으로 Start 누르고 export 받는 건 비효율이다.

ESCON Studio가 명령행 자동화는 직접 지원하지 않지만, 두 가지 우회로가 있다.

첫째, ESCON Studio의 매크로 기능. Tools → Macros에서 채널 설정 + 트리거 + Start + Export를 한 시퀀스로 묶을 수 있다. 단축키에 매핑해두면 한 키로 측정 1회가 돌아간다.

둘째, STM32 쪽에서 시험 시퀀스를 코드화. main 루프에 시험 시퀀스를 넣고, 시퀀스가 끝나면 LED를 점멸시키든 UART로 "DONE"을 찍든. PC에서 그 신호를 받아 Data Recorder export를 자동화하는 식.

```c
typedef enum {
    TEST_IDLE,
    TEST_STEP_UP_1500,
    TEST_HOLD,
    TEST_STEP_DOWN,
    TEST_DONE,
} test_state_t;

static test_state_t test_state = TEST_IDLE;
static uint32_t test_t0 = 0;

void test_sequence_tick(void)
{
    uint32_t elapsed = HAL_GetTick() - test_t0;
    switch (test_state) {
    case TEST_IDLE:
        // 외부 트리거 대기 (버튼, UART 등)
        break;
    case TEST_STEP_UP_1500:
        motor_set_speed(1500);
        if (elapsed > 2000) {
            test_state = TEST_HOLD;
            test_t0 = HAL_GetTick();
        }
        break;
    case TEST_HOLD:
        if (elapsed > 1000) {
            test_state = TEST_STEP_DOWN;
            test_t0 = HAL_GetTick();
        }
        break;
    case TEST_STEP_DOWN:
        motor_set_speed(0);
        if (elapsed > 2000) {
            test_state = TEST_DONE;
        }
        break;
    case TEST_DONE:
        HAL_UART_Transmit(&huart2, (uint8_t*)"DONE\r\n", 6, 100);
        test_state = TEST_IDLE;
        break;
    }
}
```

이렇게 해 두면 게인 바꿀 때마다 "Start 시퀀스 → Data Recorder export → Python 분석" 사이클이 1분 안에 끝난다. 튜닝 효율이 확연히 달라진다.

## 함정 몇 가지

처음 Data Recorder를 쓰면 결과가 이상해 보일 때가 있다. 흔한 함정 정리.

**USB 끊김**. Data Recorder는 USB로 PC에 스트리밍하는데, USB 케이블이 길거나 EMI가 심한 환경이면 중간에 끊긴다. 끊긴 구간은 파형이 0으로 떨어지거나 멈춰 보인다. 캡처 중엔 케이블을 최단으로 직결하고 다른 USB 장비를 빼는 게 좋다.

**Pre-trigger가 짧음**. 5%로 잡으면 트리거 직전이 거의 안 잡혀서 "스텝 명령이 떨어진 순간"이 시각적으로 보이지 않는다. 10% 이상이 적당하다.

**샘플 주기와 캡처 길이의 trade-off**. 0.1 ms 샘플이면 캡처 길이가 3초로 줄어든다. 1 ms로 30초까지 갈 수 있다. 분석 목적에 맞춰 잡아야 한다. 속도 응답이면 1 ms가 적당하고, 전류 루프 자세히 보려면 0.1 ms로 잡되 캡처 길이는 짧게.

**Speed Ramp가 응답을 가리는 경우**. Auto Tuning 결과 보면 PI 게인이 굉장히 좋아 보이는데 실제 측정에선 응답이 느려 보일 수 있다. Speed Ramp 설정이 1000 rpm/s 같은 보수적 값이면 PI 응답이 아무리 빨라도 Ramp가 깎아낸다. PI 게인 측정할 땐 Ramp를 일시적으로 매우 큰 값(예: 50000 rpm/s)으로 올려놓고 측정한 뒤 원래대로 돌려놓는다.

**AO Averaged 옵션의 영향**. 1편에서 AO1/AO2에 Averaged 옵션을 켰는데, 이 평활화는 ESCON 내부 변수가 아니라 Analog Output에만 적용된다. Data Recorder는 ESCON 내부 raw 변수를 직접 보는 거라 AO 평활화의 영향을 안 받는다. 즉 STM32 ADC로 보는 값과 Data Recorder로 보는 값은 살짝 다를 수 있다는 점.

## 정리하면

ADC + UART로 보는 값으로는 시스템 응답 특성을 정량적으로 평가하기 어렵다. Data Recorder는 ESCON 내부에서 일어나는 일을 외부 측정 없이 직접 들여다보는 도구라 펌웨어 디버깅과 튜닝의 표준 절차로 박아두는 게 맞다.

채널 4개를 신중하게 고르는 게 분석의 절반이다. "보고 싶은 변수"가 아니라 "그 변수가 어디서 왔는지 추적할 수 있는 변수"를 같이 띄워야 한다. Set vs Actual, raw vs filtered, voltage saturation 여부 — 이런 짝지음이 디버깅에 결정적이다.

CSV export + Python 분석을 한 번 자동화해두면 튜닝 사이클이 비교가 안 되게 빨라진다. Auto Tuning이 잡아준 게인을 그대로 쓰는 시스템과, 측정 + 분석 + 재조정 사이클을 두세 번 돈 시스템은 응답 품질이 눈에 띄게 다르다.

Expert Tuning의 파라미터 변경은 한 번에 한 개씩, 10%씩, 측정 + 비교로 검증. 이 원칙을 깨면 어디서 어떤 효과가 났는지 추적이 안 된다.

## 다음에 할 것

3편으로 Motor-Control 시리즈 1차 마무리. 이후 후속 글에서 다룰 수 있는 주제.

- UART/CAN으로 속도·전류 로깅 외부 노트북으로 송출 (STM32 쪽 측정)
- ADC 노이즈 줄이기: RC 필터, 차폐 케이블, GND 토폴로지
- E-stop 시퀀스 구현: DI/O4 Ready 입력 + 외부 인터럽트
- Position Controller 모드로 전환 + 엔코더 기반 위치 제어
- maxon EPOS 시리즈 비교 (CANopen, EtherCAT)
- 국산 BLDC + 국산 드라이버 조합 케이스 스터디

이 중 어떤 게 다음 주제가 될지는 다음 모터 제어 건 들어오는 거 보고 정한다.

## 참고

- [maxon ESCON Studio 공식](https://www.maxongroup.com/maxon/view/content/escon-detailsite)
- [ESCON 50/5 Hardware Reference](https://www.maxongroup.com/medias/sys_master/root/8837504466974/409510-ESCON-50-5-Hardware-Reference-En.pdf)
- maxon Application Note "Tuning of Speed Controllers" — Expert Tuning 챕터의 게인 의미 설명
