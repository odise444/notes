---
title: "Ghidra STM32 역분석 #33 - 실제 장비 테스트"
date: 2024-12-14
draft: false
tags: ["Ghidra", "STM32", "리버스엔지니어링", "역분석", "테스트"]
categories: ["역분석"]
series: ["Ghidra 역분석"]
summary: "Python 업로더로 실제 BMS 장비에 펌웨어 업로드 테스트."
---

만든 Python 업로더가 실제로 동작하는지 테스트.

---

## 테스트 환경

- BMS 보드 (STM32F103VE)
- PCAN-USB
- Ubuntu 22.04
- 테스트 펌웨어 (LED 점멸)

---

## 테스트 1: 연결

```bash
$ python3 uploader.py test_firmware.bin
Connecting...
FwChk: 12345678, Date: 2020-04-29
Key: B7A5
Connected!
```

✅ 연결 성공. Key 계산도 맞음.

---

## 테스트 2: 업로드

```bash
Firmware size: 32768 bytes
Page 1 done
Page 2 done
...
Page 16 done
Verified!
Done! Device rebooting...
```

✅ 업로드 성공.

---

## 테스트 3: 동작 확인

재부팅 후 LED가 1초 간격으로 점멸한다.

✅ 펌웨어 정상 동작.

---

## 테스트 4: 롤백

다시 원래 펌웨어 업로드:

```bash
$ python3 uploader.py original_app.bin
```

✅ 정상 복구.

---

## 테스트 5: 에러 케이스

잘못된 Key 보내기:

```python
# 일부러 틀린 Key
self.send([0x31, 0x00, 0x00, 0, 0, 0, 0, 0])
```

```
Auth failed
```

✅ 잘못된 Key 거부.

---

## 테스트 6: 전송 중단

업로드 중간에 Ctrl+C:

```bash
Page 5 done
^C
```

재시작하면 부트로더 모드로 다시 진입. 기존 앱은 그대로.

✅ 버퍼 방식 덕분에 안전함.

---

## 결론

Python 업로더가 Windows EXE 업로더를 완전히 대체할 수 있다.

---

다음 글에서 부트로더 개선.

[#34 - 부트로더 개선](/posts/ghidra/ghidra-stm32-re-34/)
