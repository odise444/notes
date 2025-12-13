---
title: "Ghidra로 STM32 부트로더 역분석한 썰 #16 - Connection Key 알고리즘"
date: 2024-12-13
draft: false
tags: ["Ghidra", "STM32", "리버스엔지니어링", "역분석", "인증", "보안"]
categories: ["역분석"]
summary: "부트로더 인증의 핵심! FwChk와 FwDate로 Key를 계산하는 알고리즘을 밝혀낸다."
---

## 지난 글 요약

[지난 글](/posts/ghidra/ghidra-stm32-re-15/)에서 상태 머신을 복원했다. 7개 상태, 타임아웃 10초. 이제 **인증 알고리즘**을 역분석하자.

## 인증 흐름 복습

```
PC                          BMS
│                            │
│──────── 0x30 ─────────────►│  Connect
│                            │
│◄─────── 0x40 ──────────────│  FwChk[4] + FwDate[3]
│                            │
│     [PC가 Key 계산]         │
│                            │
│──────── 0x31 [calc] ──────►│  Key 전송
│                            │
│◄─────── 0x41 or 0x4F ──────│  OK or Error
```

**핵심 질문**: PC는 FwChk와 FwDate로 어떻게 Key를 계산하는가?

## CalcKey 함수 찾기

인증 처리 함수 `FUN_08003600`에서:

```c
void FUN_08003600(uint8_t *data) {
    // ...
    
    // 예상 Calc 계산
    expected_calc = FUN_08003D00();  // ← 이 함수!
    
    if (received_calc == expected_calc) {
        // 성공
    }
}
```

`FUN_08003D00`이 Key 계산 함수!

## FUN_08003D00 디컴파일

```c
uint32_t FUN_08003D00(void) {
    uint32_t fw_chk;
    uint32_t fw_date;
    uint32_t result;
    
    // FwChk 로드 (Little Endian)
    fw_chk = *(uint32_t *)0x20000100;
    
    // FwDate 로드 (3바이트 → 4바이트)
    fw_date = *(uint8_t *)0x20000104 |
              (*(uint8_t *)0x20000105 << 8) |
              (*(uint8_t *)0x20000106 << 16);
    
    // XOR 연산
    result = fw_chk ^ fw_date;
    
    // 추가 연산
    result = result ^ (result >> 16);
    result = result ^ (result >> 8);
    
    return result;
}
```

**알고리즘 발견!**

## 알고리즘 분석

### 단계별 분해

```
Step 1: FwChk와 FwDate XOR
─────────────────────────────
FwChk  = 0x12345678
FwDate = 0x00241215  (2024년 12월 15일?)

result = FwChk ^ FwDate
       = 0x12345678 ^ 0x00241215
       = 0x1210446D

Step 2: 상위 16비트 XOR
─────────────────────────────
result = result ^ (result >> 16)
       = 0x1210446D ^ 0x00001210
       = 0x1210567D

Step 3: 상위 8비트 XOR
─────────────────────────────
result = result ^ (result >> 8)
       = 0x1210567D ^ 0x00121056
       = 0x1202462B
```

### 목적 분석

```
원본: 0x12345678 (32비트)
결과: 0x1202462B (모든 비트가 섞임)

- 단순 XOR보다 복잡도 증가
- 모든 비트가 결과에 영향
- 간단하지만 예측 어려움
```

## FwChk, FwDate 출처

Flash에 저장된 펌웨어 정보:

```c
// 0x08003FF0 (부트로더 끝부분)에 저장
const uint8_t FW_INFO[] __attribute__((section(".fw_info"))) = {
    0x78, 0x56, 0x34, 0x12,  // FwChk (Little Endian)
    0x15, 0x12, 0x24,        // FwDate: 2024-12-15 (YY-MM-DD)
    0x00                      // Padding
};
```

부팅 시 RAM에 복사:

```c
void FUN_08002500(void) {  // 초기화
    // FW 정보 복사
    memcpy((void *)0x20000100, (void *)0x08003FF0, 8);
}
```

## FwDate 형식

```
FwDate[0] = 0x15 = 21  → 일 (Day)
FwDate[1] = 0x12 = 18  → 월 (Month)? 이상함...
FwDate[2] = 0x24 = 36  → 연도 하위 2자리?

또는:

FwDate[0] = 0x15 = 21  → 연도 (20xx)
FwDate[1] = 0x12 = 18  → 월 * 1.5?
FwDate[2] = 0x24 = 36  → 일 * 1.5?

아직 불확실...
```

**실제 펌웨어 빌드 스크립트를 찾아야 정확한 형식을 알 수 있음**

## 복원된 Key 계산 함수

```c
// iap_auth.c

// 펌웨어 정보 (Flash에서 로드)
static uint32_t g_fw_check;
static uint8_t g_fw_date[3];

void IAP_LoadFwInfo(void) {
    // Flash 특정 위치에서 로드
    g_fw_check = *(uint32_t *)FW_INFO_ADDR;
    g_fw_date[0] = *(uint8_t *)(FW_INFO_ADDR + 4);
    g_fw_date[1] = *(uint8_t *)(FW_INFO_ADDR + 5);
    g_fw_date[2] = *(uint8_t *)(FW_INFO_ADDR + 6);
}

uint32_t IAP_CalcKey(void) {
    uint32_t fw_chk = g_fw_check;
    uint32_t fw_date = g_fw_date[0] | 
                       (g_fw_date[1] << 8) | 
                       (g_fw_date[2] << 16);
    
    uint32_t result = fw_chk ^ fw_date;
    result ^= (result >> 16);
    result ^= (result >> 8);
    
    return result;
}
```

## PC측 Key 계산

Python으로 구현:

```python
# iap_client.py

def calc_key(fw_chk: bytes, fw_date: bytes) -> int:
    """부트로더 인증 키 계산"""
    
    # Little Endian으로 해석
    chk = int.from_bytes(fw_chk, 'little')
    date = int.from_bytes(fw_date + b'\x00', 'little')  # 3바이트 + 패딩
    
    # XOR 연산
    result = chk ^ date
    result ^= (result >> 16) & 0xFFFF
    result ^= (result >> 8) & 0xFF
    
    return result & 0xFFFFFFFF


def authenticate(can_interface, timeout=5.0):
    """부트로더 인증 수행"""
    
    # 1. Connection 요청
    can_interface.send(0x5FF, bytes([0x30]))
    
    # 2. FW 정보 수신 대기
    msg = can_interface.receive(timeout)
    if msg is None or msg.data[0] != 0x40:
        raise Exception("Connection failed")
    
    fw_chk = msg.data[1:5]
    fw_date = msg.data[5:8]
    
    print(f"FwChk: {fw_chk.hex()}")
    print(f"FwDate: {fw_date.hex()}")
    
    # 3. Key 계산
    key = calc_key(fw_chk, fw_date)
    print(f"Calculated Key: 0x{key:08X}")
    
    # 4. Key 전송
    key_bytes = key.to_bytes(4, 'little')
    can_interface.send(0x5FF, bytes([0x31]) + key_bytes)
    
    # 5. 응답 확인
    msg = can_interface.receive(timeout)
    if msg is None:
        raise Exception("Auth timeout")
    
    if msg.data[0] == 0x41:
        print("Authentication successful!")
        return True
    elif msg.data[0] == 0x4F:
        raise Exception("Authentication failed")
    
    return False
```

## 보안 분석

### 취약점

1. **알고리즘 노출**: 바이너리에서 쉽게 추출 가능
2. **고정 키**: FwChk/FwDate가 변하지 않으면 Key도 고정
3. **Challenge-Response 없음**: 재전송 공격 가능

### 실제 보안 수준

```
보안 목적: 무단 펌웨어 업로드 방지
실제 효과: 
  - 일반 사용자: ✅ 충분
  - 리버서: ❌ 쉽게 우회
```

### 개선 제안

```c
// 개선된 인증 (Challenge-Response)
uint32_t IAP_CalcKey_Improved(uint32_t challenge) {
    uint32_t fw_chk = g_fw_check;
    uint32_t fw_date = ...;
    
    // Challenge 포함
    uint32_t result = fw_chk ^ fw_date ^ challenge;
    
    // 더 복잡한 mixing
    result = (result << 13) | (result >> 19);
    result ^= 0xDEADBEEF;
    result = (result << 7) | (result >> 25);
    
    return result;
}
```

## 삽질: Endianness

FwChk를 Big Endian으로 읽어서 삽질:

```c
// 잘못된 해석
fw_chk = 0x78563412  // 바이트 순서 뒤집음

// 올바른 해석 (Little Endian)
fw_chk = 0x12345678
```

## 삽질: 3바이트 확장

FwDate가 3바이트인데 4바이트로 확장 시:

```c
// 잘못된 코드
fw_date = *(uint32_t *)0x20000104;  // 4바이트 읽음 → 쓰레기 포함

// 올바른 코드
fw_date = *(uint8_t *)0x20000104 |
          (*(uint8_t *)0x20000105 << 8) |
          (*(uint8_t *)0x20000106 << 16);
// 상위 바이트는 0
```

## Ghidra 라벨 정리

```
주소/함수 → 이름
─────────────────────────────
0x08003FF0 → FW_INFO_SECTION
0x20000100 → g_fw_check
0x20000104 → g_fw_date
FUN_08003D00 → IAP_CalcKey
```

## 정리

| 항목 | 값/설명 |
|------|---------|
| 입력 1 | FwChk (4바이트, Flash 저장) |
| 입력 2 | FwDate (3바이트, Flash 저장) |
| 알고리즘 | XOR + bit mixing |
| 출력 | 32비트 Key |
| 보안 수준 | 낮음 (역분석 가능) |

**알고리즘**:
```
Key = (FwChk ^ FwDate) ^ ((FwChk ^ FwDate) >> 16) 
                       ^ ((result) >> 8)
```

**다음 글에서**: 페이지 전송 프로토콜 - 2KB 데이터를 CAN으로.

---

## 시리즈 목차

**Part 4: CAN IAP 프로토콜 역분석편**
- [#13 - CAN 수신 핸들러 분석](/posts/ghidra/ghidra-stm32-re-13/)
- [#14 - 명령 코드 체계 파악](/posts/ghidra/ghidra-stm32-re-14/)
- [#15 - 상태 머신 복원](/posts/ghidra/ghidra-stm32-re-15/)
- **#16 - Connection Key 알고리즘** ← 현재 글
- #17 - 페이지 전송 프로토콜
- #18 - CRC 검증 함수 분석

---

## 참고 자료

- [Embedded Security Best Practices](https://interrupt.memfault.com/blog/secure-bootloader)
- [Challenge-Response Authentication](https://en.wikipedia.org/wiki/Challenge%E2%80%93response_authentication)
