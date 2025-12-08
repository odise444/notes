# STM32 커스텀 부트로더 개발기 시리즈 기획

## 프로젝트 개요

- **목표**: STM32 CAN/UART 부트로더 직접 구현
- **MCU**: STM32F103 (기준), STM32F4/G4 확장
- **통신**: CAN 500kbps, UART 115200
- **도구**: STM32CubeIDE, PlatformIO, Ghidra (역분석)

---

## Part 1: 부트로더 기초편

| # | 제목 | 핵심 내용 | 상태 |
|---|------|----------|------|
| 1 | 왜 커스텀 부트로더인가? | 현장 업데이트, 다중 노드, 보안 | |
| 2 | STM32 메모리 맵 이해하기 | Flash 구조, 섹터/페이지, Option Bytes | |
| 3 | 부트로더 vs 앱 영역 분리 | 링커 스크립트, 시작 주소 설정 | |
| 4 | Vector Table 재배치 | SCB->VTOR, 앱 점프 메커니즘 | |

## Part 2: 앱 점프 구현편

| # | 제목 | 핵심 내용 | 상태 |
|---|------|----------|------|
| 5 | 앱 유효성 검사 | Stack Pointer 검증, Magic Number | |
| 6 | 앱으로 점프하기 | MSP 설정, PC 점프, 인터럽트 정리 | |
| 7 | 부트 진입 조건 설계 | GPIO 핀, 타임아웃, CAN 매직 메시지 | |
| 8 | 소프트웨어 리셋으로 부트 진입 | Backup Register, RAM 플래그 | |

## Part 3: Flash 프로그래밍편

| # | 제목 | 핵심 내용 | 상태 |
|---|------|----------|------|
| 9 | Flash Unlock/Lock | FLASH_CR, 키 시퀀스 | |
| 10 | Page Erase 구현 | 페이지 크기(1KB/2KB), 대기 시간 | |
| 11 | HalfWord/Word Program | 프로그래밍 순서, 검증 | |
| 12 | Flash 에러 처리 | PGERR, WRPRTERR, 타임아웃 | |

## Part 4: CAN IAP 프로토콜편

| # | 제목 | 핵심 내용 | 상태 |
|---|------|----------|------|
| 13 | CAN 부트로더 프로토콜 설계 | 명령 코드, 응답 코드, 상태 머신 | |
| 14 | Connection & Authentication | Seed-Key, FwChk XOR FwDate | |
| 15 | 펌웨어 크기/페이지 협상 | 크기 전송, 페이지 수 계산 | |
| 16 | 페이지 데이터 수신 | 8바이트씩 수신, 버퍼 관리 | |
| 17 | CRC 검증 구현 | STM32 내장 CRC32, 소프트웨어 CRC | |
| 18 | 버퍼 → 앱 영역 복사 | 이중 버퍼, 롤백 전략 | |

## Part 5: UART IAP 프로토콜편

| # | 제목 | 핵심 내용 | 상태 |
|---|------|----------|------|
| 19 | UART 부트로더 기초 | 간단한 프로토콜, 매직 바이트 | |
| 20 | XMODEM/YMODEM 구현 | 표준 프로토콜, 패킷 구조 | |
| 21 | 듀얼 인터페이스 (CAN + UART) | 통합 상태 머신 | |

## Part 6: 보안편

| # | 제목 | 핵심 내용 | 상태 |
|---|------|----------|------|
| 22 | Read Protection (RDP) | Level 0/1/2, Option Bytes | |
| 23 | Write Protection | 섹터별 보호, 부트로더 영역 잠금 | |
| 24 | 펌웨어 암호화 | AES-128, 키 관리 | |
| 25 | Secure Boot 개념 | 서명 검증, 인증서 체인 | |

## Part 7: PC 업로더 구현편

| # | 제목 | 핵심 내용 | 상태 |
|---|------|----------|------|
| 26 | Python CAN 업로더 | python-can, PCAN/슬CAN 어댑터 | |
| 27 | Python UART 업로더 | pyserial, 프로그레스 바 | |
| 28 | GUI 업로더 (PyQt/Tkinter) | 파일 선택, 진행 상황, 로그 | |
| 29 | CLI 도구 완성 | argparse, 자동화 스크립트 | |

## Part 8: 역분석/복원편

| # | 제목 | 핵심 내용 | 상태 |
|---|------|----------|------|
| 30 | Ghidra로 부트로더 분석하기 | 바이너리 로드, ARM Cortex-M 설정 | |
| 31 | CAN 프로토콜 역분석 | 스니핑, 메시지 해석 | |
| 32 | 기존 부트로더 복원하기 | 디컴파일, 재구현 | |

## Part 9: 실전/양산편

| # | 제목 | 핵심 내용 | 상태 |
|---|------|----------|------|
| 33 | 다중 노드 업데이트 | 순차 업데이트, 브로드캐스트 | |
| 34 | 업데이트 실패 복구 | 롤백, 골든 이미지 | |
| 35 | 양산 라인 적용 | 지그 설계, 자동화 | |

## 번외편

| # | 제목 | 핵심 내용 | 상태 |
|---|------|----------|------|
| B1 | STM32 내장 부트로더 분석 | System Memory, DFU 모드 | |
| B2 | 오픈소스 부트로더 비교 | mcuboot, OpenBLT, stm32-can-bootloader | |
| B3 | UDS (ISO 14229) 적용 | 자동차 표준 진단 프로토콜 | |
| B4 | CANopen 부트로더 | 산업용 표준 | |

---

## 메모리 맵 예시 (STM32F103VE - 512KB)

```
0x08080000 ┌─────────────────────┐
           │                     │
           │   App (246KB)       │
           │   0x08042800~       │
           │                     │
0x08042800 ├─────────────────────┤
           │   Buffer (254KB)    │
           │   (펌웨어 수신용)    │
0x08004000 ├─────────────────────┤
           │   Bootloader (16KB) │
0x08000000 └─────────────────────┘
```

## CAN IAP 프로토콜 요약

| 방향 | CAN ID | 명령 | 설명 |
|------|--------|------|------|
| PC→BMS | 0x5FF | 0x30 | Connection Key 요청 |
| BMS→PC | 0x5FE | 0x40 | FwChk + FwDate 응답 |
| PC→BMS | 0x5FF | 0x31 | Calc 값 전송 (인증) |
| BMS→PC | 0x5FE | 0x41 | 인증 성공 |
| BMS→PC | 0x5FE | 0x42 | 펌웨어 크기 요청 |
| PC→BMS | 0x5FF | 0x32 | 크기 응답 |
| BMS→PC | 0x5FE | 0x43 | 페이지 데이터 요청 |
| PC→BMS | 0x5FF | 0x33 | 페이지 시작 |
| PC→BMS | 0x5FF | data | 페이지 데이터 (N×8바이트) |
| PC→BMS | 0x5FF | 0x34 | 페이지 끝 |
| BMS→PC | 0x5FE | 0x45 | 검증 시작 |
| BMS→PC | 0x5FE | 0x46 | 검증 완료 |

---

## 참고 자료

- [STM32F10x Flash Programming Manual (PM0075)](https://www.st.com/resource/en/programming_manual/pm0075-stm32f10xxx-flash-memory-microcontrollers-stmicroelectronics.pdf)
- [STM32 System Memory Boot Mode (AN2606)](https://www.st.com/resource/en/application_note/an2606-stm32-microcontroller-system-memory-boot-mode-stmicroelectronics.pdf)
- [matejx/stm32f1-CAN-bootloader](https://github.com/matejx/stm32f1-CAN-bootloader)
- [jsphuebner/stm32-CANBootloader](https://github.com/jsphuebner/stm32-CANBootloader)
- [OpenBLT Bootloader](https://www.feaser.com/openblt/)

---

## 메모

- 총 39개 글 (본편 35 + 번외 4)
- BMS 프로젝트와 연계 가능 (CAN IAP)
- 임베디드 개발자 타겟
- 실무 경험 기반 삽질 공유
