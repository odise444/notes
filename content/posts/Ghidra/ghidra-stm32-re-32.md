---
title: "Ghidra STM32 역분석 #32 - Python 업로더"
date: 2024-12-14
draft: false
tags: ["Ghidra", "STM32", "리버스엔지니어링", "역분석", "Python", "CAN"]
categories: ["역분석"]
series: ["Ghidra 역분석"]
summary: "역분석 결과로 Python CAN 업로더 제작."
---

Windows EXE 업로더 대신 Python으로 직접 만들자. Linux에서도 쓸 수 있고 수정도 가능.

---

## 의존성

```bash
pip install python-can
```

---

## 전체 코드

```python
#!/usr/bin/env python3
import can
import struct
import time
import sys

class BMSUploader:
    RX_ID = 0x5FF  # PC → BMS
    TX_ID = 0x5FE  # BMS → PC
    
    def __init__(self, channel='can0', bustype='socketcan'):
        self.bus = can.interface.Bus(channel=channel, bustype=bustype)
    
    def send(self, data):
        msg = can.Message(arbitration_id=self.RX_ID, data=data, is_extended_id=False)
        self.bus.send(msg)
    
    def recv(self, timeout=5):
        msg = self.bus.recv(timeout)
        if msg and msg.arbitration_id == self.TX_ID:
            return list(msg.data)
        return None
    
    def calc_key(self, fw_chk, fw_date):
        key = fw_chk[0] ^ fw_chk[1]
        key = (key << 8) | (fw_chk[2] ^ fw_chk[3])
        key ^= (fw_date[0] << 16)
        key ^= (fw_date[1] << 8)
        key ^= fw_date[2]
        key = (key * 0x1234 + 0x5678) & 0xFFFFFFFF
        key = (key >> 16) ^ (key & 0xFFFF)
        return key
    
    def connect(self):
        print("Connecting...")
        self.send([0x30, 0, 0, 0, 0, 0, 0, 0])
        
        resp = self.recv()
        if not resp or resp[0] != 0x40:
            raise Exception("Connection failed")
        
        fw_chk = resp[1:5]
        fw_date = resp[5:8]
        print(f"FwChk: {bytes(fw_chk).hex()}, Date: 20{fw_date[0]:02d}-{fw_date[1]:02d}-{fw_date[2]:02d}")
        
        key = self.calc_key(fw_chk, fw_date)
        print(f"Key: {key:04X}")
        
        self.send([0x31, key & 0xFF, (key >> 8) & 0xFF, 0, 0, 0, 0, 0])
        
        resp = self.recv()
        if not resp or resp[0] != 0x41:
            raise Exception("Auth failed")
        
        print("Connected!")
    
    def upload(self, filename):
        with open(filename, 'rb') as f:
            data = f.read()
        
        size = len(data)
        print(f"Firmware size: {size} bytes")
        
        # Size 전송
        self.send([0x32, size & 0xFF, (size >> 8) & 0xFF, 
                   (size >> 16) & 0xFF, (size >> 24) & 0xFF, 0, 0, 0])
        self.recv()
        
        # 데이터 전송
        seq = 0
        for i in range(0, len(data), 7):
            chunk = list(data[i:i+7])
            while len(chunk) < 7:
                chunk.append(0xFF)
            
            self.send([seq & 0xFF] + chunk)
            seq += 1
            
            if seq % 256 == 0:
                resp = self.recv(timeout=10)
                if resp and resp[0] == 0x43:
                    print(f"Page {seq // 256} done")
        
        # Verify
        crc = self.calc_crc(data)
        self.send([0x35, crc & 0xFF, (crc >> 8) & 0xFF,
                   (crc >> 16) & 0xFF, (crc >> 24) & 0xFF, 0, 0, 0])
        
        resp = self.recv(timeout=30)
        if not resp or resp[0] != 0x45:
            raise Exception("Verify failed")
        
        print("Verified!")
        
        # Jump
        self.send([0x36, 0, 0, 0, 0, 0, 0, 0])
        print("Done! Device rebooting...")
    
    def calc_crc(self, data):
        # STM32 CRC-32 (polynomial: 0x04C11DB7)
        crc = 0xFFFFFFFF
        for i in range(0, len(data), 4):
            word = struct.unpack('<I', data[i:i+4].ljust(4, b'\xFF'))[0]
            crc ^= word
            for _ in range(32):
                if crc & 0x80000000:
                    crc = (crc << 1) ^ 0x04C11DB7
                else:
                    crc <<= 1
                crc &= 0xFFFFFFFF
        return crc

if __name__ == '__main__':
    if len(sys.argv) < 2:
        print(f"Usage: {sys.argv[0]} <firmware.bin>")
        sys.exit(1)
    
    uploader = BMSUploader()
    uploader.connect()
    uploader.upload(sys.argv[1])
```

---

## 사용법

```bash
# CAN 인터페이스 설정
sudo ip link set can0 type can bitrate 500000
sudo ip link set up can0

# 업로드
python3 uploader.py firmware.bin
```

---

다음 글에서 실제 장비 테스트.

[#33 - 실제 장비 테스트](/posts/ghidra/ghidra-stm32-re-33/)
