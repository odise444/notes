---
title: "STM32 부트로더 개발기 #18 - 테스트 및 검증"
date: 2024-12-17
tags: ["STM32", "Bootloader", "테스트", "검증"]
categories: ["개발기"]
series: ["STM32 부트로더 개발기"]
summary: "부트로더가 제대로 동작하는지 테스트."
---

부트로더 다 만들었다. 이제 테스트.

---

## 테스트 케이스

1. **정상 업로드** - 처음부터 끝까지
2. **앱 점프** - 업로드 후 앱 실행
3. **부트 진입** - 버튼/CAN/RAM 플래그
4. **에러 복구** - 전송 중단, CRC 실패
5. **앱 무효** - 앱 없을 때 부트로더 유지

---

## 테스트 1: 정상 업로드

```bash
python uploader.py test_app.bin
```

확인사항:
- [ ] 연결 성공
- [ ] 인증 통과
- [ ] 모든 페이지 전송
- [ ] CRC 검증 통과
- [ ] 완료 메시지

---

## 테스트 2: 앱 점프

업로드 후 리셋.

```c
// 앱 main()에 확인용 출력
int main(void) {
    HAL_Init();
    SystemClock_Config();
    
    printf("App started! Version: 1.0.0\n");
    
    while (1) {
        HAL_GPIO_TogglePin(LED_GPIO_Port, LED_Pin);
        HAL_Delay(500);
    }
}
```

LED 깜빡이면 성공.

---

## 테스트 3: 부트 진입 조건

### GPIO (버튼)

1. 버튼 누른 채로 전원 ON
2. 부트로더 진입 확인 (LED 패턴 다름)

### RAM 플래그

앱에서:
```c
void enter_bootloader_cmd(void) {
    *(volatile uint32_t *)0x2000FFF0 = 0xDEADBEEF;
    NVIC_SystemReset();
}
```

### CAN 명령

3초 내 CONNECT (0x30) 보내면 부트로더 유지.

---

## 테스트 4: 에러 복구

### 전송 중단

업로드 중간에 CAN 케이블 뽑기.

- 타임아웃 후 부트로더 초기 상태로
- 다시 연결하면 처음부터

### CRC 실패

일부러 데이터 변조:
```python
# 테스트용: 데이터 1바이트 변경
page_data = bytearray(page_data)
page_data[0] ^= 0xFF
```

- PAGE_ACK 대신 ERROR 와야 함
- 재전송 요청

---

## 테스트 5: 앱 무효

1. Flash 전체 삭제 (부트로더 제외)
2. 전원 ON
3. 부트로더 유지되어야 함 (앱 점프 안 함)

---

## 디버깅 팁

### UART 로그

부트로더에 UART printf 추가:

```c
#define DEBUG_UART huart1

void debug_print(const char *msg) {
    HAL_UART_Transmit(&DEBUG_UART, (uint8_t *)msg, strlen(msg), 100);
}

// 사용
debug_print("Bootloader started\n");
debug_print("Waiting for connection...\n");
```

### CAN 스니핑

PCAN-View로 CAN 메시지 모니터링.

```
Time     ID     Data
0.000    5FF    30                    <- CONNECT
0.001    5FE    40 12 34 56 78        <- CHALLENGE
0.002    5FF    31 AA BB CC DD        <- AUTH
0.003    5FE    41                    <- AUTH_OK
```

---

## 체크리스트

| 항목 | 결과 |
|------|------|
| 정상 업로드 | ☐ |
| 앱 실행 | ☐ |
| GPIO 부트 진입 | ☐ |
| RAM 플래그 부트 진입 | ☐ |
| CAN 부트 진입 | ☐ |
| 전송 중단 복구 | ☐ |
| CRC 실패 재전송 | ☐ |
| 앱 무효 시 부트 유지 | ☐ |

---

다음 글에서 보안 고려사항.

[#19 - 보안 고려사항](/posts/stm32-bootloader/stm32-bootloader-19/)
