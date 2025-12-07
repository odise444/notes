---
title: "AD7280A BMS 개발 삽질기 #7 - 2개 연결했더니 통신이 안 된다"
date: 2024-12-07
draft: false
tags: ["BMS", "AD7280A", "임베디드", "배터리", "삽질", "데이지체인"]
categories: ["BMS 개발"]
summary: "1개일 때 잘 되던 게 2개 연결하니까 안 된다. 데이지체인 초기화 순서가 따로 있었다."
---

## 지난 글 요약

[지난 글](/posts/bms/ad7280a-bms-dev-06/)에서 Alert 설정을 잡았다. 이제 1개짜리 EVAL 보드는 완벽하게 동작한다. 

12셀이 아니라 24셀, 48셀을 모니터링하려면? AD7280A를 여러 개 데이지체인으로 연결해야 한다.

## 데이지체인 구조

AD7280A는 최대 8개까지 데이지체인으로 연결할 수 있다. 48셀 모니터링 가능.

![](ad7280a-bms-dev-07-1.png)

- **Dev 0**: 배터리 스택 하위 (Cell 1~6)
- **Dev 1**: 배터리 스택 상위 (Cell 7~12)
- SDO → SDI로 체인 연결

## 증상

EVAL 보드 2개를 연결했다. 1개일 때 잘 되던 코드를 그대로 돌렸더니:

```
Dev 0 Cell 1: 4012 mV ✓
Dev 0 Cell 2: 4008 mV ✓
...
Dev 0 Cell 6: 4015 mV ✓
Dev 1 Cell 1: 0 mV ✗
Dev 1 Cell 2: 0 mV ✗
...
Dev 1 Cell 6: 0 mV ✗
```

Dev 0은 잘 읽히는데 Dev 1이 전부 0이다.

## 삽질 1: 주소 할당을 안 했다

1개일 때는 주소가 0으로 고정이라 신경 안 썼다. 근데 2개 이상이면 **각 디바이스에 고유 주소를 할당**해야 한다.

AD7280A는 재밌는 자동 주소 할당 메커니즘이 있다.

```c
// Control LB 레지스터
#define AD7280A_CTRL_LB_INC_DEV_ADDR    (1 << 1)  // 주소 증가
#define AD7280A_CTRL_LB_LOCK_DEV_ADDR   (1 << 2)  // 주소 잠금
```

### 주소 할당 순서

1. **모든 디바이스에 INC_DEV_ADDR = 1 전송**
   - 각 디바이스가 주소를 1씩 증가시키며 전달
   - Dev 0 = 0, Dev 1 = 1, Dev 2 = 2...

2. **모든 디바이스에 LOCK_DEV_ADDR = 1 전송**
   - 주소 고정

```c
void ad7280a_assign_addresses(uint8_t num_devices) {
    // 1단계: 주소 증가 활성화
    uint8_t ctrl_lb = AD7280A_CTRL_LB_MUST_SET |
                      AD7280A_CTRL_LB_INC_DEV_ADDR;
    
    uint32_t cmd = ad7280a_crc_write(
        ((uint32_t)AD7280A_REG_CONTROL_LB << 21) |
        ((uint32_t)ctrl_lb << 13) |
        (1 << 12)  // All devices
    );
    
    // num_devices번 전송 (각 디바이스가 주소 증가)
    for (int i = 0; i < num_devices; i++) {
        ad7280a_transfer_32bits(cmd);
    }
    
    // 2단계: 주소 잠금
    ctrl_lb = AD7280A_CTRL_LB_MUST_SET |
              AD7280A_CTRL_LB_LOCK_DEV_ADDR;
    
    cmd = ad7280a_crc_write(
        ((uint32_t)AD7280A_REG_CONTROL_LB << 21) |
        ((uint32_t)ctrl_lb << 13) |
        (1 << 12)
    );
    ad7280a_transfer_32bits(cmd);
}
```

## 삽질 2: 데이지체인 리드백 활성화 안 함

주소 할당했는데도 Dev 1 데이터가 안 읽힌다. **DAISY_CHAIN_RB_EN** 비트를 켜야 했다.

```c
#define AD7280A_CTRL_LB_DAISY_CHAIN_RB_EN  (1 << 0)
```

이 비트가 0이면 데이터가 체인을 타고 MCU까지 전달되지 않는다.

```c
// Control LB 최종 설정
uint8_t ctrl_lb = AD7280A_CTRL_LB_MUST_SET |        // D4 = 1
                  AD7280A_ACQ_TIME_1600NS |          // D6:D5 = 11
                  AD7280A_CTRL_LB_LOCK_DEV_ADDR |    // D2 = 1
                  AD7280A_CTRL_LB_DAISY_CHAIN_RB_EN; // D0 = 1 ← 중요!
```

## 삽질 3: VIN6-VIN0 연결 문제

데이터시트 Figure 29를 보면, 데이지체인에서 **Dev 0의 VIN6**와 **Dev 1의 VIN0**가 **같은 전압**이어야 한다.

```
배터리 스택:
Cell 12 ─── Dev1-VIN6 (36V)
Cell 11
Cell 10
Cell 9
Cell 8
Cell 7  ─── Dev1-VIN0 = Dev0-VIN6 (18V) ← 같은 노드!
Cell 6  ─── Dev0-VIN6 (18V)
Cell 5
Cell 4
Cell 3
Cell 2
Cell 1  ─── Dev0-VIN0 (0V, GND)
```

EVAL 보드 2개를 테스트할 때 저항 분배기를 따로따로 만들었더니 이 연결이 끊어져 있었다.

![](static/imgs/ad7280a-resistor-divider.png)

**해결**: J2-7 (Dev0 VIN6)과 J3-1 (Dev1 VIN0)을 물리적으로 연결.

## 초기화 순서 정리

데이지체인 초기화는 순서가 중요하다:

```c
void ad7280a_daisy_chain_init(uint8_t num_devices) {
    // 0. 하드웨어 리셋 (PD 핀 토글)
    gpio_put(PIN_PD, 0);
    sleep_ms(1);
    gpio_put(PIN_PD, 1);
    sleep_ms(1);
    
    // 1. 소프트웨어 리셋 (모든 디바이스)
    ad7280a_software_reset();
    sleep_ms(1);
    
    // 2. 주소 할당
    ad7280a_assign_addresses(num_devices);
    
    // 3. Control LB 설정 (데이지체인 리드백 활성화)
    uint8_t ctrl_lb = AD7280A_CTRL_LB_MUST_SET |
                      AD7280A_ACQ_TIME_1600NS |
                      AD7280A_CTRL_LB_LOCK_DEV_ADDR |
                      AD7280A_CTRL_LB_DAISY_CHAIN_RB_EN;
    
    ad7280a_write_all(AD7280A_REG_CONTROL_LB, ctrl_lb);
    
    // 4. Control HB 설정 (변환 모드)
    uint8_t ctrl_hb = AD7280A_CONV_ALL |
                      AD7280A_CONV_AVG_8;
    
    ad7280a_write_all(AD7280A_REG_CONTROL_HB, ctrl_hb);
}
```

## 데이터 읽기 순서

데이지체인에서 데이터는 **Dev N → Dev N-1 → ... → Dev 0 → MCU** 순서로 나온다.

```c
void ad7280a_read_all_cells(uint8_t num_devices) {
    // 변환 시작
    gpio_put(PIN_CNVST, 0);
    sleep_us(1);
    gpio_put(PIN_CNVST, 1);
    sleep_us(50);  // 변환 완료 대기
    
    // 모든 디바이스의 모든 채널 읽기
    // 총 전송 횟수 = num_devices * 6 (셀) + 여유분
    for (int i = 0; i < num_devices * 8; i++) {
        uint32_t rx = ad7280a_transfer_32bits(AD7280A_READ_CMD);
        
        uint8_t dev_addr = (rx >> 27) & 0x1F;
        uint8_t reg_addr = (rx >> 23) & 0x0F;
        uint16_t data = (rx >> 11) & 0x0FFF;
        
        if (reg_addr <= 5) {  // Cell voltage register
            float voltage = data * 0.976f + 1000.0f;
            printf("Dev%d Cell%d: %.0f mV\n", 
                   dev_addr, reg_addr + 1, voltage);
        }
    }
}
```

## 결과

초기화 순서 맞추고 VIN6-VIN0 연결하니까:

```
Dev 0 Cell 1: 4012 mV ✓
Dev 0 Cell 2: 4008 mV ✓
...
Dev 0 Cell 6: 4015 mV ✓
Dev 1 Cell 1: 4005 mV ✓
Dev 1 Cell 2: 4011 mV ✓
...
Dev 1 Cell 6: 4009 mV ✓
```

12셀 전부 읽힌다!

## 삽질 포인트 정리

1. **주소 할당 필수** - INC_DEV_ADDR → LOCK_DEV_ADDR 순서
2. **DAISY_CHAIN_RB_EN = 1** - 이거 안 켜면 데이터 안 나옴
3. **VIN6-VIN0 연결** - 체인 사이 전압 노드 공유
4. **초기화 순서** - 리셋 → 주소 할당 → 설정

다음 글에서는 주소 할당 메커니즘을 더 자세히 파본다.

---

## 참고 자료

- [AD7280A Datasheet - Daisy-Chain Configuration (p.17, Figure 29)](https://www.analog.com/media/en/technical-documentation/data-sheets/ad7280a.pdf)
- [AD7280A Datasheet - Control Register (p.28-29)](https://www.analog.com/media/en/technical-documentation/data-sheets/ad7280a.pdf)
