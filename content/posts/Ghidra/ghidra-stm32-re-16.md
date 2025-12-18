---
title: "Ghidra STM32 역분석 #16 - Connection Key 알고리즘"
date: 2024-12-13
draft: false
tags: ["Ghidra", "STM32", "리버스엔지니어링", "역분석", "인증"]
categories: ["역분석"]
series: ["Ghidra 역분석"]
summary: "부트로더 인증의 핵심! Key 계산 알고리즘을 밝혀낸다."
---

PC가 Connection 요청하면 BMS가 FwChk와 FwDate를 보낸다. PC는 이걸로 Key를 계산해서 보내야 한다.

---

## CalcKey 함수

인증 처리 함수에서 호출하는 `FUN_08003D00`:

```c
uint32_t CalcKey(void) {
    uint8_t *fw_chk = (uint8_t *)0x20000100;
    uint8_t *fw_date = (uint8_t *)0x20000104;
    
    uint32_t key = 0;
    
    // FwChk XOR 연산
    key = fw_chk[0] ^ fw_chk[1];
    key = (key << 8) | (fw_chk[2] ^ fw_chk[3]);
    
    // FwDate 추가
    key ^= (fw_date[0] << 16);
    key ^= (fw_date[1] << 8);
    key ^= fw_date[2];
    
    // 상수 연산
    key = key * 0x1234 + 0x5678;
    key = (key >> 16) ^ (key & 0xFFFF);
    
    return key;
}
```

---

## Python으로 구현

```python
def calc_key(fw_chk, fw_date):
    key = fw_chk[0] ^ fw_chk[1]
    key = (key << 8) | (fw_chk[2] ^ fw_chk[3])
    
    key ^= (fw_date[0] << 16)
    key ^= (fw_date[1] << 8)
    key ^= fw_date[2]
    
    key = (key * 0x1234 + 0x5678) & 0xFFFFFFFF
    key = (key >> 16) ^ (key & 0xFFFF)
    
    return key

# 테스트
fw_chk = [0x12, 0x34, 0x56, 0x78]
fw_date = [0x20, 0x04, 0x29]
print(hex(calc_key(fw_chk, fw_date)))
```

CAN 스니핑 결과랑 비교해서 검증.

---

## 보안 수준

솔직히 보안 수준은 낮다. XOR + 곱셈 정도는 역분석으로 바로 알아낼 수 있다.

하지만 일반 사용자가 아무 펌웨어나 올리는 건 막을 수 있다.

---

다음 글에서 데이터 프레임 처리.

[#17 - 데이터 프레임 처리](/posts/ghidra/ghidra-stm32-re-17/)
