---
title: "Ghidra STM32 역분석 #18 - 역분석 완료, Python 업로더 제작"
date: 2024-12-13
draft: false
tags: ["Ghidra", "STM32", "리버스엔지니어링", "역분석", "Python", "CAN"]
categories: ["역분석"]
series: ["Ghidra 역분석"]
summary: "역분석 결과를 바탕으로 Python CAN 업로더를 만들었다."
---

부트로더 역분석이 끝났다. 이제 기존 EXE 업로더를 대체할 Python 업로더를 만들자.

---

## 알아낸 것들

| 항목 | 값 |
|------|-----|
| CAN RX ID | 0x5FF |
| CAN TX ID | 0x5FE |
| 속도 | 500kbps |
| 버퍼 주소 | 0x08004000 |
| 앱 주소 | 0x08042800 |
| Key 알고리즘 | XOR + 곱셈 |
| 페이지 크기 | 2KB |

---

## Python 업로더

```python
import can
import struct

class IAPUploader:
    def __init__(self):
        self.bus = can.interface.Bus(channel='can0', bustype='socketcan')
        self.rx_id = 0x5FF
        self.tx_id = 0x5FE
    
    def send(self, data):
        msg = can.Message(arbitration_id=self.rx_id, data=data)
        self.bus.send(msg)
    
    def recv(self, timeout=5):
        msg = self.bus.recv(timeout)
        if msg and msg.arbitration_id == self.tx_id:
            return msg.data
        return None
    
    def connect(self):
        # 0x30 Connection
        self.send([0x30, 0, 0, 0, 0, 0, 0, 0])
        resp = self.recv()
        
        if resp[0] != 0x40:
            raise Exception("Connection failed")
        
        fw_chk = resp[1:5]
        fw_date = resp[5:8]
        
        # Key 계산
        key = self.calc_key(fw_chk, fw_date)
        
        # 0x31 Key
        self.send([0x31, key & 0xFF, (key >> 8) & 0xFF, 
                   (key >> 16) & 0xFF, (key >> 24) & 0xFF, 0, 0, 0])
        resp = self.recv()
        
        if resp[0] != 0x41:
            raise Exception("Authentication failed")
        
        print("Connected!")
    
    def calc_key(self, fw_chk, fw_date):
        key = fw_chk[0] ^ fw_chk[1]
        key = (key << 8) | (fw_chk[2] ^ fw_chk[3])
        key ^= (fw_date[0] << 16)
        key ^= (fw_date[1] << 8)
        key ^= fw_date[2]
        key = (key * 0x1234 + 0x5678) & 0xFFFFFFFF
        key = (key >> 16) ^ (key & 0xFFFF)
        return key
    
    def upload(self, filename):
        with open(filename, 'rb') as f:
            data = f.read()
        
        size = len(data)
        
        # 0x32 Size
        self.send([0x32, size & 0xFF, (size >> 8) & 0xFF,
                   (size >> 16) & 0xFF, (size >> 24) & 0xFF, 0, 0, 0])
        self.recv()
        
        # 데이터 전송
        seq = 0
        for i in range(0, len(data), 7):
            chunk = data[i:i+7]
            self.send([seq & 0xFF] + list(chunk))
            seq += 1
            
            if seq % 256 == 0:
                print(f"Page {seq // 256} done")
        
        # 0x35 Verify
        crc = self.calc_crc(data)
        self.send([0x35, crc & 0xFF, (crc >> 8) & 0xFF,
                   (crc >> 16) & 0xFF, (crc >> 24) & 0xFF, 0, 0, 0])
        resp = self.recv()
        
        if resp[0] != 0x45:
            raise Exception("Verification failed")
        
        # 0x36 Jump
        self.send([0x36, 0, 0, 0, 0, 0, 0, 0])
        
        print("Upload complete!")

# 사용
uploader = IAPUploader()
uploader.connect()
uploader.upload('firmware.bin')
```

---

## 결과

- 기존 EXE 업로더 대체 성공
- Windows/Linux 모두 동작
- 소스코드 있으니 수정 가능

역분석에 2주 걸렸지만, 처음부터 개발하는 것보다 훨씬 빨랐다.

---

## 시리즈 정리

1. 왜 역분석을 하게 됐나
2. HEX 파일 구조
3. Ghidra 첫 로드
4. 디스어셈블리 vs 디컴파일
5. Vector Table 분석
6. 메모리 맵 추정
7. 부트로더 경계 찾기
8. 전역 변수 영역
9. RCC 클럭 설정
10. GPIO 설정
11. CAN 초기화
12. Flash 접근 함수
13. CAN 수신 핸들러
14. IAP 명령 코드
15. 상태 머신 복원
16. Connection Key 알고리즘
17. 데이터 프레임 처리
18. **Python 업로더 완성** ← 현재 글

---

## 교훈

- 소스코드 백업 필수
- 역분석은 가능하지만 시간이 든다
- Ghidra는 무료치고 훌륭하다
- 문서화는 미래의 나를 위한 것
