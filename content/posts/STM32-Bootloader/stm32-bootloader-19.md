---
title: "STM32 부트로더 개발기 #19 - 보안 고려사항"
date: 2024-12-17
tags: ["STM32", "Bootloader", "보안", "암호화"]
categories: ["개발기"]
series: ["STM32 부트로더 개발기"]
summary: "아무나 펌웨어 올리면 안 되니까."
---

지금 부트로더는 보안이 약하다.

실제 제품에선 신경 써야 할 것들.

---

## 현재 취약점

1. **인증**: 단순 XOR - 키 알아내기 쉬움
2. **펌웨어 암호화 없음** - 바이너리 그대로 전송
3. **Flash 보호 없음** - ST-Link로 읽기 가능
4. **디버그 포트 열림** - SWD로 접근 가능

---

## 인증 강화

### HMAC-SHA256

```c
// 단순 XOR 대신
uint8_t calc_key(uint8_t *seed, uint8_t *secret_key) {
    // HMAC-SHA256(secret_key, seed)
    // mbedTLS 또는 직접 구현
}
```

secret_key는 부트로더에 하드코딩 (읽기 보호 필수).

### Challenge-Response

```
1. BMS: 랜덤 challenge 전송
2. PC: HMAC(secret, challenge) 계산해서 전송
3. BMS: 검증
```

---

## 펌웨어 암호화

### AES-128

```
빌드 후:
plain.bin → AES 암호화 → encrypted.bin

전송:
PC: encrypted.bin 전송
BMS: 수신 → AES 복호화 → Flash에 쓰기
```

키 관리가 문제. 부트로더에 키 있어야 함.

---

## Read Protection (RDP)

STM32 Flash 읽기 보호.

```c
// Option Bytes로 설정
FLASH_OBProgramInitTypeDef ob;
HAL_FLASHEx_OBGetConfig(&ob);
ob.RDPLevel = OB_RDP_LEVEL_1;  // Level 1
HAL_FLASHEx_OBProgram(&ob);
```

| Level | 설명 |
|-------|------|
| 0 | 보호 없음 |
| 1 | 디버거로 Flash 읽기 불가, 해제 시 전체 삭제 |
| 2 | 영구 잠금 (복구 불가) |

**Level 1 추천**. Level 2는 개발 중 실수하면 답 없음.

---

## Write Protection

부트로더 영역 쓰기 보호.

```c
// 0~3 페이지 쓰기 보호 (16KB)
ob.WRPPage = OB_WRP_PAGES0TO3;
ob.WRPState = OB_WRPSTATE_ENABLE;
HAL_FLASHEx_OBProgram(&ob);
```

실수로 부트로더 덮어쓰기 방지.

---

## 디버그 비활성화

SWD 포트 막기:

```c
// GPIO를 일반 IO로 재설정
__HAL_AFIO_REMAP_SWJ_DISABLE();
```

또는 RDP Level 1/2 설정하면 자동으로 막힘.

---

## 서명 검증

펌웨어에 디지털 서명 추가.

```
빌드:
firmware.bin + RSA 개인키 → signature 생성
firmware.bin + signature 패키징

검증:
BMS: RSA 공개키로 signature 검증
```

공개키는 부트로더에 내장. 개인키는 빌드 서버에만.

구현 복잡. 대부분 제품에선 오버스펙.

---

## 실용적인 보안 수준

제품 특성에 따라:

**저보안** (내부용, 프로토타입):
- 간단한 인증만
- RDP Level 0

**중보안** (일반 제품):
- HMAC 인증
- RDP Level 1
- 부트로더 쓰기 보호

**고보안** (금융, 의료):
- 펌웨어 암호화
- 서명 검증
- RDP Level 2
- Secure Boot

---

다음 글에서 트러블슈팅 모음.

[#20 - 트러블슈팅 모음](/posts/stm32-bootloader/stm32-bootloader-20/)
