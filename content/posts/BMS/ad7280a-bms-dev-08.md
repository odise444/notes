---
title: "AD7280A BMS 개발 삽질기 #8 - Device Address 할당의 비밀"
date: 2024-12-08
draft: false
tags: ["BMS", "AD7280A", "임베디드", "배터리", "삽질", "데이지체인"]
categories: ["BMS 개발"]
summary: "주소가 자동으로 할당된다고? 그 메커니즘을 파헤쳐봤다."
---

## 지난 글 요약

[지난 글](/posts/bms/ad7280a-bms-dev-07/)에서 데이지체인 2개를 연결했다. 주소 할당이 필요하다는 건 알았는데, 정확히 어떻게 동작하는 걸까?

## 주소 할당 메커니즘

AD7280A는 **자동 주소 할당** 기능이 있다. MCU가 각 디바이스에 개별적으로 주소를 써줄 필요가 없다.

핵심은 Control LB 레지스터의 두 비트:

| 비트 | 이름 | 기능 |
|------|------|------|
| D1 | INC_DEV_ADDR | 주소 증가 모드 |
| D2 | LOCK_DEV_ADDR | 주소 잠금 |

## 동작 원리

### 초기 상태

전원 인가 후 모든 디바이스는 **주소 0**으로 시작한다.

```
Dev 0: addr = 0
Dev 1: addr = 0
Dev 2: addr = 0
```

### INC_DEV_ADDR = 1 전송

`INC_DEV_ADDR` 비트가 1인 명령을 보내면:

1. **Dev 0**이 명령을 받음
2. Dev 0이 자기 주소를 **+1** 증가
3. 명령을 **다음 디바이스로 전달**
4. **Dev 1**이 받고 주소 +1
5. 반복...

```
1차 전송 후:
Dev 0: addr = 1
Dev 1: addr = 1
Dev 2: addr = 1

2차 전송 후:
Dev 0: addr = 2
Dev 1: addr = 2
Dev 2: addr = 2

3차 전송 후:
Dev 0: addr = 3
Dev 1: addr = 3
Dev 2: addr = 3
```

어? 다 같은 주소네? 이러면 안 되는데...

## 진짜 동작 방식

내가 처음에 잘못 이해했다. 실제로는 **체인을 따라 전파되면서** 주소가 할당된다.

```
전원 인가 직후:
MCU ─── Dev0(addr=0) ─── Dev1(addr=0) ─── Dev2(addr=0)

INC_DEV_ADDR=1 첫 번째 전송:
- Dev0: 명령 받음 → addr = 0 유지, 다음으로 전달
- Dev1: 명령 받음 → addr = 1로 증가
- Dev2: 명령 받음 → addr = 2로 증가

결과:
MCU ─── Dev0(addr=0) ─── Dev1(addr=1) ─── Dev2(addr=2)
```

아! **첫 번째 디바이스(Dev0)는 주소가 안 바뀌고**, 체인 뒤쪽으로 갈수록 주소가 증가한다.

## 코드로 확인

```c
void ad7280a_assign_addresses(uint8_t num_devices) {
    // INC_DEV_ADDR = 1 전송
    uint8_t ctrl_lb = AD7280A_CTRL_LB_MUST_SET |
                      AD7280A_CTRL_LB_INC_DEV_ADDR;
    
    uint32_t cmd = ad7280a_crc_write(
        ((uint32_t)AD7280A_REG_CONTROL_LB << 21) |
        ((uint32_t)ctrl_lb << 13) |
        (1 << 12)  // All devices
    );
    
    // 한 번만 전송하면 됨!
    ad7280a_transfer_32bits(cmd);
    
    // 주소 잠금
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

## 삽질: 여러 번 전송하면?

처음에 `num_devices`번 전송해야 하는 줄 알았다.

```c
// 잘못된 코드
for (int i = 0; i < num_devices; i++) {
    ad7280a_transfer_32bits(cmd);  // 3번 전송
}
```

결과:
```
Dev0: addr = 0
Dev1: addr = 3  // 1+1+1
Dev2: addr = 6  // 2+2+2
```

주소가 이상하게 할당된다! **한 번만 전송**해야 맞다.

## LOCK_DEV_ADDR의 중요성

주소 할당 후 **반드시 잠금**해야 한다.

```c
ctrl_lb = AD7280A_CTRL_LB_MUST_SET |
          AD7280A_CTRL_LB_LOCK_DEV_ADDR;
```

잠금 안 하면:
- 이후 통신에서 주소가 또 바뀔 수 있음
- 데이터 읽기/쓰기가 엉뚱한 디바이스로 갈 수 있음

## 주소 확인 방법

할당된 주소가 맞는지 확인하려면 각 디바이스의 레지스터를 읽어본다.

```c
void ad7280a_verify_addresses(uint8_t num_devices) {
    for (uint8_t addr = 0; addr < num_devices; addr++) {
        // 특정 주소의 Control LB 읽기
        uint32_t cmd = ad7280a_crc_write(
            ((uint32_t)addr << 27) |           // Device address
            ((uint32_t)AD7280A_REG_READ << 21) |
            ((uint32_t)AD7280A_REG_CONTROL_LB << 13)
        );
        ad7280a_transfer_32bits(cmd);
        
        uint32_t rx = ad7280a_transfer_32bits(AD7280A_NOP);
        uint8_t rx_addr = (rx >> 27) & 0x1F;
        
        printf("Expected: %d, Got: %d %s\n", 
               addr, rx_addr, 
               (addr == rx_addr) ? "✓" : "✗");
    }
}
```

## 주소 체계 정리

| 물리 위치 | 주소 | 셀 범위 |
|-----------|------|---------|
| 스택 최하위 (GND 근처) | 0 | Cell 1~6 |
| 중간 | 1 | Cell 7~12 |
| 중간 | 2 | Cell 13~18 |
| ... | ... | ... |
| 스택 최상위 (고전압) | N-1 | Cell (N*6-5)~(N*6) |

**주소 0이 항상 GND 쪽 디바이스**라는 걸 기억하자.

## 특정 디바이스만 제어하기

주소가 할당되면 개별 디바이스를 제어할 수 있다.

```c
// Dev 1의 셀 밸런싱만 켜기
void ad7280a_balance_dev1_cell3(void) {
    uint8_t balance = (1 << 2);  // Cell 3 밸런싱
    
    uint32_t cmd = ad7280a_crc_write(
        ((uint32_t)1 << 27) |                    // Device 1 지정
        ((uint32_t)AD7280A_REG_CELL_BALANCE << 21) |
        ((uint32_t)balance << 13) |
        (0 << 12)  // This device only (All=0)
    );
    ad7280a_transfer_32bits(cmd);
}
```

## All Devices vs Single Device

명령의 bit 12가 **All Devices** 플래그다.

| Bit 12 | 동작 |
|--------|------|
| 1 | 모든 디바이스에 적용 |
| 0 | 지정된 주소의 디바이스만 |

```c
// 모든 디바이스 변환 시작
cmd = ... | (1 << 12);  // All = 1

// Dev 2만 밸런싱
cmd = ((uint32_t)2 << 27) | ... | (0 << 12);  // All = 0
```

## 삽질 포인트 정리

1. **INC_DEV_ADDR는 한 번만** - 여러 번 보내면 주소 꼬임
2. **LOCK_DEV_ADDR 필수** - 안 하면 주소 불안정
3. **주소 0 = 최하위** - GND 쪽 디바이스
4. **All vs Single** - Bit 12로 구분

다음 글에서는 8개 풀체인(48셀) 구성을 다룬다.

---

## 참고 자료

- [AD7280A Datasheet - Control Register (p.28-29)](https://www.analog.com/media/en/technical-documentation/data-sheets/ad7280a.pdf)
- [AD7280A Datasheet - Addressing (p.16-17)](https://www.analog.com/media/en/technical-documentation/data-sheets/ad7280a.pdf)
