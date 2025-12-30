---
title: "STM32 부트로더 개발기 #8 - 부트 진입 조건"
date: 2024-12-17
tags: ["STM32", "Bootloader", "GPIO", "진입조건"]
categories: ["개발기"]
series: ["STM32 부트로더 개발기"]
summary: "언제 부트로더에 머물고, 언제 앱으로 점프할지."
---

부트로더는 항상 먼저 실행된다.

근데 매번 부트로더에서 멈추면 안 됨. 정상 상황에선 앱으로 바로 가야 함.

---

## 진입 조건 종류

1. **GPIO 핀** - 특정 버튼 누르고 있으면
2. **CAN 메시지** - 특정 메시지 수신하면
3. **타임아웃** - 일정 시간 내 명령 없으면 앱으로
4. **RAM 플래그** - 소프트웨어 리셋으로 진입
5. **앱 무효** - 앱이 없거나 손상됐으면

---

## GPIO 방식

```c
#define BOOT_PIN       GPIO_PIN_2
#define BOOT_GPIO_PORT GPIOB

bool check_boot_pin(void) {
    // GPIO 클럭 활성화
    __HAL_RCC_GPIOB_CLK_ENABLE();
    
    // 풀업 입력 설정
    GPIO_InitTypeDef gpio = {0};
    gpio.Pin = BOOT_PIN;
    gpio.Mode = GPIO_MODE_INPUT;
    gpio.Pull = GPIO_PULLUP;
    HAL_GPIO_Init(BOOT_GPIO_PORT, &gpio);
    
    // LOW면 부트 진입 (버튼 눌림)
    return HAL_GPIO_ReadPin(BOOT_GPIO_PORT, BOOT_PIN) == GPIO_PIN_RESET;
}
```

전원 켤 때 버튼 누르고 있으면 부트로더 모드.

---

## RAM 플래그 방식

앱에서 소프트웨어 리셋으로 부트로더 진입하고 싶을 때.

```c
// 링커 스크립트에서 특정 주소 예약
// 또는 RAM 맨 끝 주소 사용
#define BOOT_FLAG_ADDR  0x2000FFF0
#define BOOT_MAGIC      0xDEADBEEF

// 부트로더에서 체크
bool check_ram_flag(void) {
    uint32_t flag = *(volatile uint32_t *)BOOT_FLAG_ADDR;
    
    if (flag == BOOT_MAGIC) {
        // 플래그 클리어
        *(volatile uint32_t *)BOOT_FLAG_ADDR = 0;
        return true;
    }
    return false;
}

// 앱에서 부트로더 진입할 때
void enter_bootloader(void) {
    *(volatile uint32_t *)BOOT_FLAG_ADDR = BOOT_MAGIC;
    NVIC_SystemReset();  // 소프트웨어 리셋
}
```

소프트웨어 리셋해도 RAM은 유지됨.

---

## CAN 메시지 방식

부팅 후 일정 시간 동안 특정 CAN 메시지 기다림.

```c
bool wait_for_can_boot_command(uint32_t timeout_ms) {
    uint32_t start = HAL_GetTick();
    
    while ((HAL_GetTick() - start) < timeout_ms) {
        CAN_RxHeaderTypeDef rx_header;
        uint8_t rx_data[8];
        
        if (HAL_CAN_GetRxFifoFillLevel(&hcan, CAN_RX_FIFO0) > 0) {
            HAL_CAN_GetRxMessage(&hcan, CAN_RX_FIFO0, &rx_header, rx_data);
            
            // 부트 진입 명령 (예: ID 0x5FF, data[0] = 0x30)
            if (rx_header.StdId == 0x5FF && rx_data[0] == 0x30) {
                return true;
            }
        }
    }
    return false;
}
```

---

## 타임아웃 방식

```c
#define BOOT_TIMEOUT_MS  3000  // 3초

void bootloader_main(void) {
    // 앱 유효하지 않으면 부트로더에 머묾
    if (!is_app_valid()) {
        run_bootloader();
        return;
    }
    
    // GPIO 체크
    if (check_boot_pin()) {
        run_bootloader();
        return;
    }
    
    // RAM 플래그 체크
    if (check_ram_flag()) {
        run_bootloader();
        return;
    }
    
    // CAN 명령 대기 (타임아웃)
    if (wait_for_can_boot_command(BOOT_TIMEOUT_MS)) {
        run_bootloader();
        return;
    }
    
    // 조건 없으면 앱으로
    jump_to_app();
}
```

---

## 내가 쓰는 조합

1. **앱 무효** → 부트로더
2. **GPIO (PB2) LOW** → 부트로더
3. **RAM 매직값** → 부트로더
4. **3초 내 CAN 명령** → 부트로더
5. **그 외** → 앱 점프

현장에서 여러 상황에 대응 가능.

---

다음 글에서 Flash Unlock/Lock.

[#9 - Flash Unlock/Lock](/posts/stm32-bootloader/stm32-bootloader-09/)
