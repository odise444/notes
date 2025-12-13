---
title: "Ghidra로 STM32 부트로더 역분석한 썰 #17 - 페이지 전송 프로토콜"
date: 2024-12-13
draft: false
tags: ["Ghidra", "STM32", "리버스엔지니어링", "역분석", "CAN", "Flash"]
categories: ["역분석"]
summary: "2KB 페이지를 8바이트 CAN 프레임으로 어떻게 보낼까? 페이지 전송 프로토콜 상세 분석."
---

## 지난 글 요약

[지난 글](/posts/ghidra/ghidra-stm32-re-16/)에서 Connection Key 알고리즘을 분석했다. XOR + bit mixing. 이제 **페이지 전송 프로토콜**을 상세히 분석하자.

## 문제: 2KB를 8바이트로?

```
Flash 페이지: 2048 bytes
CAN 프레임:   8 bytes max

2048 / 8 = 256 프레임 필요!
```

## 페이지 전송 흐름

```
PC                              BMS
│                                │
│◄────── 0x43 [page_num] ────────│  Page Request
│                                │
│─────── 0x33 [page_num] ───────►│  Page Start
│                                │
│─────── [data 0-7] ────────────►│  Data Frame 0
│─────── [data 8-15] ───────────►│  Data Frame 1
│─────── [data 16-23] ──────────►│  Data Frame 2
│           ...                  │
│─────── [data 2040-2047] ──────►│  Data Frame 255
│                                │
│─────── 0x34 ──────────────────►│  Page End
│                                │
│◄────── 0x43 [page+1] ──────────│  Next Page (or 0x45)
```

## 데이터 프레임 구분

**문제**: 데이터 프레임에 명령 코드가 없다!

### 해결책 분석

`FUN_08003300`에서:

```c
void FUN_08003300(void) {
    uint8_t cmd = *(uint8_t *)0x20000200;
    
    // 명령 코드 범위 체크
    if (cmd >= 0x30 && cmd <= 0x3F) {
        // 명령 처리
        switch (cmd) {
            case 0x30: ... break;
            // ...
        }
    } else {
        // 데이터 프레임으로 처리
        if (*(uint8_t *)0x20000300 == 4) {  // STATE_PAGE_DATA
            FUN_08003E00((uint8_t *)0x20000208, 
                         *(uint32_t *)0x20000204);
        }
    }
}
```

**규칙**: 첫 바이트가 0x30~0x3F가 아니면 → 데이터 프레임!

## 데이터 프레임 처리

`FUN_08003E00` 분석:

```c
void FUN_08003E00(uint8_t *data, uint32_t len) {
    uint32_t offset = *(uint32_t *)0x20000314;  // g_page_offset
    
    // 오버플로 체크
    if (offset + len > 2048) {
        // 에러: 페이지 크기 초과
        *(uint8_t *)0x20000300 = 0;  // STATE_IDLE
        return;
    }
    
    // 버퍼에 복사
    uint8_t *buffer = (uint8_t *)0x20000400;  // g_page_buffer
    
    for (uint32_t i = 0; i < len; i++) {
        buffer[offset + i] = data[i];
    }
    
    // 오프셋 업데이트
    *(uint32_t *)0x20000314 = offset + len;
}
```

## 페이지 버퍼 구조

```
RAM 레이아웃:
─────────────────────────────
0x20000300: g_iap_state (1 byte)
0x20000304: g_timeout (4 bytes)
0x20000308: g_fw_size (4 bytes)
0x2000030C: g_total_pages (4 bytes)
0x20000310: g_current_page (4 bytes)
0x20000314: g_page_offset (4 bytes)
...
0x20000400: g_page_buffer[2048]
0x20000C00: (버퍼 끝)
```

## 페이지 시작 처리

```c
void FUN_08003800(uint8_t *data) {
    if (*(uint8_t *)0x20000300 != 3) {  // STATE_RECEIVING
        return;
    }
    
    uint8_t page_num = data[1];
    
    // 페이지 번호 검증
    if (page_num != *(uint32_t *)0x20000310) {
        // 순서 오류 - 무시하거나 에러
        return;
    }
    
    // 버퍼 초기화 (0xFF = Flash 기본값)
    uint8_t *buffer = (uint8_t *)0x20000400;
    for (int i = 0; i < 2048; i++) {
        buffer[i] = 0xFF;
    }
    
    // 오프셋 리셋
    *(uint32_t *)0x20000314 = 0;
    
    // 상태 변경
    *(uint8_t *)0x20000300 = 4;  // STATE_PAGE_DATA
}
```

## 페이지 끝 처리

```c
void FUN_08003900(void) {
    if (*(uint8_t *)0x20000300 != 4) {
        return;
    }
    
    uint32_t current_page = *(uint32_t *)0x20000310;
    uint32_t total_pages = *(uint32_t *)0x2000030C;
    
    // Flash에 쓰기
    uint32_t flash_addr = 0x08004000 + current_page * 2048;
    FUN_08002E00(flash_addr, (uint8_t *)0x20000400);
    
    // 다음 페이지
    current_page++;
    *(uint32_t *)0x20000310 = current_page;
    
    if (current_page >= total_pages) {
        // 모든 페이지 완료 → 검증 단계
        uint8_t response[1] = {0x45};  // VERIFY_START
        FUN_08003C00(0x5FE, response, 1);
        
        *(uint8_t *)0x20000300 = 5;  // STATE_VERIFYING
    } else {
        // 다음 페이지 요청
        uint8_t response[2] = {0x43, (uint8_t)current_page};
        FUN_08003C00(0x5FE, response, 2);
        
        *(uint8_t *)0x20000300 = 3;  // STATE_RECEIVING
    }
}
```

## 마지막 페이지 처리

마지막 페이지는 2048바이트 미만일 수 있음:

```c
// 크기 계산 예시
fw_size = 10000 bytes

total_pages = (10000 + 2047) / 2048 = 5 pages

Page 0: 2048 bytes (offset 0~2047)
Page 1: 2048 bytes (offset 2048~4095)
Page 2: 2048 bytes (offset 4096~6143)
Page 3: 2048 bytes (offset 6144~8191)
Page 4: 1808 bytes (offset 8192~9999)  ← 마지막
```

부트로더에서:
```c
// 마지막 페이지는 나머지 바이트만 수신
// 나머지 영역은 0xFF로 채워짐 (버퍼 초기화 시)
```

## 복원된 PC측 코드

```python
# page_transfer.py

PAGE_SIZE = 2048
CAN_DATA_SIZE = 8

def send_firmware(can_if, firmware: bytes):
    """펌웨어 전송"""
    
    total_pages = (len(firmware) + PAGE_SIZE - 1) // PAGE_SIZE
    
    for page_num in range(total_pages):
        # 페이지 요청 대기
        msg = can_if.receive(timeout=5.0)
        if msg is None or msg.data[0] != 0x43:
            raise Exception(f"Expected page request, got {msg}")
        
        if msg.data[1] != page_num:
            raise Exception(f"Page mismatch: expected {page_num}, got {msg.data[1]}")
        
        # 페이지 시작
        can_if.send(0x5FF, bytes([0x33, page_num]))
        time.sleep(0.001)  # 짧은 딜레이
        
        # 페이지 데이터 추출
        start = page_num * PAGE_SIZE
        end = min(start + PAGE_SIZE, len(firmware))
        page_data = firmware[start:end]
        
        # 2048바이트 미만이면 0xFF로 패딩
        if len(page_data) < PAGE_SIZE:
            page_data += bytes([0xFF] * (PAGE_SIZE - len(page_data)))
        
        # 데이터 프레임 전송 (256개)
        for i in range(0, PAGE_SIZE, CAN_DATA_SIZE):
            chunk = page_data[i:i+CAN_DATA_SIZE]
            can_if.send(0x5FF, chunk)
            time.sleep(0.0005)  # 0.5ms 간격
        
        # 페이지 끝
        can_if.send(0x5FF, bytes([0x34]))
        
        print(f"Page {page_num}/{total_pages-1} sent")
    
    # 검증 시작 대기
    msg = can_if.receive(timeout=5.0)
    if msg is None or msg.data[0] != 0x45:
        raise Exception("Verify start not received")
    
    print("All pages sent, verification starting...")
```

## 전송 속도 계산

```
CAN 500kbps:
- 1 비트 = 2μs
- 8바이트 + 오버헤드 ≈ 120비트 ≈ 240μs/프레임

2KB 페이지 전송:
- 256 프레임 × 240μs = 61.4ms
- + 0.5ms 딜레이 × 256 = 128ms
- 총 ~190ms/페이지

100KB 펌웨어 (50페이지):
- 50 × 190ms = 9.5초

실제 측정: 약 10~15초 (재전송, 처리 시간 포함)
```

## 데이터 무결성

### 프레임 손실 감지

CAN은 하드웨어 CRC가 있지만, 프레임 손실은 별도 처리:

```c
// 수신 바이트 수 체크
void FUN_08003900(void) {
    uint32_t received = *(uint32_t *)0x20000314;
    
    // 정확히 2048바이트 수신했는지 확인
    if (received != 2048) {
        // 에러: 데이터 손실
        uint8_t response[1] = {0x4F};
        FUN_08003C00(0x5FE, response, 1);
        
        // 같은 페이지 재요청
        uint8_t retry[2] = {0x43, *(uint32_t *)0x20000310};
        FUN_08003C00(0x5FE, retry, 2);
        return;
    }
    
    // 정상 처리...
}
```

### 흐름 제어

PC가 너무 빨리 보내면 손실 발생:

```python
# 송신 간격 조절
for chunk in page_chunks:
    can_if.send(0x5FF, chunk)
    time.sleep(0.0005)  # 최소 0.5ms 간격
```

## 삽질: 프레임 순서

CAN은 순서 보장 안 됨 (우선순위 기반):

```
# 같은 ID면 순서 보장 (FIFO)
# 다른 ID 끼어들면 순서 밀림

해결책: 
- 페이지 전송 중 다른 메시지 금지
- 또는 시퀀스 번호 추가 (미구현)
```

## 삽질: 버퍼 초기화

버퍼를 0x00으로 초기화하면 Flash 문제:

```c
// 잘못된 초기화
memset(buffer, 0x00, 2048);

// Flash 쓰기 후:
// 원본: 0xFF (빈 영역)
// 실제: 0x00 (잘못된 데이터)

// 올바른 초기화
memset(buffer, 0xFF, 2048);  // Flash 기본값
```

## 복원된 프로토콜 명세

### 페이지 전송 시퀀스

```
1. BMS → PC: 0x43 [page_num]          // 페이지 요청
2. PC → BMS: 0x33 [page_num]          // 페이지 시작
3. PC → BMS: [data 0~7]               // 데이터 프레임 0
4. PC → BMS: [data 8~15]              // 데이터 프레임 1
   ...
5. PC → BMS: [data 2040~2047]         // 데이터 프레임 255
6. PC → BMS: 0x34                     // 페이지 끝
7. (다음 페이지 또는 검증)
```

### 타이밍 요구사항

| 구간 | 최대 시간 |
|------|----------|
| 페이지 요청 → 시작 | 1초 |
| 프레임 간 간격 | 10ms |
| 시작 → 끝 | 5초 |
| 전체 전송 | 10초/페이지 |

## 정리

| 항목 | 값 |
|------|-----|
| 페이지 크기 | 2048 bytes |
| CAN 프레임 크기 | 8 bytes |
| 프레임 수/페이지 | 256개 |
| 전송 속도 | ~200ms/페이지 |
| 데이터 구분 | 첫 바이트 < 0x30 |
| 버퍼 초기값 | 0xFF |

**다음 글에서**: CRC 검증 함수 분석 - 전송 완료 후 검증.

---

## 시리즈 목차

**Part 4: CAN IAP 프로토콜 역분석편**
- [#13 - CAN 수신 핸들러 분석](/posts/ghidra/ghidra-stm32-re-13/)
- [#14 - 명령 코드 체계 파악](/posts/ghidra/ghidra-stm32-re-14/)
- [#15 - 상태 머신 복원](/posts/ghidra/ghidra-stm32-re-15/)
- [#16 - Connection Key 알고리즘](/posts/ghidra/ghidra-stm32-re-16/)
- **#17 - 페이지 전송 프로토콜** ← 현재 글
- #18 - CRC 검증 함수 분석

---

## 참고 자료

- [CAN Bus Flow Control](https://www.csselectronics.com/pages/can-bus-flow-control)
- [STM32 Flash Page Size](https://www.st.com/resource/en/reference_manual/rm0008.pdf)
