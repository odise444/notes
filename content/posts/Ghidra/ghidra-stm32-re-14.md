---
title: "Ghidra로 STM32 부트로더 역분석한 썰 #14 - 명령 코드 체계 파악"
date: 2024-12-13
draft: false
tags: ["Ghidra", "STM32", "리버스엔지니어링", "역분석", "CAN", "프로토콜"]
categories: ["역분석"]
summary: "0x30, 0x31, 0x32... 각 명령 코드의 의미를 밝혀낸다. 요청/응답 쌍 분석."
---

## 지난 글 요약

[지난 글](/posts/ghidra/ghidra-stm32-re-13/)에서 CAN 수신 핸들러를 분석했다. PC→BMS는 0x5FF, BMS→PC는 0x5FE. 이제 **명령 코드**의 의미를 파악하자.

## 명령 디스패처 재분석

`FUN_08003300`의 switch-case:

```c
void FUN_08003300(void) {
    uint8_t cmd = *(uint8_t *)0x20000200;
    
    switch (cmd) {
    case 0x30: FUN_08003500(); break;  // ?
    case 0x31: FUN_08003600(data); break;  // ?
    case 0x32: FUN_08003700(data); break;  // ?
    case 0x33: FUN_08003800(data); break;  // ?
    case 0x34: FUN_08003900(); break;  // ?
    case 0x35: FUN_08003A00(); break;  // ?
    case 0x36: FUN_08003B00(data); break;  // ?
    default: break;
    }
}
```

각 함수를 분석해서 의미를 밝히자.

## 0x30: Connection 요청

```c
void FUN_08003500(void) {
    uint8_t response[8];
    
    // 상태 체크
    if (*(uint8_t *)0x20000300 != 0) {  // g_iap_state
        // 이미 연결됨 - 무시
        return;
    }
    
    // 응답 생성
    response[0] = 0x40;  // 응답 코드
    response[1] = *(uint8_t *)0x20000100;  // FwChk[0]
    response[2] = *(uint8_t *)0x20000101;  // FwChk[1]
    response[3] = *(uint8_t *)0x20000102;  // FwChk[2]
    response[4] = *(uint8_t *)0x20000103;  // FwChk[3]
    response[5] = *(uint8_t *)0x20000104;  // FwDate[0]
    response[6] = *(uint8_t *)0x20000105;  // FwDate[1]
    response[7] = *(uint8_t *)0x20000106;  // FwDate[2]
    
    FUN_08003C00(0x5FE, response, 8);
    
    // 상태 변경
    *(uint8_t *)0x20000300 = 1;  // STATE_WAIT_CALC
    
    // 타임아웃 리셋
    *(uint32_t *)0x20000304 = 0;
}
```

**해석**:
- PC가 0x30 보내면 부트로더가 FwChk + FwDate를 0x40으로 응답
- FwChk: 펌웨어 체크섬 (4바이트)
- FwDate: 펌웨어 날짜 (3바이트)

## 0x31: Calc 값 수신

```c
void FUN_08003600(uint8_t *data) {
    uint32_t received_calc;
    uint32_t expected_calc;
    
    // 상태 체크
    if (*(uint8_t *)0x20000300 != 1) {  // STATE_WAIT_CALC
        return;
    }
    
    // 수신된 Calc 값
    received_calc = data[1] | (data[2] << 8) | 
                    (data[3] << 16) | (data[4] << 24);
    
    // 예상 Calc 계산
    expected_calc = FUN_08003D00();  // CalcKey 함수
    
    if (received_calc == expected_calc) {
        // 인증 성공
        uint8_t response[1] = {0x41};
        FUN_08003C00(0x5FE, response, 1);
        
        *(uint8_t *)0x20000300 = 2;  // STATE_AUTHENTICATED
    } else {
        // 인증 실패
        uint8_t response[1] = {0x4F};  // Error
        FUN_08003C00(0x5FE, response, 1);
        
        *(uint8_t *)0x20000300 = 0;  // STATE_IDLE
    }
}
```

**해석**:
- PC가 계산한 Key를 0x31로 전송
- 부트로더가 검증 후 0x41(성공) 또는 0x4F(실패) 응답

## 0x32: 크기 응답

```c
void FUN_08003700(uint8_t *data) {
    if (*(uint8_t *)0x20000300 != 2) {  // STATE_AUTHENTICATED
        return;
    }
    
    // 펌웨어 크기 수신
    uint32_t fw_size = data[1] | (data[2] << 8) | 
                       (data[3] << 16) | (data[4] << 24);
    
    // 크기 검증
    if (fw_size > 0x3E800) {  // 256KB 제한
        uint8_t response[1] = {0x4F};  // Error: too large
        FUN_08003C00(0x5FE, response, 1);
        return;
    }
    
    // 크기 저장
    *(uint32_t *)0x20000308 = fw_size;  // g_fw_size
    
    // 페이지 수 계산
    uint32_t pages = (fw_size + 2047) / 2048;  // 올림
    *(uint32_t *)0x2000030C = pages;  // g_total_pages
    *(uint32_t *)0x20000310 = 0;      // g_current_page
    
    // 첫 페이지 요청
    uint8_t response[2];
    response[0] = 0x43;  // 페이지 요청
    response[1] = 0;     // 페이지 번호
    FUN_08003C00(0x5FE, response, 2);
    
    *(uint8_t *)0x20000300 = 3;  // STATE_RECEIVING
}
```

**해석**:
- 인증 후 부트로더가 0x42로 크기 요청
- PC가 0x32로 크기 응답
- 부트로더가 0x43으로 첫 페이지 요청

## 0x33: 페이지 시작

```c
void FUN_08003800(uint8_t *data) {
    if (*(uint8_t *)0x20000300 != 3) {  // STATE_RECEIVING
        return;
    }
    
    uint8_t page_num = data[1];
    
    // 페이지 번호 확인
    if (page_num != *(uint32_t *)0x20000310) {  // g_current_page
        // 순서 오류
        return;
    }
    
    // 페이지 버퍼 초기화
    *(uint32_t *)0x20000314 = 0;  // g_page_offset
    
    // 상태 변경
    *(uint8_t *)0x20000300 = 4;  // STATE_PAGE_DATA
}
```

**해석**:
- PC가 0x33 + 페이지 번호로 페이지 전송 시작 알림
- 이후 데이터 프레임들이 연속으로 옴

## 0x34: 페이지 끝

```c
void FUN_08003900(void) {
    if (*(uint8_t *)0x20000300 != 4) {  // STATE_PAGE_DATA
        return;
    }
    
    // 페이지 버퍼 → Flash 쓰기
    uint32_t page_addr = 0x08004000 + 
                         *(uint32_t *)0x20000310 * 2048;  // g_current_page
    
    FUN_08002E00(page_addr, (uint8_t *)0x20000400);  // Flash_WritePage
    
    // 다음 페이지
    *(uint32_t *)0x20000310 += 1;
    
    // 완료 체크
    if (*(uint32_t *)0x20000310 >= *(uint32_t *)0x2000030C) {
        // 모든 페이지 수신 완료
        uint8_t response[1] = {0x45};  // 검증 시작
        FUN_08003C00(0x5FE, response, 1);
        
        *(uint8_t *)0x20000300 = 5;  // STATE_VERIFYING
    } else {
        // 다음 페이지 요청
        uint8_t response[2];
        response[0] = 0x43;
        response[1] = *(uint32_t *)0x20000310;
        FUN_08003C00(0x5FE, response, 2);
        
        *(uint8_t *)0x20000300 = 3;  // STATE_RECEIVING
    }
}
```

**해석**:
- PC가 0x34로 페이지 끝 알림
- 부트로더가 Flash에 쓰고 다음 페이지 요청 또는 검증 시작

## 0x35: 전송 완료

```c
void FUN_08003A00(void) {
    // 최종 확인 (사용 안 하는 듯?)
}
```

## 0x36: 검증 결과

```c
void FUN_08003B00(uint8_t *data) {
    if (*(uint8_t *)0x20000300 != 5) {  // STATE_VERIFYING
        return;
    }
    
    uint8_t result = data[1];
    
    if (result == 0x01) {  // 검증 성공
        // 버퍼 → 앱 영역 복사
        FUN_08002F00();  // CopyBufferToApp
        
        uint8_t response[1] = {0x46};  // 완료
        FUN_08003C00(0x5FE, response, 1);
        
        *(uint8_t *)0x20000300 = 6;  // STATE_COMPLETE
    } else {
        // 검증 실패
        uint8_t response[1] = {0x4F};
        FUN_08003C00(0x5FE, response, 1);
        
        *(uint8_t *)0x20000300 = 0;  // STATE_IDLE
    }
}
```

## 명령 코드 체계 정리

### PC → BMS (0x5FF)

| 코드 | 이름 | 데이터 | 설명 |
|------|------|--------|------|
| 0x30 | CONNECT | 없음 | 연결 요청 |
| 0x31 | CALC | [calc:4] | 인증 키 전송 |
| 0x32 | SIZE | [size:4] | 펌웨어 크기 |
| 0x33 | PAGE_START | [page:1] | 페이지 시작 |
| 0x34 | PAGE_END | 없음 | 페이지 끝 |
| 0x35 | TX_DONE | 없음 | 전송 완료 |
| 0x36 | VERIFY | [result:1] | 검증 결과 |

### BMS → PC (0x5FE)

| 코드 | 이름 | 데이터 | 설명 |
|------|------|--------|------|
| 0x40 | FWINFO | [chk:4][date:3] | FW 정보 |
| 0x41 | AUTH_OK | 없음 | 인증 성공 |
| 0x42 | SIZE_REQ | 없음 | 크기 요청 |
| 0x43 | PAGE_REQ | [page:1] | 페이지 요청 |
| 0x45 | VERIFY_START | 없음 | 검증 시작 |
| 0x46 | COMPLETE | 없음 | 완료 |
| 0x4F | ERROR | 없음 | 에러 |

## 프로토콜 시퀀스

```
PC                          BMS
│                            │
│──────── 0x30 ─────────────►│  Connect
│                            │
│◄─────── 0x40 ──────────────│  FwChk + FwDate
│                            │
│──────── 0x31 [calc] ──────►│  Authentication
│                            │
│◄─────── 0x41 ──────────────│  Auth OK
│                            │
│◄─────── 0x42 ──────────────│  Size Request
│                            │
│──────── 0x32 [size] ──────►│  Size Response
│                            │
│◄─────── 0x43 [0] ──────────│  Request Page 0
│                            │
│──────── 0x33 [0] ─────────►│  Page 0 Start
│──────── [data] ───────────►│  (8 bytes × N)
│──────── 0x34 ─────────────►│  Page 0 End
│                            │
│◄─────── 0x43 [1] ──────────│  Request Page 1
│          ...               │
│                            │
│◄─────── 0x45 ──────────────│  Verify Start
│                            │
│──────── 0x36 [1] ─────────►│  Verify OK
│                            │
│◄─────── 0x46 ──────────────│  Complete
│                            │
```

## 복원된 헤더 파일

```c
// iap_protocol.h

// CAN IDs
#define CAN_ID_PC_TO_BMS    0x5FF
#define CAN_ID_BMS_TO_PC    0x5FE

// PC → BMS Commands
#define CMD_CONNECT         0x30
#define CMD_CALC            0x31
#define CMD_SIZE            0x32
#define CMD_PAGE_START      0x33
#define CMD_PAGE_END        0x34
#define CMD_TX_DONE         0x35
#define CMD_VERIFY          0x36

// BMS → PC Responses
#define RSP_FWINFO          0x40
#define RSP_AUTH_OK         0x41
#define RSP_SIZE_REQ        0x42
#define RSP_PAGE_REQ        0x43
#define RSP_VERIFY_START    0x45
#define RSP_COMPLETE        0x46
#define RSP_ERROR           0x4F

// States
typedef enum {
    STATE_IDLE = 0,
    STATE_WAIT_CALC,
    STATE_AUTHENTICATED,
    STATE_RECEIVING,
    STATE_PAGE_DATA,
    STATE_VERIFYING,
    STATE_COMPLETE
} iap_state_t;
```

## 삽질: 데이터 프레임 구분

페이지 데이터는 명령 코드 없이 연속으로 옴:

```c
// 데이터 프레임 처리
void FUN_08003E00(uint8_t *data, uint8_t len) {
    if (*(uint8_t *)0x20000300 != 4) {  // STATE_PAGE_DATA
        return;
    }
    
    // 첫 바이트가 명령 코드가 아닌 경우 → 데이터
    if (data[0] < 0x30 || data[0] > 0x3F) {
        // 버퍼에 추가
        uint32_t offset = *(uint32_t *)0x20000314;
        memcpy((uint8_t *)(0x20000400 + offset), data, len);
        *(uint32_t *)0x20000314 = offset + len;
    }
}
```

## 정리

| 단계 | PC 명령 | BMS 응답 |
|------|---------|----------|
| 1. 연결 | 0x30 | 0x40 (FW 정보) |
| 2. 인증 | 0x31 (calc) | 0x41 (OK) |
| 3. 크기 | 0x32 (size) | 0x43 (페이지 요청) |
| 4. 데이터 | 0x33→data→0x34 | 0x43 (다음) |
| 5. 검증 | 0x36 (결과) | 0x46 (완료) |

**다음 글에서**: 상태 머신 복원 - 전체 흐름을 코드로.

---

## 시리즈 목차

**Part 4: CAN IAP 프로토콜 역분석편**
- [#13 - CAN 수신 핸들러 분석](/posts/ghidra/ghidra-stm32-re-13/)
- **#14 - 명령 코드 체계 파악** ← 현재 글
- #15 - 상태 머신 복원
- #16 - Connection Key 알고리즘
- #17 - 페이지 전송 프로토콜
- #18 - CRC 검증 함수 분석

---

## 참고 자료

- [CAN Protocol Basics](https://www.csselectronics.com/pages/can-bus-simple-intro-tutorial)
- [IAP Bootloader Design](https://www.st.com/resource/en/application_note/an3965.pdf)
