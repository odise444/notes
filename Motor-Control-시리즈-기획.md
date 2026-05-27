# Motor-Control 시리즈 기획

## 개요

| 항목 | 내용 |
|------|------|
| 시리즈명 | Motor-Control |
| 주제 | BLDC 모터 제어, 서보 드라이버, 임베디드 |
| 대상 독자 | 임베디드 엔지니어, 모터 제어 입문자 |
| 폴더 | content/posts/Motor-Control/ |

## 글 목록

| # | 제목 | 파일명 | 날짜 | 상태 |
|---|------|--------|------|------|
| 1 | maxon ESCON 50/5로 국산 BLDC 모터 제어하기 | motor-control-01.md | 2026-04-24 | ✅ 완료 |
| 2 | STM32 PWM 속도 제어 펌웨어 | motor-control-02.md | 2026-05-27 | ✅ 완료 |
| 3 | ESCON Data Recorder로 응답 측정하고 튜닝하기 | motor-control-03.md | 2026-05-28 | ✅ 완료 |

**시리즈 1차 마무리.** 후속 주제는 실제 모터 제어 건이 들어오는 시점에 결정.

## #1 글 상세

### maxon ESCON 50/5로 국산 BLDC 모터 제어하기

**하드웨어 구성**:
- 서보 컨트롤러: maxon ESCON 50/5 (#409510)
- BLDC 모터: 모터뱅크 BL57101E1K (24V, 200W)
- 감속기: 모터뱅크 PGH56S (1:16)
- 제어 MCU: STM32 (계획)

**핵심 내용**:
1. Startup Wizard 단계별 설정 (17개 이미지)
2. Diagnostics 트러블슈팅 (4건의 Fail 사례)
3. Auto Tuning 결과 분석
4. 국산 BLDC 사양서 함정 (극수, 엔코더 분해능)

**발견된 사양서 오류**:
| 항목 | 사양서 | 실측 |
|------|--------|------|
| Pole Pairs | 4 | 2 |
| Encoder Resolution | 1000 CPR | 500 CPR |

**이미지**: 27개 (`static/imgs/motor-control-01/`)

## #2 글 상세

### STM32 PWM 속도 제어 펌웨어

**MCU / 핀맵**:
- Nucleo-F411RE (STM32F411RE) 기준, HAL 라이브러리
- PA0 → ESCON DI1 (TIM2_CH1 PWM)
- PB0 → ESCON DI2 (Enable)
- PB1 → ESCON DI3 (Direction)
- PA1 → ESCON AO1 (ADC1_IN1, Speed)
- PA4 → ESCON AO2 (ADC1_IN4, Current)

**핵심 내용**:
1. PWM 1 kHz 결정 (ARR=999, PSC=83), 부팅 시 Pulse=100(10%)로 안전 시작
2. Duty 10~90% ↔ 0~3000 rpm 정수 매핑 (FPU 없이 처리)
3. `motor.h`/`motor.c` 모듈로 인터페이스 분리
4. ADC1 멀티채널 + DMA Circular 모드로 백그라운드 갱신
5. EWMA 평활화 (표시용/제어용 분리 원칙)
6. 시동 시퀀스, watchdog, hard/soft stop, Direction 변경 규칙
7. 디버깅 팁 6종

**파일**: `content/posts/Motor-Control/motor-control-02.md`

## #3 글 상세

### ESCON Data Recorder로 응답 측정하고 튜닝하기

**도구**:
- ESCON Studio → Tuning → Data Recording
- 4채널, 0.1~100 ms 가변 샘플 주기, 최대 30초 캡처
- CSV/PNG/BMP export

**핵심 내용**:
1. 채널 4개 선택 전략 (속도 루프 / 전류 루프 / PWM 검증 세 가지 조합)
2. 트리거 모드 4종 + Pre-trigger 활용
3. 무부하 스텝 응답 측정 + 응답 지표 4종 정의
   - Rise time (10~90%)
   - Overshoot
   - Settling time (±2% 밴드)
   - Steady-state current
4. 부하 변동 응답 측정 + Speed dip + Voltage saturation 해석
5. CSV export + Python 분석 자동화 (rise/overshoot/settling 계산)
6. Expert Tuning 파라미터 4종 (Current/Speed × P/I gain)과 응답 지표 매핑
7. STM32 측 시험 시퀀스 자동화 (state machine)
8. 함정 5가지 (USB 끊김, Pre-trigger, 샘플 주기, Speed Ramp, AO Averaged)

**파일**: `content/posts/Motor-Control/motor-control-03.md`

## 향후 후속 주제 (정해진 일정 없음)

- UART/CAN으로 속도·전류 로깅 외부 노트북 송출 (STM32 쪽)
- ADC 노이즈 감소: RC 필터, 차폐, GND 토폴로지
- E-stop 시퀀스 구현 (DI/O4 Ready + 외부 인터럽트)
- Position Controller 모드 + 엔코더 위치 제어
- maxon EPOS 시리즈 비교 (CANopen, EtherCAT)
- 국산 BLDC + 국산 드라이버 조합 케이스 스터디

## 태그

`모터제어`, `BLDC`, `maxon`, `ESCON`, `임베디드`, `STM32`, `PWM`, `HAL`, `ADC`, `DMA`, `Data Recorder`, `튜닝`

## 참고 자료

- [maxon ESCON Studio 공식 문서](https://www.maxongroup.com/maxon/view/content/escon-detailsite)
- [ESCON 50/5 Hardware Reference](https://www.maxongroup.com/medias/sys_master/root/8837504466974/409510-ESCON-50-5-Hardware-Reference-En.pdf)
- [모터뱅크](https://www.motorbank.kr/)
