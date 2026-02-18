---
title: "STM32 부트로더 개발기 #17 - Python 업로더"
date: 2024-12-17
tags: ["STM32", "Bootloader", "Python", "CAN"]
categories: ["개발기"]
series: ["STM32 부트로더 개발기"]
summary: "PC에서 CAN으로 펌웨어 전송하는 Python 스크립트."
---

부트로더는 됐고, 이제 PC 쪽 업로더.

python-can 라이브러리 사용.

---

## 설치

```bash
pip install python-can
```

PCAN-USB 쓰면:
```bash
pip install python-can[pcan]
```

---

## 기본 구조

```python
import can
import struct
import time
from pathlib import Path

class BMSUploader:
    CAN_ID_TX = 0x5FF
    CAN_ID_RX = 0x5FE
    PAGE_SIZE = 2048
    
    def __init__(self, interface='pcan', channel='PCAN_USBBUS1', bitrate=500000):
        self.bus = can.interface.Bus(
            interface=interface,
            channel=channel,
            bitrate=bitrate
        )
    
    def send(self, data):
        msg = can.Message(arbitration_id=self.CAN_ID_TX, data=data, is_extended_id=False)
        self.bus.send(msg)
    
    def recv(self, timeout=5.0):
        msg = self.bus.recv(timeout=timeout)
        if msg and msg.arbitration_id == self.CAN_ID_RX:
            return msg.data
        return None
```

---

## 연결 및 인증

```python
def connect(self):
    # CONNECT 전송
    self.send([0x30])
    
    # CHALLENGE 대기
    resp = self.recv()
    if resp is None or resp[0] != 0x40:
        raise Exception("연결 실패")
    
    seed = struct.unpack('<I', bytes(resp[1:5]))[0]
    print(f"Seed: 0x{seed:08X}")
    
    # Key 계산
    key = self.calc_key(seed)
    print(f"Key: 0x{key:08X}")
    
    # AUTH 전송
    auth_data = [0x31] + list(struct.pack('<I', key))
    self.send(auth_data)
    
    # AUTH_OK 대기
    resp = self.recv()
    if resp is None or resp[0] != 0x41:
        raise Exception("인증 실패")
    
    print("인증 성공")

def calc_key(self, seed):
    # 간단한 XOR (실제론 더 복잡하게)
    return seed ^ 0x12345678
```

---

## 펌웨어 업로드

```python
def upload(self, filepath):
    data = Path(filepath).read_bytes()
    fw_size = len(data)
    total_pages = (fw_size + self.PAGE_SIZE - 1) // self.PAGE_SIZE
    
    print(f"펌웨어 크기: {fw_size} bytes ({total_pages} pages)")
    
    # SIZE_REQ 대기
    resp = self.recv()
    if resp[0] != 0x42:
        raise Exception("크기 요청 안 옴")
    
    # SIZE 응답
    size_data = [0x32] + list(struct.pack('<I', fw_size))
    self.send(size_data)
    
    # 페이지 전송
    for page_num in range(total_pages):
        self.send_page(data, page_num, fw_size)
        progress = (page_num + 1) * 100 // total_pages
        print(f"\r진행률: {progress}%", end='')
    
    print("\n업로드 완료")
```

---

## 페이지 전송

```python
def send_page(self, data, page_num, fw_size):
    # PAGE_REQ 대기
    resp = self.recv(timeout=10)
    if resp is None or resp[0] != 0x43:
        raise Exception(f"페이지 요청 안 옴: {resp}")
    
    start = page_num * self.PAGE_SIZE
    end = min(start + self.PAGE_SIZE, fw_size)
    page_data = data[start:end]
    
    # CRC 계산
    crc = self.crc32(page_data)
    
    # PAGE_START
    self.send([0x33, page_num & 0xFF, (page_num >> 8) & 0xFF])
    
    # 데이터 전송 (8바이트씩)
    for i in range(0, len(page_data), 8):
        chunk = page_data[i:i+8]
        self.send(list(chunk))
        time.sleep(0.001)  # 1ms 딜레이
    
    # PAGE_END + CRC
    end_data = [0x34] + list(struct.pack('<I', crc))
    self.send(end_data)
    
    # PAGE_ACK 대기
    resp = self.recv(timeout=10)
    if resp is None or resp[0] != 0x44:
        raise Exception(f"페이지 ACK 실패: {resp}")
```

---

## CRC32

```python
import binascii

def crc32(self, data):
    return binascii.crc32(data) & 0xFFFFFFFF
```

Python binascii가 IEEE CRC32 사용.

---

## 메인

```python
if __name__ == '__main__':
    import sys
    
    if len(sys.argv) < 2:
        print("Usage: python uploader.py firmware.bin")
        sys.exit(1)
    
    uploader = BMSUploader()
    
    try:
        uploader.connect()
        uploader.upload(sys.argv[1])
    except Exception as e:
        print(f"에러: {e}")
    finally:
        uploader.bus.shutdown()
```

---

## 실행

```bash
python uploader.py app.bin
```

```
Seed: 0xA5B6C7D8
Key: 0xB783F1A0
인증 성공
펌웨어 크기: 51234 bytes (26 pages)
진행률: 100%
업로드 완료
```

---

다음 글에서 테스트 및 검증.

[#18 - 테스트 및 검증](/posts/stm32-bootloader/stm32-bootloader-18/)
