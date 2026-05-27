---
title: "Motor Control #2 - STM32 PWM 속도 제어 펌웨어"
date: 2026-05-27
lastmod: 2026-05-27
tags: ["모터제어", "BLDC", "ESCON", "STM32", "PWM", "HAL", "임베디드"]
categories: ["Motor-Control"]
series: ["Motor-Control"]
summary: "ESCON 50/5에 STM32로 직접 PWM 명령을 던지고, 아날로그 모니터 출력을 ADC로 받아오는 펌웨어 전체 코드."
ShowToc: true
TocOpen: true
---

## 들어가며

지난 글에서 ESCON Studio의 Set Value Window로 BLDC를 돌리는 데까지 했다. 그건 셋업 검증이고, 실제 제품에 들어갈 때는 MCU가 ESCON에 명령을 줘야 한다. 이번 글에서는 STM32로 PWM 신호를 만들어 ESCON에 속도 명령을 던지고, ESCON의 아날로그 모니터 출력을 ADC로 받아오는 단계까지 한다.

다룰 내용:

- ESCON Set Value PWM 입력 사양
- 핀 매핑과 신호 레벨 이슈
- STM32 Timer PWM 코드 (HAL)
- Enable / Direction GPIO 제어
- ADC + DMA로 속도/전류 모니터링
- 통합 메인 루프 예제

보드는 Nucleo-F446RE 기준이다. F4 계열이면 핀만 바꿔서 그대로 쓸 수 있고, F1/G4도 HAL이라 큰 차이 없다.

## ESCON Set Value PWM 입력 사양

#1에서 Startup Wizard 돌릴 때 Set Value를 "PWM Set Value"로 잡았다. 이때 ESCON이 받아들이는 신호 사양은 이렇다.

| 항목 | 값 |
|------|----|
| 주파수 | 10 Hz ~ 5 kHz |
| 권장 주파수 | 1 kHz |
| 듀티 유효 범위 | 10% ~ 90% |
| 입력 레벨 | TTL (HIGH ≥ 2.4V) |
| 매핑 (Unipolar) | 10% → 0 rpm, 90% → 설정 최대 rpm |
| 매핑 (Bipolar) | 10% → -최대, 50% → 0, 90% → +최대 |

듀티가 10% 미만이거나 90% 초과면 ESCON이 신호 오류로 인식해서 Enable이 들어와도 모터를 안 돌린다. PWM 단선(0%, 100%)을 잡아내려고 일부러 데드밴드를 둔 거다.

이번 글은 방향을 별도 디지털 입력으로 받는 Unipolar 모드로 간다. ESCON Studio에서 Digital Input 2를 "Direction" 기능으로 할당해 둔 상태다.

레벨이 TTL 5V라는 점이 STM32에서 한 가지 걸린다. STM32 GPIO 출력은 3.3V인데, TTL 임계값(VIH ≥ 2.4V)을 넘기 때문에 보통은 직결로 인식된다. 다만 케이블이 길거나 노이즈가 끼면 마진이 부족해서 인식이 들쭉날쭉할 수 있다. 안 잡히면 74HCT04 같은 5V TTL 버퍼 하나 끼우는 게 답이다.

## 하드웨어 연결

Nucleo-F446RE 기준 핀 매핑이다.

| STM32 핀 | 기능 | ESCON 신호 | 비고 |
|----------|------|------------|------|
| PA8 | TIM1_CH1 PWM | Set Value (PWM IN) | 3.3V → 5V TTL |
| PA9 | GPIO 출력 | DigIN 1 (Enable) | |
| PA10 | GPIO 출력 | DigIN 2 (Direction) | |
| PA0 | ADC1_IN0 | AnOUT 1 (Speed) | 스케일링 필요 |
| PA1 | ADC1_IN1 | AnOUT 2 (Current) | 스케일링 필요 |
| GND | - | Signal GND | 공통 그라운드 필수 |

ESCON 쪽 핀 번호는 케이블/커넥터 버전에 따라 다르니 데이터시트의 J5 또는 J6 핀맵 확인하고 매핑한다.

아날로그 출력 스케일링은 한 가지 짚고 가야 한다. ESCON 50/5의 AnOUT 1/2는 기본이 ±10V 출력이다. STM32 ADC는 0~3.3V만 받는다. 두 가지 선택지:

1. ESCON Studio에서 AnOUT 설정을 "0..5V Unipolar"로 바꾼다. 그러면 0~5V 단방향이 되어 저항 분압 두 개로 끝난다.
2. ±10V를 유지하고 외부 OP07 차동 증폭기로 0~3.3V 변환. 양방향 토크/속도가 필요할 때만 이쪽.

이번 글은 Unipolar 5V로 가정한다. PA0/PA1 앞단에 R1=22k, R2=33k 분압이면 5V → 약 3.0V로 떨어진다.

## STM32 Timer PWM 설정

TIM1_CH1을 PA8로 빼서 1 kHz, 듀티 해상도 0.1%로 만든다. TIM1은 APB2(168 MHz)에 물려 있다.

```c
// timer1.h
#ifndef TIMER1_H
#define TIMER1_H

#include "stm32f4xx_hal.h"

void Timer1_PWM_Init(void);
void Timer1_PWM_SetDutyTenth(uint16_t duty_x10);  // 0~1000 (0.0%~100.0%)

#endif
```

```c
// timer1.c
#include "timer1.h"

static TIM_HandleTypeDef htim1;

void Timer1_PWM_Init(void)
{
    __HAL_RCC_TIM1_CLK_ENABLE();
    __HAL_RCC_GPIOA_CLK_ENABLE();

    // PA8 → TIM1_CH1, AF1
    GPIO_InitTypeDef gpio = {0};
    gpio.Pin       = GPIO_PIN_8;
    gpio.Mode      = GPIO_MODE_AF_PP;
    gpio.Pull      = GPIO_NOPULL;
    gpio.Speed     = GPIO_SPEED_FREQ_HIGH;
    gpio.Alternate = GPIO_AF1_TIM1;
    HAL_GPIO_Init(GPIOA, &gpio);

    // TIM1 clock 168 MHz → Prescaler 168 → 1 MHz tick
    // ARR 1000 → 1 kHz PWM, 해상도 0.1%
    htim1.Instance               = TIM1;
    htim1.Init.Prescaler         = 168 - 1;
    htim1.Init.CounterMode       = TIM_COUNTERMODE_UP;
    htim1.Init.Period            = 1000 - 1;
    htim1.Init.ClockDivision     = TIM_CLOCKDIVISION_DIV1;
    htim1.Init.RepetitionCounter = 0;
    htim1.Init.AutoReloadPreload = TIM_AUTORELOAD_PRELOAD_ENABLE;
    HAL_TIM_PWM_Init(&htim1);

    TIM_OC_InitTypeDef oc = {0};
    oc.OCMode     = TIM_OCMODE_PWM1;
    oc.Pulse      = 100;                  // 시작 듀티 10% = ESCON 정지
    oc.OCPolarity = TIM_OCPOLARITY_HIGH;
    oc.OCFastMode = TIM_OCFAST_DISABLE;
    HAL_TIM_PWM_ConfigChannel(&htim1, &oc, TIM_CHANNEL_1);

    HAL_TIM_PWM_Start(&htim1, TIM_CHANNEL_1);
}

void Timer1_PWM_SetDutyTenth(uint16_t duty_x10)
{
    if (duty_x10 > 1000) duty_x10 = 1000;
    // ESCON 사양 범위로 강제 클램프
    if (duty_x10 < 100)  duty_x10 = 100;
    if (duty_x10 > 900)  duty_x10 = 900;

    __HAL_TIM_SET_COMPARE(&htim1, TIM_CHANNEL_1, duty_x10);
}
```

체크 포인트 몇 개:

- TIM1은 어드밴스드 타이머라 PWM 출력을 켜려면 보통 `__HAL_TIM_MOE_ENABLE`(BDTR.MOE)를 켜줘야 하는데, HAL의 `HAL_TIM_PWM_Start`가 내부에서 처리한다. 그래서 명시적으로 안 써도 동작한다.
- `AutoReloadPreload` 켜둔 건 듀티 바꿀 때 그 주기 한 발만 망가지는 걸 막기 위한 거다. 안 켜도 1 kHz에서 큰 문제는 없지만 습관적으로 켠다.
- 듀티 해상도 0.1%면 ESCON 명령 분해능으로는 차고 넘친다. 모터 최대 5000 rpm이면 0.1%당 약 6.25 rpm.

## Enable / Direction GPIO

PWM은 신호일 뿐이고, ESCON이 실제로 모터를 돌리려면 DigIN 1 (Enable)이 HIGH여야 한다. 방향은 DigIN 2.

```c
// motor_ctrl.h
#ifndef MOTOR_CTRL_H
#define MOTOR_CTRL_H

#include <stdint.h>

typedef enum {
    MOTOR_DIR_CW  = 0,
    MOTOR_DIR_CCW = 1
} MotorDir_t;

void MotorCtrl_Init(void);
void MotorCtrl_Enable(void);
void MotorCtrl_Disable(void);
void MotorCtrl_SetDir(MotorDir_t dir);
void MotorCtrl_SetSpeedPercentTenth(uint16_t pct_x10);  // 0~1000

#endif
```

```c
// motor_ctrl.c
#include "motor_ctrl.h"
#include "timer1.h"
#include "stm32f4xx_hal.h"

#define EN_PORT   GPIOA
#define EN_PIN    GPIO_PIN_9
#define DIR_PORT  GPIOA
#define DIR_PIN   GPIO_PIN_10

void MotorCtrl_Init(void)
{
    __HAL_RCC_GPIOA_CLK_ENABLE();

    GPIO_InitTypeDef gpio = {0};
    gpio.Mode  = GPIO_MODE_OUTPUT_PP;
    gpio.Pull  = GPIO_NOPULL;
    gpio.Speed = GPIO_SPEED_FREQ_LOW;
    gpio.Pin   = EN_PIN | DIR_PIN;
    HAL_GPIO_Init(GPIOA, &gpio);

    // 부팅 직후 안전 상태: Disable, CW, PWM 10% (정지)
    HAL_GPIO_WritePin(EN_PORT,  EN_PIN,  GPIO_PIN_RESET);
    HAL_GPIO_WritePin(DIR_PORT, DIR_PIN, GPIO_PIN_RESET);

    Timer1_PWM_Init();
    Timer1_PWM_SetDutyTenth(100);
}

void MotorCtrl_Enable(void)
{
    HAL_GPIO_WritePin(EN_PORT, EN_PIN, GPIO_PIN_SET);
}

void MotorCtrl_Disable(void)
{
    HAL_GPIO_WritePin(EN_PORT, EN_PIN, GPIO_PIN_RESET);
}

void MotorCtrl_SetDir(MotorDir_t dir)
{
    HAL_GPIO_WritePin(DIR_PORT, DIR_PIN,
                      (dir == MOTOR_DIR_CW) ? GPIO_PIN_RESET : GPIO_PIN_SET);
}

void MotorCtrl_SetSpeedPercentTenth(uint16_t pct_x10)
{
    // 사용자 입력 0.0~100.0% → ESCON 듀티 10.0~90.0%
    if (pct_x10 > 1000) pct_x10 = 1000;
    uint16_t duty = 100 + (uint32_t)pct_x10 * 800u / 1000u;  // 100~900
    Timer1_PWM_SetDutyTenth(duty);
}
```

`SetSpeedPercentTenth` 한 함수가 이 모듈의 존재 이유다. 호출하는 쪽은 "속도 70.0%"라고 자연스럽게 부르면 되고, ESCON 데드밴드 10~90% 매핑은 안에서 처리한다. 나중에 ESCON 설정을 Bipolar로 바꿔도 함수 안만 고치면 된다. 호출하는 코드는 안 건드린다.

부팅 직후 순서도 중요하다. PWM을 먼저 정지(10%)로 띄워두고, Enable은 사용자 코드에서 따로 켜야 한다. 부팅하자마자 갑자기 모터가 튀어나가는 사고를 막는다.

## ADC + DMA 모니터링

ESCON AnOUT 1/2를 PA0/PA1로 받는다. 두 채널을 ADC1 스캔 모드로 돌리고 DMA를 순환으로 걸어두면, 메인 루프는 그냥 변수만 읽으면 된다.

```c
// adc_monitor.h
#ifndef ADC_MONITOR_H
#define ADC_MONITOR_H

#include <stdint.h>

typedef struct {
    uint16_t speed_raw;
    uint16_t current_raw;
} EsconMonitor_t;

extern volatile EsconMonitor_t g_monitor;

void  AdcMonitor_Init(void);
float AdcMonitor_GetSpeedRpm(void);
float AdcMonitor_GetCurrentA(void);

#endif
```

```c
// adc_monitor.c
#include "adc_monitor.h"
#include "stm32f4xx_hal.h"

// ESCON Studio Analog Output 설정과 일치시킬 것
#define MONITOR_SPEED_MAX_RPM   5000.0f
#define MONITOR_CURRENT_MAX_A   5.0f       // ESCON 50/5 연속 전류
#define ADC_FULL_SCALE_V        3.0f       // 분압 후 풀스케일
#define ADC_MAX                 4095.0f

static ADC_HandleTypeDef hadc1;
static DMA_HandleTypeDef hdma_adc1;

volatile EsconMonitor_t g_monitor;
static uint16_t adc_buf[2];

void AdcMonitor_Init(void)
{
    __HAL_RCC_GPIOA_CLK_ENABLE();
    __HAL_RCC_ADC1_CLK_ENABLE();
    __HAL_RCC_DMA2_CLK_ENABLE();

    // PA0, PA1 아날로그 입력
    GPIO_InitTypeDef gpio = {0};
    gpio.Pin  = GPIO_PIN_0 | GPIO_PIN_1;
    gpio.Mode = GPIO_MODE_ANALOG;
    gpio.Pull = GPIO_NOPULL;
    HAL_GPIO_Init(GPIOA, &gpio);

    // DMA2 Stream0, Channel0 = ADC1
    hdma_adc1.Instance                 = DMA2_Stream0;
    hdma_adc1.Init.Channel             = DMA_CHANNEL_0;
    hdma_adc1.Init.Direction           = DMA_PERIPH_TO_MEMORY;
    hdma_adc1.Init.PeriphInc           = DMA_PINC_DISABLE;
    hdma_adc1.Init.MemInc              = DMA_MINC_ENABLE;
    hdma_adc1.Init.PeriphDataAlignment = DMA_PDATAALIGN_HALFWORD;
    hdma_adc1.Init.MemDataAlignment    = DMA_MDATAALIGN_HALFWORD;
    hdma_adc1.Init.Mode                = DMA_CIRCULAR;
    hdma_adc1.Init.Priority            = DMA_PRIORITY_HIGH;
    hdma_adc1.Init.FIFOMode            = DMA_FIFOMODE_DISABLE;
    HAL_DMA_Init(&hdma_adc1);
    __HAL_LINKDMA(&hadc1, DMA_Handle, hdma_adc1);

    // ADC1: 12bit, 2채널 스캔, 연속 변환, DMA 순환
    hadc1.Instance                   = ADC1;
    hadc1.Init.ClockPrescaler        = ADC_CLOCK_SYNC_PCLK_DIV4;
    hadc1.Init.Resolution            = ADC_RESOLUTION_12B;
    hadc1.Init.ScanConvMode          = ENABLE;
    hadc1.Init.ContinuousConvMode    = ENABLE;
    hadc1.Init.DiscontinuousConvMode = DISABLE;
    hadc1.Init.ExternalTrigConvEdge  = ADC_EXTERNALTRIGCONVEDGE_NONE;
    hadc1.Init.DataAlign             = ADC_DATAALIGN_RIGHT;
    hadc1.Init.NbrOfConversion       = 2;
    hadc1.Init.DMAContinuousRequests = ENABLE;
    hadc1.Init.EOCSelection          = ADC_EOC_SINGLE_CONV;
    HAL_ADC_Init(&hadc1);

    ADC_ChannelConfTypeDef ch = {0};
    ch.SamplingTime = ADC_SAMPLETIME_84CYCLES;

    ch.Channel = ADC_CHANNEL_0;  // PA0 → speed
    ch.Rank    = 1;
    HAL_ADC_ConfigChannel(&hadc1, &ch);

    ch.Channel = ADC_CHANNEL_1;  // PA1 → current
    ch.Rank    = 2;
    HAL_ADC_ConfigChannel(&hadc1, &ch);

    HAL_NVIC_SetPriority(DMA2_Stream0_IRQn, 5, 0);
    HAL_NVIC_EnableIRQ(DMA2_Stream0_IRQn);

    HAL_ADC_Start_DMA(&hadc1, (uint32_t*)adc_buf, 2);
}

void HAL_ADC_ConvCpltCallback(ADC_HandleTypeDef *hadc)
{
    (void)hadc;
    g_monitor.speed_raw   = adc_buf[0];
    g_monitor.current_raw = adc_buf[1];
}

void DMA2_Stream0_IRQHandler(void)
{
    HAL_DMA_IRQHandler(&hdma_adc1);
}

float AdcMonitor_GetSpeedRpm(void)
{
    float v = (g_monitor.speed_raw / ADC_MAX) * ADC_FULL_SCALE_V;
    // 0~3.0V (= ESCON 0~5V 분압 후) → 0~MAX_RPM
    return (v / ADC_FULL_SCALE_V) * MONITOR_SPEED_MAX_RPM;
}

float AdcMonitor_GetCurrentA(void)
{
    float v = (g_monitor.current_raw / ADC_MAX) * ADC_FULL_SCALE_V;
    return (v / ADC_FULL_SCALE_V) * MONITOR_CURRENT_MAX_A;
}
```

스케일링 상수 (`MONITOR_SPEED_MAX_RPM`, `MONITOR_CURRENT_MAX_A`)는 ESCON Studio의 Analog Output 설정 화면에 표시된 풀스케일 값과 정확히 일치시켜야 한다. 여기 어긋나면 ADC 값은 잘 들어오는데 표시 rpm/A가 실측과 안 맞는다.

DMA Circular 모드로 잡아둬서 한 번 시작하면 영구히 두 채널을 갱신한다. 메인 루프 부담은 0이고, 콜백에서 raw 값만 `volatile` 변수로 옮긴다.

## 통합 메인 루프

명령 속도를 0 → 80% → 0으로 천천히 왕복시키면서 실제 rpm/전류를 100 ms마다 찍는 예제다. printf는 ITM이든 USART든 어디로 라우팅돼 있다고 가정한다.

```c
// main.c
#include "stm32f4xx_hal.h"
#include "motor_ctrl.h"
#include "adc_monitor.h"
#include <stdio.h>

void SystemClock_Config(void);   // CubeMX 생성분

int main(void)
{
    HAL_Init();
    SystemClock_Config();

    MotorCtrl_Init();
    AdcMonitor_Init();

    HAL_Delay(100);
    MotorCtrl_SetDir(MOTOR_DIR_CW);
    MotorCtrl_Enable();

    uint16_t cmd_x10 = 0;
    int8_t   step    = 5;     // 0.5%/step

    while (1)
    {
        MotorCtrl_SetSpeedPercentTenth(cmd_x10);

        float rpm = AdcMonitor_GetSpeedRpm();
        float amp = AdcMonitor_GetCurrentA();
        printf("cmd=%3u.%u%%  rpm=%6.1f  I=%5.2fA\r\n",
               cmd_x10 / 10, cmd_x10 % 10, rpm, amp);

        // 0% ↔ 80% 왕복
        if (cmd_x10 >= 800) step = -5;
        if (cmd_x10 == 0)   step = 5;
        cmd_x10 = (uint16_t)((int16_t)cmd_x10 + step);

        HAL_Delay(100);
    }
}
```

처음 띄울 때 무조건 `Enable` 켜기 전에 PWM이 정지(10%)인 걸 한 번 확인하고 넘어간다. `MotorCtrl_Init` 안에서 그렇게 만들어뒀지만, 디버거 붙여서 `__HAL_TIM_GET_COMPARE`로 100인지 한 번 보는 게 마음 편하다.

## 동작 확인 체크리스트

처음 돌릴 때 자주 막히는 지점들.

1. 모터가 안 돈다, ESCON LED는 초록 → Enable 신호 안 들어간 거다. ESCON Studio의 Monitor → I/O Monitor에서 DigIN 1이 High로 바뀌는지 본다.
2. 모터가 안 돈다, ESCON LED는 빨강 → 듀티가 10% 미만이거나 90% 초과다. 오실로로 PA8 찍어보면 바로 보인다. ARR/Prescaler 계산 실수가 제일 흔하다.
3. PWM은 잘 나오는데 ESCON이 듀티를 잘못 읽는다 → 5V TTL 임계점 문제. 케이블 짧게 하거나 74HCT04 끼우면 해결.
4. 속도 명령은 들어가는데 실제 rpm이 한참 안 맞는다 → ESCON Studio Set Value의 "Scaling" 페이지에서 풀스케일 rpm 다시 확인. 코드의 `MONITOR_SPEED_MAX_RPM`과 일치하는지.
5. ADC가 다 0 또는 다 풀스케일 → 분압 회로 확인. 오실로로 PA0/PA1 핀에서 실제 0~3.3V 신호가 보이는지.
6. ADC 값이 들쭉날쭉 → Signal GND와 STM32 GND가 단일점으로 묶여 있는지. 별도 그라운드면 노이즈 다 받는다.
7. Direction을 토글했더니 모터가 갑자기 반대로 풀스피드로 튕긴다 → Direction 바꾸기 전에 PWM을 10%(정지)로 내리고, 모터 정지 확인 후 방향 바꾸고, 다시 가속하는 시퀀스로 코드 작성. ESCON이 부드러운 감속을 보장해주긴 하지만 펌웨어 쪽에서도 보호 로직을 두는 게 맞다.

## 다음 편

다음은 ESCON Studio의 Data Recorder를 켜놓고 STM32에서 만든 명령 신호와 실제 모터 거동을 비교하는 글이다. 스텝 응답, 가감속 응답을 측정해서 응답성을 정량적으로 파악하고, Expert Tuning으로 PID를 손볼 준비를 한다.
