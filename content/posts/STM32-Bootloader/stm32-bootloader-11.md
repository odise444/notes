---
title: "STM32 부트로더 개발기 #11 - Half-Word Program"
date: 2024-12-17
tags: ["STM32", "Bootloader", "Flash", "프로그래밍"]
categories: ["개발기"]
series: ["STM32 부트로더 개발기"]
summary: "STM32F103은 2바이트씩만 쓸 수 있다."
---

Flash 지웠다. 이제 쓰자.

근데 STM32F103은 **Half-Word (2바이트)** 단위로만 쓸 수 있다.

---

## Half-Word 제약

1바이트만 쓰고 싶어도 2바이트 써야 함.

홀수 주소에 쓰기 불가.

```c
// OK
*(uint16_t *)0x08004000 = 0x1234;

// NG - 홀수 주소
*(uint16_t *)0x08004001 = 0x1234;  // HardFault
```

---

## Program 절차

1. Unlock
2. `PG` 비트 설정 (Programming)
3. 주소에 Half-Word 쓰기
4. BSY 대기
5. `PG` 클리어

---

## 코드

```c
bool flash_program_halfword(uint32_t addr, uint16_t data) {
    // 주소 정렬 체크
    if (addr & 1) {
        return false;
    }
    
    // BSY 대기
    while (FLASH->SR & FLASH_SR_BSY);
    
    // Programming 모드
    FLASH->CR |= FLASH_CR_PG;
    
    // Half-Word 쓰기
    *(volatile uint16_t *)addr = data;
    
    // 완료 대기
    while (FLASH->SR & FLASH_SR_BSY);
    
    // PG 클리어
    FLASH->CR &= ~FLASH_CR_PG;
    
    // 에러 체크
    if (FLASH->SR & (FLASH_SR_PGERR | FLASH_SR_WRPRTERR)) {
        return false;
    }
    
    // 검증
    if (*(volatile uint16_t *)addr != data) {
        return false;
    }
    
    return true;
}
```

---

## 버퍼 쓰기

바이트 배열을 Flash에 쓰기:

```c
bool flash_program_buffer(uint32_t addr, uint8_t *data, uint32_t len) {
    // 짝수 길이로 맞춤
    uint32_t aligned_len = (len + 1) & ~1;
    
    for (uint32_t i = 0; i < aligned_len; i += 2) {
        uint16_t halfword;
        
        if (i + 1 < len) {
            halfword = data[i] | (data[i + 1] << 8);
        } else {
            halfword = data[i] | 0xFF00;  // 패딩
        }
        
        if (!flash_program_halfword(addr + i, halfword)) {
            return false;
        }
    }
    
    return true;
}
```

리틀 엔디안 주의.

---

## HAL 함수

```c
HAL_FLASH_Program(FLASH_TYPEPROGRAM_HALFWORD, addr, data);
```

Word(4바이트)도 있지만 내부적으로 Half-Word 두 번 씀.

---

## 쓰기 시간

Half-Word 하나 쓰는 데 약 40~70µs.

페이지 2KB = 1024 Half-Words ≈ 50~70ms.

---

## 주의사항

### 1. 같은 주소 두 번 쓰기 금지

한번 쓴 주소에 또 쓰면 에러.

지우고 다시 써야 함.

### 2. 정렬

주소가 2의 배수여야 함.

### 3. 쓰기 전 0xFF 확인

이미 값이 있으면 PGERR 발생.

```c
if (*(volatile uint16_t *)addr != 0xFFFF) {
    // 이미 프로그래밍됨 - 지워야 함
    return false;
}
```

---

다음 글에서 에러 처리.

[#12 - 에러 처리](/posts/stm32-bootloader/stm32-bootloader-12/)
