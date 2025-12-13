---
title: "Ghidra로 STM32 부트로더 역분석한 썰 #18 - CRC 검증 함수 분석"
date: 2024-12-13
draft: false
tags: ["Ghidra", "STM32", "리버스엔지니어링", "역분석", "CRC", "무결성"]
categories: ["역분석"]
summary: "펌웨어 전송 완료! 하지만 제대로 왔을까? CRC 검증으로 무결성을 확인하는 로직."
---

## 지난 글 요약

[지난 글](/posts/ghidra/ghidra-stm32-re-17/)에서 페이지 전송 프로토콜을 분석했다. 256 프레임/페이지. 이제 전송 완료 후 **CRC 검증** 로직을 분석하자.

## 검증 흐름

```
PC                              BMS
│                                │
│◄────── 0x45 ──────────────────│  Verify Start
│                                │
│     [PC가 CRC 계산]            │  [BMS도 CRC 계산]
│                                │
│─────── 0x36 [result] ─────────►│  Verify Result
│                                │
│◄────── 0x46 or 0x4F ──────────│  Complete or Error
```

## CRC 계산 함수 찾기

검증 처리 함수 `FUN_08003B00`에서:

```c
void FUN_08003B00(uint8_t *data) {
    if (*(uint8_t *)0x20000300 != 5) {  // STATE_VERIFYING
        return;
    }
    
    // PC가 보낸 CRC
    uint32_t pc_crc = data[1] | (data[2] << 8) | 
                      (data[3] << 16) | (data[4] << 24);
    
    // BMS가 계산한 CRC
    uint32_t bms_crc = FUN_08003F00();  // ← CRC 계산 함수
    
    if (pc_crc == bms_crc) {
        // 검증 성공
        FUN_08002F00();  // 버퍼 → 앱 복사
        
        uint8_t response[1] = {0x46};  // Complete
        FUN_08003C00(0x5FE, response, 1);
        
        *(uint8_t *)0x20000300 = 6;  // STATE_COMPLETE
    } else {
        // 검증 실패
        uint8_t response[1] = {0x4F};  // Error
        FUN_08003C00(0x5FE, response, 1);
        
        *(uint8_t *)0x20000300 = 0;  // STATE_IDLE
    }
}
```

## FUN_08003F00 분석: STM32 HW CRC

```c
uint32_t FUN_08003F00(void) {
    uint32_t *data = (uint32_t *)0x08004000;  // 버퍼 시작
    uint32_t size = *(uint32_t *)0x20000308;  // g_fw_size
    uint32_t words = (size + 3) / 4;          // 워드 수
    
    // CRC 유닛 리셋
    *(uint32_t *)0x40023008 = 0x01;  // CRC_CR.RESET = 1
    
    // 데이터 입력
    for (uint32_t i = 0; i < words; i++) {
        *(uint32_t *)0x40023000 = data[i];  // CRC_DR
    }
    
    // 결과 읽기
    return *(uint32_t *)0x40023000;  // CRC_DR
}
```

**STM32 하드웨어 CRC 사용!**

## STM32 CRC 유닛

### 레지스터

| 레지스터 | 주소 | 설명 |
|----------|------|------|
| CRC_DR | 0x40023000 | 데이터 레지스터 |
| CRC_IDR | 0x40023004 | 독립 데이터 (8비트) |
| CRC_CR | 0x40023008 | 제어 (리셋만) |

### 특성

- **다항식**: 0x04C11DB7 (CRC-32/MPEG-2)
- **초기값**: 0xFFFFFFFF
- **입력**: 32비트 워드 단위
- **출력**: 32비트 CRC
- **NOT 없음**: 최종 XOR 없음

## STM32 CRC vs 표준 CRC-32

```
STM32 CRC:
- Polynomial: 0x04C11DB7
- Initial: 0xFFFFFFFF
- Input: No reflection
- Output: No reflection
- Final XOR: 0x00000000

표준 CRC-32 (ZIP, PNG):
- Polynomial: 0x04C11DB7
- Initial: 0xFFFFFFFF
- Input: Reflected (bit order reversed)
- Output: Reflected
- Final XOR: 0xFFFFFFFF

결과: 서로 다른 값!
```

## PC측 CRC 계산

Python으로 STM32 호환 CRC 구현:

```python
# crc32_stm32.py

def crc32_stm32(data: bytes) -> int:
    """STM32 하드웨어 CRC와 호환되는 CRC-32 계산"""
    
    # 4바이트 정렬
    if len(data) % 4 != 0:
        data = data + bytes([0xFF] * (4 - len(data) % 4))
    
    crc = 0xFFFFFFFF
    
    for i in range(0, len(data), 4):
        # Little Endian으로 32비트 워드 읽기
        word = int.from_bytes(data[i:i+4], 'little')
        
        crc ^= word
        
        for _ in range(32):
            if crc & 0x80000000:
                crc = ((crc << 1) ^ 0x04C11DB7) & 0xFFFFFFFF
            else:
                crc = (crc << 1) & 0xFFFFFFFF
    
    return crc


def crc32_stm32_fast(data: bytes) -> int:
    """테이블 기반 고속 CRC 계산"""
    
    # CRC 테이블 생성
    table = []
    for i in range(256):
        crc = i << 24
        for _ in range(8):
            if crc & 0x80000000:
                crc = ((crc << 1) ^ 0x04C11DB7) & 0xFFFFFFFF
            else:
                crc = (crc << 1) & 0xFFFFFFFF
        table.append(crc)
    
    # 4바이트 정렬
    if len(data) % 4 != 0:
        data = data + bytes([0xFF] * (4 - len(data) % 4))
    
    crc = 0xFFFFFFFF
    
    for i in range(0, len(data), 4):
        word = int.from_bytes(data[i:i+4], 'little')
        crc ^= word
        
        for _ in range(4):
            crc = ((crc << 8) ^ table[(crc >> 24) & 0xFF]) & 0xFFFFFFFF
    
    return crc


# 테스트
if __name__ == "__main__":
    test_data = bytes([0x01, 0x02, 0x03, 0x04, 0x05, 0x06, 0x07, 0x08])
    
    crc1 = crc32_stm32(test_data)
    crc2 = crc32_stm32_fast(test_data)
    
    print(f"Basic: 0x{crc1:08X}")
    print(f"Fast:  0x{crc2:08X}")
    
    # STM32에서 계산한 값과 비교 필요
```

## 검증 프로토콜 복원

```c
// BMS측 (부트로더)
void IAP_StartVerify(void) {
    // CRC 계산
    uint32_t crc = CRC_Calculate((uint32_t *)BUFFER_START, g_iap.fw_size);
    
    // 계산 완료 알림
    uint8_t response[1] = {RSP_VERIFY_START};
    CAN_Transmit(CAN_ID_BMS_TO_PC, response, 1);
    
    // CRC 저장 (PC와 비교용)
    g_iap.calculated_crc = crc;
}

void IAP_HandleVerify(uint8_t *data, uint8_t len) {
    // PC가 계산한 CRC
    uint32_t pc_crc = data[1] | (data[2] << 8) | 
                      (data[3] << 16) | (data[4] << 24);
    
    // 비교
    if (pc_crc == g_iap.calculated_crc) {
        // 성공!
        IAP_CopyBufferToApp();
        
        uint8_t response[1] = {RSP_COMPLETE};
        CAN_Transmit(CAN_ID_BMS_TO_PC, response, 1);
    } else {
        // 실패
        uint8_t response[1] = {RSP_ERROR};
        CAN_Transmit(CAN_ID_BMS_TO_PC, response, 1);
    }
}
```

## PC측 검증 코드

```python
# verify.py

def verify_firmware(can_if, firmware: bytes):
    """펌웨어 CRC 검증"""
    
    # 검증 시작 대기
    msg = can_if.receive(timeout=10.0)
    if msg is None or msg.data[0] != 0x45:
        raise Exception("Verify start not received")
    
    print("Calculating CRC...")
    
    # CRC 계산
    crc = crc32_stm32(firmware)
    print(f"CRC: 0x{crc:08X}")
    
    # CRC 전송
    crc_bytes = crc.to_bytes(4, 'little')
    can_if.send(0x5FF, bytes([0x36, 0x01]) + crc_bytes)
    
    # 결과 대기
    msg = can_if.receive(timeout=5.0)
    if msg is None:
        raise Exception("Verify response timeout")
    
    if msg.data[0] == 0x46:
        print("Verification PASSED!")
        return True
    elif msg.data[0] == 0x4F:
        print("Verification FAILED!")
        return False
    
    return False
```

## CRC 클럭 활성화

CRC 유닛도 클럭 필요:

```c
// SystemInit 또는 초기화에서
void CRC_Init(void) {
    // AHB 클럭 활성화
    RCC->AHBENR |= RCC_AHBENR_CRCEN;
}
```

## 버퍼 → 앱 복사

검증 성공 후:

```c
void FUN_08002F00(void) {  // IAP_CopyBufferToApp
    uint32_t src = 0x08004000;   // 버퍼 시작
    uint32_t dst = 0x08042800;   // 앱 시작
    uint32_t size = *(uint32_t *)0x20000308;  // g_fw_size
    
    Flash_Unlock();
    
    // 앱 영역 Erase
    uint32_t pages = (size + 2047) / 2048;
    for (uint32_t i = 0; i < pages; i++) {
        Flash_ErasePage(dst + i * 2048);
    }
    
    // 복사 (Half-word 단위)
    for (uint32_t i = 0; i < size; i += 2) {
        uint16_t data = *(uint16_t *)(src + i);
        Flash_ProgramHalfWord(dst + i, data);
    }
    
    Flash_Lock();
}
```

## 이중 버퍼 구조 이유

```
Flash Layout:
┌────────────────┐ 0x08000000
│   Bootloader   │ 16KB
├────────────────┤ 0x08004000
│                │
│     Buffer     │ 254KB (수신 영역)
│                │
├────────────────┤ 0x08042800
│                │
│   Application  │ 246KB (실행 영역)
│                │
└────────────────┘ 0x08080000
```

**장점**:
1. 수신 중 오류 → 기존 앱 유지
2. 검증 실패 → 기존 앱으로 동작
3. 전원 차단 → 앱 또는 재시도 가능

**단점**:
- Flash 절반만 앱에 사용

## 삽질: Endianness

STM32 CRC는 32비트 단위:

```c
// 잘못된 코드: 바이트 순서 문제
for (int i = 0; i < size; i++) {
    CRC->DR = data[i];  // 8비트씩 입력 → 다른 결과!
}

// 올바른 코드: 32비트 단위
for (int i = 0; i < size/4; i++) {
    CRC->DR = ((uint32_t *)data)[i];
}
```

## 삽질: 패딩

크기가 4의 배수가 아니면:

```c
// 잘못된 코드
words = size / 4;  // 나머지 버림

// 올바른 코드
words = (size + 3) / 4;  // 올림

// 남는 바이트는 0xFF로 취급 (Flash 기본값)
```

## Ghidra 라벨 정리

```
주소/함수 → 이름
─────────────────────────────
0x40023000 → CRC_DR
0x40023008 → CRC_CR
FUN_08003F00 → CRC_Calculate
FUN_08002F00 → IAP_CopyBufferToApp
FUN_08003B00 → IAP_HandleVerify
```

## Part 4 완료!

프로토콜 역분석 완료:

| 항목 | 내용 |
|------|------|
| CAN ID | PC→BMS: 0x5FF, BMS→PC: 0x5FE |
| 명령 체계 | 0x30 시리즈 (요청), 0x40 시리즈 (응답) |
| 인증 | FwChk ⊕ FwDate + bit mixing |
| 페이지 | 2KB, 256 프레임 |
| CRC | STM32 HW CRC-32 (0x04C11DB7) |

---

## 정리

| 항목 | 값 |
|------|-----|
| CRC 다항식 | 0x04C11DB7 |
| 초기값 | 0xFFFFFFFF |
| 입력 단위 | 32비트 워드 |
| 최종 XOR | 없음 |
| 하드웨어 | STM32 CRC 유닛 |

**다음 Part**: 핵심 로직 역분석 - 부트 진입 조건, 앱 유효성 검사, 점프 코드.

---

## 시리즈 목차

**Part 4: CAN IAP 프로토콜 역분석편** ✅
- [#13 - CAN 수신 핸들러 분석](/posts/ghidra/ghidra-stm32-re-13/)
- [#14 - 명령 코드 체계 파악](/posts/ghidra/ghidra-stm32-re-14/)
- [#15 - 상태 머신 복원](/posts/ghidra/ghidra-stm32-re-15/)
- [#16 - Connection Key 알고리즘](/posts/ghidra/ghidra-stm32-re-16/)
- [#17 - 페이지 전송 프로토콜](/posts/ghidra/ghidra-stm32-re-17/)
- **#18 - CRC 검증 함수 분석** ← 현재 글

**Part 5: 핵심 로직 역분석편**
- #19 - 부트 진입 조건 분석
- #20 - 앱 유효성 검사
- #21 - 앱 점프 코드 분석
- #22 - 버퍼 → 앱 복사 로직

---

## 참고 자료

- [STM32 CRC Calculation Unit](https://www.st.com/resource/en/reference_manual/rm0008.pdf)
- [CRC-32 Variants](https://reveng.sourceforge.io/crc-catalogue/17plus.htm)
- [Online CRC Calculator](https://crccalc.com/)
