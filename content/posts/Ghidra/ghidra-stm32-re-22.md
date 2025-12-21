---
title: "Ghidra STM32 역분석 #22 - 버퍼 복사 로직"
date: 2024-12-14
draft: false
tags: ["Ghidra", "STM32", "리버스엔지니어링", "역분석", "Flash"]
categories: ["역분석"]
series: ["Ghidra 역분석"]
summary: "버퍼에 받은 펌웨어를 앱 영역으로 복사하는 과정."
---

CAN으로 받은 펌웨어는 버퍼 영역(0x08004000)에 저장된다. CRC 검증 후 앱 영역(0x08042800)으로 복사.

---

## 왜 버퍼를 쓰나?

직접 앱 영역에 쓰면 안 되나?

문제가 있다:
- 전송 중 전원 꺼지면 앱이 깨짐
- 부트로더도 못 들어가는 벽돌

버퍼에 먼저 받고 검증 후 복사하면:
- 전송 실패해도 기존 앱은 살아있음
- 복사 중 꺼지면... 그건 어쩔 수 없음

---

## 복사 함수

```c
void Copy_Buffer_To_App(void) {
    uint32_t src = BUFFER_ADDR;   // 0x08004000
    uint32_t dst = APP_ADDR;      // 0x08042800
    uint32_t size = g_app_size;
    
    Flash_Unlock();
    
    // 앱 영역 지우기
    for (uint32_t addr = dst; addr < dst + size; addr += 2048) {
        Flash_ErasePage(addr);
    }
    
    // 복사
    for (uint32_t i = 0; i < size; i += 2) {
        uint16_t data = *(uint16_t *)(src + i);
        Flash_WriteHalfWord(dst + i, data);
    }
    
    Flash_Lock();
}
```

---

## 페이지 단위 삭제

STM32F103은 페이지 단위(2KB)로 지운다. 앱 크기만큼 페이지 삭제.

```c
for (uint32_t addr = dst; addr < dst + size; addr += 2048) {
    Flash_ErasePage(addr);
}
```

---

## Half-Word 단위 쓰기

STM32 Flash는 16비트 단위로 쓴다.

```c
for (uint32_t i = 0; i < size; i += 2) {
    uint16_t data = *(uint16_t *)(src + i);
    Flash_WriteHalfWord(dst + i, data);
}
```

---

## 검증 후 버퍼 삭제?

이 부트로더는 버퍼를 안 지운다. 다음 업데이트 때 덮어쓰니까.

어떤 부트로더는 보안상 버퍼를 지우기도 한다.

---

Part 5 끝. 다음은 삽질 & 트러블슈팅.

[#23 - Thumb 모드 함정](/posts/ghidra/ghidra-stm32-re-23/)
