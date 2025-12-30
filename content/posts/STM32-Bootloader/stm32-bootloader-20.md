---
title: "STM32 부트로더 개발기 #20 - 트러블슈팅 모음"
date: 2024-12-17
tags: ["STM32", "Bootloader", "트러블슈팅", "디버깅"]
categories: ["개발기"]
series: ["STM32 부트로더 개발기"]
summary: "부트로더 만들면서 겪은 삽질들."
---

부트로더 개발하면서 만난 문제들 정리.

---

## 앱으로 점프 안 됨

### 증상
jump_to_app() 호출 후 HardFault.

### 원인 1: VTOR 설정 안 함
```c
// 빠뜨림
SCB->VTOR = APP_ADDR;
```

### 원인 2: 앱 시작 주소 틀림
링커 스크립트와 부트로더의 APP_ADDR 불일치.

### 원인 3: Thumb 비트
Reset Handler 주소에 `& ~1` 하면 안 됨.

---

## Flash 쓰기 안 됨

### 증상
flash_program_halfword() 실패.

### 원인 1: Unlock 안 함
```c
flash_unlock();  // 빠뜨림
```

### 원인 2: 이미 값 있음
지우지 않고 쓰기 시도.
```c
// 쓰기 전 삭제 필수
flash_erase_page(addr);
```

### 원인 3: 쓰기 보호
Option Bytes 확인.

---

## CAN 메시지 안 옴

### 증상
can_receive() 타임아웃.

### 원인 1: 필터 설정
```c
// 모든 메시지 받기
sFilterConfig.FilterIdHigh = 0x0000;
sFilterConfig.FilterIdLow = 0x0000;
sFilterConfig.FilterMaskIdHigh = 0x0000;
sFilterConfig.FilterMaskIdLow = 0x0000;
sFilterConfig.FilterFIFOAssignment = CAN_RX_FIFO0;
sFilterConfig.FilterActivation = ENABLE;
```

### 원인 2: Bitrate 불일치
양쪽 500kbps인지 확인.

### 원인 3: 터미네이션
CAN 버스 양 끝에 120Ω 저항.

---

## 업로드 중간에 멈춤

### 증상
몇 페이지 전송 후 응답 없음.

### 원인 1: PC 너무 빠름
```python
time.sleep(0.001)  # 패킷 사이 딜레이
```

### 원인 2: BMS 처리 못 따라감
Flash 쓰기 중 다음 패킷 와서 놓침.

### 원인 3: CAN 버퍼 오버플로우
FIFO 빨리 비워야 함.

---

## CRC 불일치

### 증상
PAGE_END 후 CRC 에러.

### 원인 1: 엔디안
```c
// Little endian으로 맞추기
uint32_t crc = data[0] | (data[1]<<8) | (data[2]<<16) | (data[3]<<24);
```

### 원인 2: 다항식 다름
STM32 내장 CRC vs IEEE CRC32 다름.

### 원인 3: 초기값
```c
iap.crc = 0xFFFFFFFF;  // 초기값 확인
```

---

## 리셋 후 앱 실행 안 됨

### 증상
업로드 성공, 리셋하면 부트로더로 감.

### 원인 1: 앱 유효성 검사 실패
```c
// SP, Reset Handler 주소 확인
printf("SP: 0x%08X\n", *(uint32_t*)APP_ADDR);
printf("Reset: 0x%08X\n", *(uint32_t*)(APP_ADDR+4));
```

### 원인 2: Buffer → App 복사 안 함
수신만 하고 복사 안 했을 때.

### 원인 3: 부트 조건에 걸림
GPIO 핀 상태, RAM 플래그 확인.

---

## 업로드 후 앱 이상 동작

### 증상
앱 실행되는데 기능 이상.

### 원인 1: 인터럽트 정리 안 함
```c
// 점프 전 정리
__disable_irq();
SysTick->CTRL = 0;
for (int i = 0; i < 8; i++) {
    NVIC->ICER[i] = 0xFFFFFFFF;
    NVIC->ICPR[i] = 0xFFFFFFFF;
}
```

### 원인 2: 주변장치 상태
부트로더에서 쓴 주변장치 DeInit 필요할 수 있음.

---

## 디버깅 체크리스트

| 확인 항목 | 방법 |
|----------|------|
| APP_ADDR 일치 | 링커 스크립트 vs 부트로더 매크로 |
| Flash 내용 | ST-Link로 메모리 뷰 |
| CAN 통신 | PCAN-View 스니핑 |
| VTOR 값 | 디버거로 SCB->VTOR 확인 |
| SP 값 | APP_ADDR 첫 4바이트 |
| Reset Handler | APP_ADDR+4 값 |

---

## 시리즈 끝

20개 글에 걸쳐 STM32 CAN 부트로더 개발 과정을 정리했다.

요약:
- 메모리 맵과 링커 스크립트
- 앱 유효성 검사, VTOR, 점프
- Flash Unlock/Erase/Program
- CAN IAP 프로토콜과 상태 머신
- Python 업로더
- 보안과 트러블슈팅

Ghidra 시리즈가 "역분석"이었다면, 이 시리즈는 "처음부터 만들기".

---

[#1 - 왜 커스텀 부트로더인가](/posts/stm32-bootloader/stm32-bootloader-01/)로 돌아가기
