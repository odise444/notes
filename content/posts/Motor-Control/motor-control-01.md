---
title: "Motor Control #1 - maxon ESCON 50/5로 국산 BLDC 돌리기"
date: 2026-04-24
lastmod: 2026-04-24
tags: ["모터제어", "BLDC", "maxon", "ESCON", "STM32", "PWM"]
categories: ["Motor-Control"]
series: ["Motor-Control"]
summary: "maxon ESCON 50/5에 모터뱅크 BL57101E1K를 물려 돌려본 기록. Startup Wizard, Diagnostics, Auto Tuning까지."
ShowToc: true
TocOpen: true
---

## 들어가며

BMS 펌웨어 일을 주로 해왔는데 최근에 모터 제어 건이 하나 들어왔다. 고객이 지정한 컨트롤러는 maxon ESCON 50/5인데, 붙여야 할 모터는 maxon 제품이 아니라 국산이다. 모터뱅크의 BL57101E1K, 감속기는 PGH56S.

ESCON은 maxon 모터에 맞춰 설계된 물건이라 국산 BLDC를 물리면 사양서와 실제값이 안 맞는 경우가 종종 있다. 국내에 한국어 자료도 거의 없어서 삽질한 내용을 정리해 둔다.

## 하드웨어 구성

| 구성 | 제품 | 주요 사양 |
|---|---|---|
| 서보 컨트롤러 | maxon ESCON 50/5 (#409510) | 50V, 연속 5A / 피크 15A |
| BLDC 모터 | 모터뱅크 BL57101E1K | 24V, 200W, 57각, Hall + 1000CPR 엔코더 |
| 감속기 | 모터뱅크 PGH56S | 1단 유성, 1:16, 백래시 15arcmin |
| 제어 MCU | STM32 (예정) | PWM + GPIO로 속도/방향 제어 |

### 모터 사양서

모터뱅크에서 같이 준 BL57101E1K 사양서다.

![모터 사양서](motor-control-01/01.png)

찾아볼 값만 추리면 이 정도.

- 3상, 8폴 (사양서 표기) — 실제 pole pairs는 뒤에서 뒤집힌다
- 정격 전압 24 VDC
- 무부하 속도 3700 ±10% RPM
- 정격 속도 3000 RPM
- 정격 토크 0.63 Nm
- 정격 출력 200 W
- 정격 전류 12A 이하

### 감속기 사양

![감속기 사양](motor-control-01/02.png)

PGH56S16은 56mm 플랜지 1단 유성 감속기. 감속비 1:16, 효율 94%, 백래시 기본 60 arcmin 이하, 고정밀 옵션은 15 arcmin.

## 사양서를 그대로 믿지 말 것

이 포스트에서 가장 먼저 짚어두고 싶은 건 이거다. 국산/중국산 BLDC는 사양서 값과 실측값이 어긋나는 경우가 꽤 있다. 이번에도 두 군데가 달랐다.

| 항목 | 사양서 | 실제 (ESCON 실측) |
|---|---|---|
| 극수 (Pole Pairs) | 4 (8폴) | 2 (4폴) |
| 엔코더 분해능 | 1000 CPR | 500 CPR |

왜 그런지는 아래에서 하나씩 푼다. 다행인 건 ESCON의 Diagnostics 툴이 이 차이를 자동으로 잡아준다는 점이다.

## Startup Wizard

ESCON Studio 실행하면 Startup Wizard가 뜬다. 총 18단계인데 주요 화면만 본다.

### Motor Data

![Motor Data 설정](motor-control-01/03.png)

- Speed Constant: 149.9 rpm/V
  - 무부하 3700 rpm ÷ 24V ≈ 154 rpm/V. IR 강하 고려해서 살짝 낮춰 잡았다.
- Thermal Time Constant Winding: 4.0s
  - 정확한 값 모를 때는 보수적으로. 57각 200W급이면 보통 30s 정도 쓸 수 있지만, 이후에 Current Limit을 낮게 잡을 거라 짧게 둬도 안전하다.
- Number of Pole Pairs: 4
  - 사양서대로 넣음. 나중에 2로 바뀐다.

### System Data

![System Data 설정](motor-control-01/04.png)

- Max. Permissible Speed: 3000 rpm
  - 실제론 4000 rpm 정도 여유를 두는 게 낫다.
- Nominal Current: 5.0 A
  - ESCON 50/5의 연속 정격이 5A라 여기에 맞췄다. 모터는 8.3A까지 견디지만 컨트롤러 쪽이 병목이다.
- Max. Output Current Limit: 10.0 A

이 단계에서 ESCON 50/5가 이 모터에 좀 작다는 게 드러난다. 고부하 연속 운전이면 70/10이 정답인데 저부하 테스트용이라 50/5로 충분하다.

### Detection of Rotor Position

![Rotor Position 설정](motor-control-01/05.png)

- Sensor Type: Digital Hall Sensors
- Hall Sensor Polarity: maxon

국산이라 Polarity가 maxon 표준과 다를 수 있지만 일단 기본으로 두고 Diagnostics에서 확인하는 흐름.

### Speed Sensor

![Speed Sensor 설정](motor-control-01/06.png)

- Sensor Type: Digital Incremental Encoder
- Encoder Resolution: 1000 Counts/turn
- Encoder Direction: maxon

여기서 주의할 건, ESCON이 내부적으로 4체배(quadrature decoding)를 자동 적용한다는 점이다. 사용자는 엔코더 원래 분해능만 넣으면 되고 4000을 직접 넣으면 안 된다.

### Enable 모드

![Enable 설정](motor-control-01/07.png)

- Functionality: Enable & Direction
- DI2: Enable (High active)
- DI3: Direction (High active)

Enable 모드는 네 가지 중에 고르게 된다.

| 모드 | 특징 | 용도 |
|---|---|---|
| Enable | 1핀, 양극성 Set Value | 가장 단순 |
| Enable & Direction | 2핀, 단극성 Set Value | 공장 HMI 스타일 (이번 선택) |
| Enable CW & Enable CCW | 2핀, 방향별 Enable | 리미트 스위치 인터록 |
| Enable & Stop | 2핀, 제어된 감속 정지 | 의료기기 같은 안전 응용 |

사람이 타는 기기(Smart Rollator 같은)에는 Enable & Stop도 좋은 선택이다.

### Set Value (PWM)

![Set Value (PWM) 설정](motor-control-01/08.png)

- Functionality: PWM Set Value
- Input: Digital Input 1
- Speed at 10% duty: 0 rpm
- Speed at 90% duty: 3000 rpm

STM32 타이머 PWM 출력을 ESCON DI1에 바로 물릴 계획이라 PWM 모드로 잡았다. 10~90%로 여유를 두는 건 안전 설계 관행이다.

0%/100%를 안 쓰는 이유는 단순하다. 0% duty는 "신호 없음"과 구분이 안 된다. 케이블이 끊겼거나 MCU가 부팅 안 된 상태랑 같은 출력이 나와버린다. 100% duty도 마찬가지로 "고정 High"랑 못 가린다. 10~90%만 쓰면 양 끝값이 에러 신호가 된다.

### Current Limit / Speed Ramp

![Current Limit 설정](motor-control-01/09.png)

![Speed Ramp 설정](motor-control-01/10.png)

- Current Limit: Fixed, 10.0 A
- Acceleration / Deceleration: 1000 rpm/s (0 → 3000 rpm에 3초)

초기 실험이니 보수적으로. 의료기기처럼 부드러운 가속이 필요한 응용엔 이 정도가 맞고, 일반 서보면 3000~5000 rpm/s로 올려야 한다.

### Offset

![Offset 설정](motor-control-01/11.png)

Fixed Offset 0.0 rpm. 처음엔 0으로 두고 실제 운전에서 Set Value가 0인데 모터가 미세 회전하면 그때 보정한다.

### Digital I/O

![Digital I/O 설정](motor-control-01/12.png)

| 핀 | 기능 |
|---|---|
| DI1 | PWM Set Value (MCU PWM) |
| DI2 | Enable (MCU GPIO) |
| DI3 | Direction (MCU GPIO) |
| DI/O4 | None (예비) |

DI/O4에 Ready 출력을 할당하면 MCU가 ESCON 에러 상태를 바로 감지할 수 있어 안전 응용에 유용하다. 이번엔 비워 뒀다.

### Analog I/O

Analog 입력은 다 None(PWM 쓰니까).

![Analog Inputs 설정](motor-control-01/13.png)

출력은 디버깅/모니터링에 쓴다.

![Analog Outputs 설정](motor-control-01/14.png)

- AO1: Actual Speed Averaged
- AO2: Actual Current Averaged

![AO1 스케일링](motor-control-01/15.png)

![AO2 스케일링](motor-control-01/16.png)

- AO1: 0.000V → 0 rpm / 3.300V → 3000 rpm
- AO2: 0.000V → 0 A / 3.300V → 10 A

최대값을 3.3V에 맞춘 건 STM32 ADC 레퍼런스가 3.3V라 풀스케일을 꽉 채우기 위함이다. 12bit ADC 기준으로 속도 분해능은 대략 0.73 rpm/LSB, 전류는 2.4 mA/LSB 나온다.

### Configuration Summary

![Configuration Summary](motor-control-01/17.png)

모든 설정이 한 화면에 요약된다. 여기서 Show Wiring Overview 버튼은 꼭 눌러 배선도를 캡처해 두자. 실제 배선할 때 없으면 고생한다.

## Diagnostics — 실제 문제는 여기서 드러난다

Startup Wizard 끝내고 바로 Auto Tuning을 돌리면 안 된다. Diagnostics로 배선과 센서 방향을 먼저 검증해야 한다. 국산 BLDC 붙였을 때 이상한 건 이 단계에서 대부분 걸린다.

### System Configuration

![Diagnostics System Config 초기](motor-control-01/18.png)

기본 화면이다. 감속기 정보를 정확히 넣어야 한다.

![Diagnostics System Config 수정](motor-control-01/19.png)

- Is a gear mounted to the motor?: Yes
- Sense of rotation: Same
- Reduction: 16.0 : 1

PGH56S는 1단 유성이라 모터 회전과 출력 회전 방향이 같다(Same). 감속비 16:1을 정확히 넣지 않으면 Pole Pairs 테스트가 엉뚱한 결과를 낸다.

하단에 "speed sensor가 감속기 출력에 장착되어 있으면 테스트 실패 가능"이라는 경고가 뜨는데, BL57101E1K는 엔코더가 모터 뒷단에 붙어 있어서 해당 사항 없음.

### Test Execution

![Diagnostics Test 대기 화면](motor-control-01/20.png)

Standby 상태에서 Start Tests 클릭. 11개 항목을 순차 자동 검증한다. 도중에 모터가 스스로 회전하니 안전 공간 확보가 중요하다. 축에 무거운 게 물려 있으면 떼놓는 게 낫다.

### Motor Direction of Rotation Failed

![Motor Direction Failed](motor-control-01/21.png)

첫 번째 실패. 에러 메시지는 이거다.

```
Possible Fault Reasons:
- 2 winding connections interchanged
- Wrong motor polarity
```

BL57101E1K의 3상 권선 극성 정의가 maxon 표준과 달라서 그렇다. 국산 BLDC에선 흔한 일.

해결 선택지는 두 가지.

1. Correct wiring: 전원 끄고 3상 중 2개 선을 바꿔 다시 물린다
2. Motor polarity corrected by software: 소프트웨어로 뒤집는다

소프트웨어 보정을 선택했다. 배선 재작업 없이 바로 해결되고, 나중에 다른 모터 다룰 때도 동일한 방식으로 일관성을 유지할 수 있다. Resume Tests.

### Number of Pole Pairs Failed

![Pole Pairs 팝업](motor-control-01/22.png)

극쌍 수 검증은 사용자가 관여하는 테스트다. 모터축에 마크를 붙이고, 전기적으로 360° 회전 명령이 갔을 때 기계적으로 실제 몇 도 돌았는지를 관찰해야 한다.

4 pole pairs면 1/4 회전(90°), 2 pole pairs면 1/2 회전(180°)이 정상.

![Pole Pairs Failed](motor-control-01/23.png)

결과.

```
Set number of pole pairs:       4   (사용자 입력)
Suggested number of pole pairs: 2   (ESCON 실측)
```

사양서엔 "3상 8폴"이라고 적혀 있는데 실제로는 4극(2 pole pairs)이다. 사양서 표기 관행의 차이다.

BLDC 쪽 극수 표기는 제조사마다 통일이 안 돼 있다.

| 표기 | 의미 | Pole Pairs |
|---|---|---|
| "8 Poles" (전체 극 수) | N 4개 + S 4개 | 4 |
| "8 Poles" (한 극성만) | N극만 8개 | 8 |
| 실제 BL57101E1K | ESCON 실측 | 2 |

무부하 3700 rpm이라는 속도를 생각하면 4극(2 pole pairs)이 물리적으로도 자연스럽다. 극수가 많을수록 토크는 세지고 속도는 느려지는 쪽으로 가는데, 이 모터는 속도가 꽤 빠른 편이다.

2로 수정하고 Resume.

### Encoder Direction Failed

![Encoder Direction Failed](motor-control-01/24.png)

Motor polarity를 소프트웨어로 뒤집었더니 이번엔 엔코더 방향이 반대가 됐다. 예상된 부작용이다.

똑같이 Encoder direction corrected by software로. 일관성 있게.

### Encoder Resolution Failed

![Encoder Resolution Failed](motor-control-01/25.png)

```
Set encoder resolution:      1000 Counts/turn
Detected encoder resolution:  500 Counts/turn
```

정확히 절반이다. 이 패턴은 엔코더 사양 표기의 혼용에서 온다.

"2CH 1000CPR"이라는 스펙 문구는 해석이 여러 가지 가능하다.

| 해석 | 엔코더 슬릿 수 | ESCON 입력값 |
|---|---|---|
| 채널당 1000 펄스 | 1000 | 1000 |
| 2채널 합계 1000 펄스 | 500 | 500 |
| 4체배 후 1000 카운트 | 250 | 250 |

BL57101E1K는 두 번째 해석에 해당한다. 엔코더 디스크에 500개 슬릿이 있고 A채널 500 + B채널 500 = 합계 1000으로 표기한 것. 중국 OEM 관행에서 흔한 방식이다.

Detected 값 500 그대로 적용하고 Resume.

### All Tests Passed

![All Tests Passed](motor-control-01/26.png)

11개 항목 모두 통과.

```
Motor:        3/3 Pass
Hall Sensors: 5/5 Pass
Encoder:      3/3 Pass
```

Hall Sensors 5개가 한 번에 붙은 건 좀 운이 좋았다. BL57101E1K의 Hall 배치가 maxon 표준(120°)과 그대로 호환된다는 얘기다.

### Diagnostics로 확정된 설정값

| 항목 | 초기값 | 최종값 | 변경 사유 |
|---|---|---|---|
| Pole Pairs | 4 | 2 | ESCON 실측 |
| Motor Polarity | maxon | Software corrected | 국산 모터 특성 |
| Encoder Resolution | 1000 | 500 Counts/turn | 사양서 표기 차이 |
| Encoder Direction | maxon | Software corrected | Motor 보정의 연쇄 |
| Hall Polarity | maxon | maxon (유지) | 호환 확인 |

## Auto Tuning

Diagnostics 끝났으니 Regulation Tuning → Auto Tuning 실행.

![Auto Tuning 완료](motor-control-01/27.png)

세 개의 제어 루프 모두 Dimensioned로 떴다.

```
Current Controller                 : Dimensioned
Speed Controller (Closed Loop)     : Dimensioned
  with Current Limiter
Speed Controller (Open Loop)       : Dimensioned
```

왼쪽 Current Loop 그래프는 2.5A 스텝에 대해 명령(파란선)을 실측(빨간선)이 거의 그대로 따라간다. Rise time 수 ms 이내, 오실레이션도 빠르게 수렴. 전류 루프 대역폭은 대략 1 kHz 정도로 보인다.

오른쪽 Speed Loop는 +500 / -500 / 0 rpm 양방향 스텝 응답이다. Overshoot 5% 이내, Settling time 100 ms 이내, Steady-state error 거의 0. CW/CCW 대칭성도 괜찮다. 일반 산업 응용엔 이 정도면 충분하다.

## 최종 설정값 요약

### Startup Wizard 입력값

| 화면 | 항목 | 값 |
|---|---|---|
| Motor Data | Speed Constant | 149.9 rpm/V |
| | Thermal Time Constant | 4.0 s |
| | Pole Pairs | 2 (Diagnostics 후) |
| System Data | Max. Permissible Speed | 3000 rpm |
| | Nominal Current | 5.0 A |
| | Max. Output Current | 10.0 A |
| Rotor Position | Sensor | Digital Hall Sensors |
| | Polarity | maxon |
| Speed Sensor | Sensor | Digital Incremental Encoder |
| | Resolution | 500 Counts/turn (Diagnostics 후) |
| | Direction | maxon (소프트웨어 보정) |
| Mode | Operation | Speed Controller (Closed Loop) |
| Enable | Mode | Enable & Direction (DI2/DI3) |
| Set Value | Type | PWM Set Value (DI1) |
| | Speed at 10% | 0 rpm |
| | Speed at 90% | 3000 rpm |
| Current Limit | Fixed | 10 A |
| Speed Ramp | Acc / Dec | 1000 / 1000 rpm/s |
| Offset | Fixed | 0 rpm |
| AO1 | Actual Speed | 0V=0rpm / 3.3V=3000rpm |
| AO2 | Actual Current | 0V=0A / 3.3V=10A |

### 배선 요약

```
[STM32 MCU]              [ESCON 50/5]
TIM_CHx (PWM)    ────>   Digital Input 1
GPIO (Enable)    ────>   Digital Input 2
GPIO (Direction) ────>   Digital Input 3
ADC1             <────   Analog Output 1 (Speed)
ADC2             <────   Analog Output 2 (Current)
GND              ────    Signal GND

[전원]
+24V DC 15A      ────>   +V Power
GND              ────>   Power GND

[모터 BL57101E1K]        [ESCON 50/5]
U / V / W                Motor U/V/W
Hall +5V/GND/H1/H2/H3    Hall Sensor 5핀
Enc +5V/GND/A/B          Encoder 4핀
```

## 정리하면

이번 건에서 건진 게 대충 네 가지다.

사양서를 100% 믿지 말 것. "8 Poles"가 4 pole pairs일 거라 넣었는데 실제는 2 pole pairs, "1000 CPR"도 채널당 1000이 아니라 합계 1000이었다. 다음부터 엔코더 구매할 땐 "디스크 슬릿이 정확히 몇 개인가"를 업체에 직접 물어봐야 한다.

Diagnostics가 사양서 오류까지 잡아 준다는 점이 ESCON의 진짜 가치라고 본다. Startup Wizard 끝나고 바로 Auto Tuning 돌리는 유혹이 있는데, Diagnostics를 먼저 돌리지 않으면 뒤에서 더 크게 꼬인다.

소프트웨어 보정이 배선 교체보다 낫다. 국산 BLDC는 대부분 극성/방향이 maxon 표준과 다른데, 배선을 뜯어 바꾸는 건 실수 유발 요소가 너무 많다. Diagnostics가 제안하는 소프트웨어 보정을 일관되게 쓰면 설정 파일에 기록되고 나중에 재현도 쉽다.

PWM Set Value는 임베디드 친화적이지만 저속에서 리플은 주의. MCU에서 DAC 없이 속도 제어가 된다는 건 편한데, 저속 구간에선 PWM 리플 때문에 미세 떨림이 올라올 수 있다. 10%/90% 엔드포인트로 Fail-safe도 같이 잡아 두는 게 좋다.

## 다음에 할 것

- 파라미터 파일(.edc) 저장하고 깃에 커밋
- STM32 쪽에 PWM + Enable + Direction 제어 로직 얹기
- ADC로 AO1/AO2 읽어 속도/전류 모니터링
- Data Recorder로 부하 걸린 상태의 응답 측정
- 필요하면 Expert Tuning에서 Controller Stiffness 손보기

STM32 쪽 제어 로직 얹는 건 다음 포스트에서 다룬다.

## 참고

- [maxon ESCON Studio 공식](https://www.maxongroup.com/maxon/view/content/escon-detailsite)
- [ESCON 50/5 Hardware Reference](https://www.maxongroup.com/medias/sys_master/root/8837504466974/409510-ESCON-50-5-Hardware-Reference-En.pdf)
- [모터뱅크 BL57101E1K](https://www.motorbank.kr/)
