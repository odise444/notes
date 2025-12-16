---
title: "Ghidra STM32 역분석 #17 - 데이터 프레임 처리"
date: 2024-12-13
draft: false
tags: ["Ghidra", "STM32", "리버스엔지니어링", "역분석", "CAN", "Flash"]
categories: ["역분석"]
series: ["Ghidra 역분석"]
summary: "2KB 페이지를 8바이트 CAN 프레임으로 전송하는 방법."
---

Flash 페이지는 2KB인데 CAN 프레임은 8바이트. 256개 프레임이 필요하다.

---

## 데이터 프레임 구조

```
Byte 0: 시퀀스 번호 (0~255)
Byte 1-7: 데이터 (7바이트)
```

시퀀스 번호로 프레임 순서를 구분한다. 2KB = 7바이트 × 293프레임 ≈ 294프레임 필요.

---

## 처리 함수

```c
void Handle_DataFrame(uint8_t *data) {
    uint8_t seq = data[0];
    
    // 버퍼에 저장
    memcpy(&g_page_buffer[seq * 7], &data[1], 7);
    
    g_received_frames++;
    
    // 페이지 완료?
    if (g_received_frames >= 293) {
        // Flash 쓰기
        Flash_WritePage(g_current_page, g_page_buffer);
        g_current_page++;
        g_received_frames = 0;
        
        // 응답
        CAN_Send(0x5FE, 0x43, g_current_page);
    }
}
```

---

## Flash 쓰기

```c
void Flash_WritePage(uint32_t page, uint8_t *data) {
    uint32_t addr = BUFFER_START + page * 2048;
    
    Flash_Unlock();
    Flash_ErasePage(addr);
    
    for (int i = 0; i < 2048; i += 2) {
        uint16_t halfword = data[i] | (data[i+1] << 8);
        Flash_WriteHalfWord(addr + i, halfword);
    }
    
    Flash_Lock();
}
```

버퍼 영역(`0x08004000`)에 먼저 쓴다. 검증 후에 앱 영역으로 복사.

---

## CRC 검증

모든 페이지 전송 후:

```c
void Handle_Verify(void) {
    uint32_t crc = CalcCRC32(BUFFER_START, g_total_size);
    
    if (crc == g_expected_crc) {
        // 버퍼 → 앱 영역 복사
        CopyBufferToApp();
        CAN_Send(0x5FE, 0x45, 0);  // OK
    } else {
        CAN_Send(0x5FE, 0x4F, 0);  // FAIL
    }
}
```

---

다음 글에서 Python 업로더 제작.

[#18 - Python 업로더](/posts/ghidra/ghidra-stm32-re-18/)
