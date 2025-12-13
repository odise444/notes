# Ghidra로 STM32 부트로더 역분석한 썰 시리즈 기획

## 프로젝트 개요

- **목표**: 기존 제품의 CAN IAP 부트로더 역분석 및 복원
- **타겟**: STM32F103VE (512KB Flash, 64KB RAM)
- **도구**: Ghidra, Python, CAN 분석기
- **원본**: 200429.hex (16KB 부트로더)

---

## Part 1: 리버스 엔지니어링 입문편 ✅

| # | 제목 | 핵심 내용 | 상태 |
|---|------|----------|------|
| 1 | 왜 역분석을 하게 됐나 | 소스코드 분실, 현장 업데이트 필요 | ✅ 완료 |
| 2 | HEX 파일 구조 이해하기 | Intel HEX 포맷, 주소 계산 | ✅ 완료 |
| 3 | Ghidra 설치와 첫 로드 | ARM Cortex-M 설정, 메모리 맵 | ✅ 완료 |
| 4 | 디스어셈블리 vs 디컴파일 | 어셈블리 읽기, C 유사 코드 해석 | ✅ 완료 |

## Part 2: STM32 구조 분석편 ✅

| # | 제목 | 핵심 내용 | 상태 |
|---|------|----------|------|
| 5 | Vector Table 분석 | Reset_Handler 찾기, SP 초기값 | ✅ 완료 |
| 6 | 메모리 맵 추정하기 | Flash/RAM 영역, 주변장치 주소 | ✅ 완료 |
| 7 | 부트로더 경계 찾기 | 16KB 확인, 앱 시작 주소 | ✅ 완료 |
| 8 | 전역 변수 영역 분석 | .data, .bss, 구조체 추정 | ✅ 완료 |

## Part 3: 주변장치 역분석편 ✅

| # | 제목 | 핵심 내용 | 상태 |
|---|------|----------|------|
| 9 | RCC 설정 복원하기 | 클럭 트리, PLL 설정 | ✅ 완료 |
| 10 | GPIO 초기화 분석 | LED, 버튼 핀 찾기 | ✅ 완료 |
| 11 | CAN 초기화 역분석 | 500kbps 설정, 필터 구성 | ✅ 완료 |
| 12 | Flash 접근 함수 찾기 | Unlock, Erase, Program | ✅ 완료 |

## Part 4: CAN IAP 프로토콜 역분석편 ✅

| # | 제목 | 핵심 내용 | 상태 |
|---|------|----------|------|
| 13 | CAN 수신 핸들러 분석 | FIFO, 메시지 파싱, 인터럽트→플래그 | ✅ 완료 |
| 14 | 명령 코드 체계 파악 | 0x30/0x40 시리즈, 요청/응답 쌍 | ✅ 완료 |
| 15 | 상태 머신 복원 | 7개 상태, 상태 전이, 타임아웃 | ✅ 완료 |
| 16 | Connection Key 알고리즘 | FwChk ⊕ FwDate + bit mixing | ✅ 완료 |
| 17 | 페이지 전송 프로토콜 | 2KB/256프레임, 데이터 구분 | ✅ 완료 |
| 18 | CRC 검증 함수 분석 | STM32 HW CRC-32 (0x04C11DB7) | ✅ 완료 |

## Part 5: 핵심 로직 역분석편

| # | 제목 | 핵심 내용 | 상태 |
|---|------|----------|------|
| 19 | 부트 진입 조건 분석 | GPIO 체크, 타임아웃, 매직 메시지 | |
| 20 | 앱 유효성 검사 | Stack Pointer 검증, CRC | |
| 21 | 앱 점프 코드 분석 | VTOR, MSP, Reset_Handler | |
| 22 | 버퍼 → 앱 복사 로직 | 더블 버퍼, 플래시 프로그래밍 | |

## Part 6: 삽질 & 트러블슈팅편

| # | 제목 | 핵심 내용 | 상태 |
|---|------|----------|------|
| 23 | Thumb 모드 함정 | 주소 +1, BX LR 이해 | |
| 24 | 구조체 크기 추정 삽질 | 패딩, 정렬, sizeof | |
| 25 | 인라인 함수 vs 매크로 | 최적화된 코드 해석 | |
| 26 | 컴파일러 최적화 패턴 | -O2 특유의 코드 변환 | |

## Part 7: 소스코드 복원편

| # | 제목 | 핵심 내용 | 상태 |
|---|------|----------|------|
| 27 | main() 함수 복원 | 초기화 순서, 메인 루프 | |
| 28 | CAN IAP 모듈 복원 | 프로토콜 핸들러 재구현 | |
| 29 | Flash 드라이버 복원 | HAL vs 레지스터 직접 접근 | |
| 30 | 빌드하고 비교하기 | 바이너리 diff, 기능 검증 | |

## Part 8: 검증 & 활용편

| # | 제목 | 핵심 내용 | 상태 |
|---|------|----------|------|
| 31 | CAN 스니핑으로 프로토콜 검증 | PCAN, candump | |
| 32 | Python 업로더 만들기 | python-can, IAP 클라이언트 | |
| 33 | 실제 장비에서 테스트 | 펌웨어 업로드, 롤백 확인 | |
| 34 | 개선된 부트로더 만들기 | 보안 강화, 기능 추가 | |

## 번외편

| # | 제목 | 핵심 내용 | 상태 |
|---|------|----------|------|
| B1 | Ghidra 스크립트 활용 | Python/Java 자동화 | |
| B2 | IDA Pro vs Ghidra 비교 | 장단점, 선택 기준 | |
| B3 | 법적 고려사항 | 역분석 합법성, 라이선스 | |
| B4 | 난독화된 펌웨어 분석 | 안티 리버싱 우회 | |

## 추가 가이드

| 제목 | 핵심 내용 | 상태 |
|------|----------|------|
| Ghidra 빠른 시작 가이드 | 설치, HEX→BIN, 기본 사용법 | ✅ 완료 |

---

## 진행 현황

```
Part 1: 입문편           [████████████] 4/4  ✅
Part 2: STM32 구조       [████████████] 4/4  ✅
Part 3: 주변장치         [████████████] 4/4  ✅
Part 4: CAN IAP 프로토콜 [████████████] 6/6  ✅
Part 5: 핵심 로직        [            ] 0/4
Part 6: 삽질 & 트러블    [            ] 0/4
Part 7: 소스코드 복원    [            ] 0/4
Part 8: 검증 & 활용      [            ] 0/4
번외편                   [            ] 0/4
추가 가이드              [████████████] 1/1  ✅

총 진행: 19/39 (49%)
```

---

## 분석 대상 요약

### 200429.hex 부트로더 스펙

| 항목 | 값 |
|------|-----|
| MCU | STM32F103VE |
| 부트로더 크기 | 16KB (0x08000000 ~ 0x08003FFF) |
| 버퍼 영역 | 254KB (0x08004000 ~ 0x080427FF) |
| 앱 시작 주소 | 0x08042800 |
| 앱 크기 | 246KB |
| CAN 속도 | 500kbps |
| CAN ID (PC→BMS) | 0x5FF |
| CAN ID (BMS→PC) | 0x5FE |

### 역분석으로 밝혀낸 메모리 맵

```
Flash (512KB)
┌─────────────────────┐ 0x08080000
│                     │
│    Application      │ 246KB
│   0x08042800 ~      │
│                     │
├─────────────────────┤ 0x08042800
│                     │
│    Buffer           │ 254KB
│   0x08004000 ~      │
│                     │
├─────────────────────┤ 0x08004000
│                     │
│    Bootloader       │ 16KB
│   0x08000000 ~      │
│                     │
└─────────────────────┘ 0x08000000

RAM (64KB)
┌─────────────────────┐ 0x20010000
│      Stack          │ ~44KB
├─────────────────────┤ 0x20005218 (Initial SP)
│    Heap / 여유      │
├─────────────────────┤ 0x20001000
│      .bss           │ ~4KB
├─────────────────────┤ 0x20000100
│      .data          │ 256B
└─────────────────────┘ 0x20000000
```

### 역분석으로 밝혀낸 프로토콜 (Part 4 완료)

```
PC → BMS (0x5FF)          BMS → PC (0x5FE)
─────────────────         ─────────────────
0x30: Connection 요청  →  0x40: FwChk + FwDate
0x31: Calc 값 전송     →  0x41: 인증 성공
                       ←  0x42: 크기 요청
0x32: 크기 응답        →
                       ←  0x43: 페이지 요청
0x33: 페이지 시작      →
[data]: 페이지 데이터  →  (256 프레임/페이지)
0x34: 페이지 끝        →
                       ←  0x45: 검증 시작
0x36: CRC 결과         →
                       ←  0x46: 완료
```

### Connection Key 알고리즘 (Part 4에서 확정)

```c
uint32_t IAP_CalcKey(void) {
    uint32_t fw_chk = g_fw_check;
    uint32_t fw_date = g_fw_date[0] | (g_fw_date[1] << 8) | (g_fw_date[2] << 16);
    
    uint32_t result = fw_chk ^ fw_date;
    result ^= (result >> 16);
    result ^= (result >> 8);
    
    return result;
}
```

### Part 3~4에서 밝혀낸 내용

| 항목 | 발견 내용 |
|------|----------|
| 클럭 | HSE 8MHz × PLL 9 = 72MHz |
| APB1/APB2 | 36MHz / 72MHz |
| CAN TX/RX | PA12 / PA11 |
| LED | PB0, PB1, PC13 |
| 버튼 | PB2 (Active Low) |
| Flash Key | 0x45670123, 0xCDEF89AB |
| 상태 머신 | 7개 상태, 10초 타임아웃 |
| CRC | STM32 HW CRC-32 (0x04C11DB7) |
| 페이지 크기 | 2KB (256 CAN 프레임) |

---

## Ghidra 설정 가이드

### ARM Cortex-M 로드 설정

```
Language: ARM:LE:32:Cortex
Compiler: default

Memory Map:
- Flash: 0x08000000, 0x80000 (512KB)
- SRAM:  0x20000000, 0x10000 (64KB)
- Periph: 0x40000000, 0x30000

Base Address: 0x08000000
```

### 유용한 Ghidra 단축키

| 단축키 | 기능 |
|--------|------|
| G | Go to address |
| D | Disassemble |
| F | Create function |
| L | Label (이름 지정) |
| T | Set data type |
| ; | Comment |
| Ctrl+Shift+E | Export decompiled |

---

## 작성된 파일 목록

```
content/posts/Ghidra/
├── ghidra-stm32-quick-start.md   # 빠른 시작 가이드 ✅
├── ghidra-stm32-re-01.md         # #1 왜 역분석을 하게 됐나 ✅
├── ghidra-stm32-re-02.md         # #2 HEX 파일 구조 ✅
├── ghidra-stm32-re-03.md         # #3 Ghidra 설치와 첫 로드 ✅
├── ghidra-stm32-re-04.md         # #4 디스어셈블리 vs 디컴파일 ✅
├── ghidra-stm32-re-05.md         # #5 Vector Table 분석 ✅
├── ghidra-stm32-re-06.md         # #6 메모리 맵 추정하기 ✅
├── ghidra-stm32-re-07.md         # #7 부트로더 경계 찾기 ✅
├── ghidra-stm32-re-08.md         # #8 전역 변수 영역 분석 ✅
├── ghidra-stm32-re-09.md         # #9 RCC 설정 복원하기 ✅
├── ghidra-stm32-re-10.md         # #10 GPIO 초기화 분석 ✅
├── ghidra-stm32-re-11.md         # #11 CAN 초기화 역분석 ✅
├── ghidra-stm32-re-12.md         # #12 Flash 접근 함수 찾기 ✅
├── ghidra-stm32-re-13.md         # #13 CAN 수신 핸들러 분석 ✅
├── ghidra-stm32-re-14.md         # #14 명령 코드 체계 파악 ✅
├── ghidra-stm32-re-15.md         # #15 상태 머신 복원 ✅
├── ghidra-stm32-re-16.md         # #16 Connection Key 알고리즘 ✅
├── ghidra-stm32-re-17.md         # #17 페이지 전송 프로토콜 ✅
└── ghidra-stm32-re-18.md         # #18 CRC 검증 함수 분석 ✅
```

---

## 참고 자료

- [Ghidra 공식 문서](https://ghidra-sre.org/)
- [ARM Cortex-M3 Technical Reference](https://developer.arm.com/documentation/ddi0337/latest)
- [STM32F103 Reference Manual (RM0008)](https://www.st.com/resource/en/reference_manual/rm0008.pdf)
- [Intel HEX Format](https://en.wikipedia.org/wiki/Intel_HEX)

---

## 메모

- 총 39개 글 (본편 34 + 번외 4 + 가이드 1)
- Part 1~4 완료 (19개, 49%)
- 다음 작성: Part 5 (#19 부트 진입 조건 분석)
- 실제 역분석 경험 기반
- 보안/법적 주의사항 포함
- Ghidra 초보자도 따라할 수 있게 구성
