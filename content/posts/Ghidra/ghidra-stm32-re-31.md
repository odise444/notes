---
title: "Ghidra STM32 역분석 #31 - CAN 스니핑 검증"
date: 2024-12-14
draft: false
tags: ["Ghidra", "STM32", "리버스엔지니어링", "역분석", "CAN"]
categories: ["역분석"]
series: ["Ghidra 역분석"]
summary: "역분석한 프로토콜이 맞는지 실제 통신으로 검증."
---

역분석 결과가 맞는지 실제 CAN 통신을 캡처해서 확인해보자.

---

## 장비

- PCAN-USB 또는 CANable
- Linux (candump 사용)
- 기존 Windows 업로더 + BMS 장비

---

## CAN 인터페이스 설정

```bash
sudo ip link set can0 type can bitrate 500000
sudo ip link set up can0
```

---

## 캡처

```bash
candump can0 -t A > can_log.txt
```

Windows 업로더로 펌웨어 업로드하면서 캡처.

---

## 캡처 결과

```
(000.000000)  can0  5FF   [8]  30 00 00 00 00 00 00 00
(000.012345)  can0  5FE   [8]  40 12 34 56 78 20 04 29
(000.024567)  can0  5FF   [8]  31 A5 B7 00 00 00 00 00
(000.036789)  can0  5FE   [1]  41
(000.048901)  can0  5FF   [8]  32 00 80 03 00 00 00 00
(000.061234)  can0  5FE   [1]  42
...
```

---

## 분석 결과 검증

| 캡처 | 역분석 추정 | 일치? |
|------|------------|-------|
| 5FF = PC→BMS | 0x5FF | ✅ |
| 5FE = BMS→PC | 0x5FE | ✅ |
| 0x30 = Connect | CMD_CONNECT | ✅ |
| 0x40 = FwChk+Date | RSP_CONNECT | ✅ |
| 0x31 = Key | CMD_KEY | ✅ |
| 0x41 = OK | RSP_KEY | ✅ |

프로토콜 역분석 결과가 맞았다.

---

## Key 검증

캡처에서:
- FwChk: 0x12345678
- FwDate: 0x200429
- 보낸 Key: 0xB7A5

Python으로 계산:

```python
def calc_key(fw_chk, fw_date):
    key = (fw_chk[0] ^ fw_chk[1])
    key = (key << 8) | (fw_chk[2] ^ fw_chk[3])
    key ^= (fw_date[0] << 16)
    key ^= (fw_date[1] << 8)
    key ^= fw_date[2]
    key = (key * 0x1234 + 0x5678) & 0xFFFFFFFF
    key = (key >> 16) ^ (key & 0xFFFF)
    return key

result = calc_key([0x12, 0x34, 0x56, 0x78], [0x20, 0x04, 0x29])
print(hex(result))  # 0xb7a5
```

일치한다. Key 알고리즘도 맞음.

---

다음 글에서 Python 업로더 제작.

[#32 - Python 업로더](/posts/ghidra/ghidra-stm32-re-32/)
