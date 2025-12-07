---
title: "AD7280A BMS 개발 삽질기 #9 - 8개 풀체인 구성하기 (48셀)"
date: 2024-12-08
draft: false
tags: ["BMS", "AD7280A", "임베디드", "배터리", "삽질", "데이지체인"]
categories: ["BMS 개발"]
summary: "48셀 모니터링을 위해 AD7280A 8개를 연결했다. 예상 못한 문제들이 터졌다."
---

## 지난 글 요약

[지난 글](/posts/bms/ad7280a-bms-dev-08/)에서 주소 할당 메커니즘을 이해했다. 이제 진짜 대규모로 가보자. 48셀 = AD7280A 8개!

## 48셀 시스템 구성

48셀 리튬이온 배터리 팩:
- 공칭 전압: 48 × 3.7V = **177.6V**
- 만충 전압: 48 × 4.2V = **201.6V**

```
┌─────────────────────────────────────┐
│ Cell 48 ─── Dev7-VIN6 (201.6V)      │
│ Cell 47                              │
│ ...                                  │
│ Cell 43 ─── Dev7-VIN0 = Dev6-VIN6   │
├─────────────────────────────────────┤
│ Cell 42 ─── Dev6-VIN6               │
│ ...                                  │
│ Cell 37 ─── Dev6-VIN0 = Dev5-VIN6   │
├─────────────────────────────────────┤
│ ...                                  │
├─────────────────────────────────────┤
│ Cell 6  ─── Dev0-VIN6               │
│ ...                                  │
│ Cell 1  ─── Dev0-VIN0 (GND)         │
└─────────────────────────────────────┘
```

## 문제 1: SPI 클럭 지연

8개 체인이면 SPI 신호가 **7번 버퍼링**된다. 각 디바이스에서 약간의 지연이 누적된다.

```
MCU → Dev0 → Dev1 → Dev2 → ... → Dev7 → MCU
      ↑      ↑      ↑            ↑
     지연   지연   지연          지연
```

### 증상

1MHz SPI 클럭에서 간헐적 CRC 에러 발생.

### 해결

**SPI 클럭을 낮춘다.** 데이터시트 최대 1MHz지만, 8개 체인에서는 500kHz가 안정적이었다.

```c
// 8개 체인 구성
#define SPI_FREQ_8DEV    500000  // 500kHz

spi_init(SPI_PORT, SPI_FREQ_8DEV);
```

## 문제 2: 변환 시간 증가

모든 디바이스가 동시에 변환하지만, 데이터 읽기는 순차적이다.

```
변환 시작 (CNVST 펄스)
    ↓
모든 Dev 동시 변환 (약 1.6μs × 6채널)
    ↓
데이터 읽기: Dev7 → Dev6 → ... → Dev0
    ↓
총 48채널 × 32bit = 1536bit 전송
```

500kHz에서 1536bit 전송 시간:
```
1536 / 500000 = 3.07ms
```

변환 + 전송 = 약 **4ms per cycle**. 초당 250Hz 샘플링 가능.

## 문제 3: 전압 레벨 점프

데이지체인 연결점에서 전압이 급격히 바뀐다.

```
Dev0-VIN6 = 25.2V (6셀 × 4.2V)
Dev1-VIN0 = 25.2V (같은 노드)
Dev1-VIN6 = 50.4V (12셀 × 4.2V)
```

**Dev0과 Dev1 사이 전압차: 25.2V**

이 연결이 끊어지거나 저항이 들어가면 측정값이 이상해진다.

### 점검 포인트

```c
void check_daisy_chain_connection(void) {
    for (int dev = 0; dev < 7; dev++) {
        // Dev N의 Cell 6 전압
        float cell6 = read_cell(dev, 6);
        // Dev N+1의 Cell 1 전압  
        float cell1_next = read_cell(dev + 1, 1);
        
        // 두 셀의 합 ≈ VIN6-VIN0 차이여야 함
        float expected = cell6 + cell1_next;
        float actual = read_vin6(dev) - read_vin0(dev + 1);
        
        if (abs(expected - actual) > 100) {  // 100mV 오차 허용
            printf("Chain break between Dev%d-Dev%d!\n", dev, dev+1);
        }
    }
}
```

## 문제 4: Alert 체인

ALERT 핀도 데이지체인된다. 어느 디바이스든 알람 발생 시 MCU까지 전달된다.

```
Dev7-ALERT → Dev6-ALERT → ... → Dev0-ALERT → MCU
```

### 어떤 디바이스가 알람인지?

ALERT 핀만으로는 **어떤 디바이스**인지 알 수 없다. 각 디바이스의 Alert Register를 읽어야 한다.

```c
void find_alert_source(uint8_t num_devices) {
    for (uint8_t dev = 0; dev < num_devices; dev++) {
        uint8_t alert_status = ad7280a_read_reg(dev, AD7280A_REG_ALERT);
        
        if (alert_status & 0xF0) {  // 상위 4비트 = 알람 상태
            printf("Alert from Dev%d: 0x%02X\n", dev, alert_status);
            
            // 어떤 채널이 문제인지 확인
            if (alert_status & (1 << 7)) printf("  - Cell OV\n");
            if (alert_status & (1 << 6)) printf("  - Cell UV\n");
            if (alert_status & (1 << 5)) printf("  - AUX OV\n");
            if (alert_status & (1 << 4)) printf("  - AUX UV\n");
        }
    }
}
```

## 문제 5: 소프트웨어 리셋 전파

소프트웨어 리셋 명령도 체인을 따라 전파된다.

```c
// 모든 디바이스 리셋
void ad7280a_reset_all(void) {
    uint8_t ctrl_lb = AD7280A_CTRL_LB_SWRST;  // D7 = 1
    
    uint32_t cmd = ad7280a_crc_write(
        ((uint32_t)AD7280A_REG_CONTROL_LB << 21) |
        ((uint32_t)ctrl_lb << 13) |
        (1 << 12)  // All devices
    );
    ad7280a_transfer_32bits(cmd);
    
    sleep_ms(10);  // 리셋 완료 대기
}
```

**주의**: 리셋 후 **주소 재할당 필수!**

## 초기화 시퀀스 (8개용)

```c
#define NUM_DEVICES 8

void ad7280a_init_48cell(void) {
    // 1. 하드웨어 리셋
    gpio_put(PIN_PD, 0);
    sleep_ms(10);
    gpio_put(PIN_PD, 1);
    sleep_ms(10);
    
    // 2. SPI 클럭 설정 (8개용 낮춤)
    spi_set_baudrate(SPI_PORT, 500000);
    
    // 3. 소프트웨어 리셋
    ad7280a_reset_all();
    sleep_ms(10);
    
    // 4. 주소 할당
    ad7280a_assign_addresses(NUM_DEVICES);
    
    // 5. Control LB 설정
    uint8_t ctrl_lb = AD7280A_CTRL_LB_MUST_SET |
                      AD7280A_ACQ_TIME_1600NS |
                      AD7280A_CTRL_LB_LOCK_DEV_ADDR |
                      AD7280A_CTRL_LB_DAISY_CHAIN_RB_EN;
    ad7280a_write_all(AD7280A_REG_CONTROL_LB, ctrl_lb);
    
    // 6. 주소 확인
    for (uint8_t dev = 0; dev < NUM_DEVICES; dev++) {
        uint8_t addr = ad7280a_verify_address(dev);
        printf("Dev%d: addr=%d %s\n", dev, addr, 
               (dev == addr) ? "OK" : "FAIL");
    }
    
    // 7. Alert 임계값 설정
    ad7280a_set_thresholds_all(
        3000,   // Undervoltage: 3.0V
        4250,   // Overvoltage: 4.25V
        0,      // AUX min
        5000    // AUX max
    );
}
```

## 전체 셀 읽기 (48셀)

```c
void ad7280a_read_48cells(float *voltages) {
    // 변환 시작
    gpio_put(PIN_CNVST, 0);
    sleep_us(1);
    gpio_put(PIN_CNVST, 1);
    sleep_us(100);  // 8개 디바이스 변환 완료 대기
    
    // Read 레지스터 설정
    ad7280a_write_all(AD7280A_REG_READ, 0x00);  // Cell 1부터
    
    // 48채널 읽기
    for (int i = 0; i < NUM_DEVICES * 8; i++) {
        uint32_t rx = ad7280a_transfer_32bits(AD7280A_NOP);
        
        uint8_t dev = (rx >> 27) & 0x1F;
        uint8_t reg = (rx >> 23) & 0x0F;
        uint16_t data = (rx >> 11) & 0x0FFF;
        
        if (reg <= 5) {  // Cell voltage
            int cell_idx = dev * 6 + reg;
            voltages[cell_idx] = data * 0.976f + 1000.0f;
        }
    }
}
```

## 성능 측정 결과

| 항목 | 2개 체인 | 8개 체인 |
|------|----------|----------|
| SPI 클럭 | 1MHz | 500kHz |
| 1회 샘플링 | ~1ms | ~4ms |
| 최대 샘플링 레이트 | 500Hz | 250Hz |
| CRC 에러율 | 0% | 0% (클럭 낮춘 후) |

## 삽질 포인트 정리

1. **SPI 클럭 낮추기** - 8개 체인에서 1MHz는 불안정
2. **변환 완료 대기** - 디바이스 수에 비례해서 늘리기
3. **체인 연결 확인** - VIN6-VIN0 노드 공유 필수
4. **Alert 소스 추적** - 각 디바이스 레지스터 확인
5. **리셋 후 재초기화** - 주소 할당 다시!

이제 진짜 48셀 BMS의 기초가 완성됐다!

---

## 참고 자료

- [AD7280A Datasheet - Daisy-Chain (p.16-19)](https://www.analog.com/media/en/technical-documentation/data-sheets/ad7280a.pdf)
- [CN0235 - 완전 절연 배터리 모니터링](https://www.analog.com/en/resources/reference-designs/circuits-from-the-lab/cn0235.html)
