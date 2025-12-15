---
title: "AD7280A BMS 개발 삽질기 #19 - 데이터 로깅"
date: 2024-12-13
draft: false
tags: ["AD7280A", "BMS", "STM32", "Flash", "로깅", "데이터"]
categories: ["BMS 개발"]
series: ["AD7280A BMS 개발"]
summary: "BMS 데이터를 Flash에 저장하자. 링 버퍼, Wear Leveling, 폴트 이력 관리까지."
---

## 지난 글 요약

[지난 글](/posts/bms/ad7280a-bms-dev-18/)에서 SOC 추정을 다뤘다. 쿨롱 카운팅과 OCV 테이블. 이제 **데이터 로깅**으로 운영 이력을 저장하자.

## 왜 로깅이 필요한가?

```
╔══ 사고 발생! ══════════════════════════╗
║                                        ║
║  "어제 화재가 났는데 원인이 뭐죠?"      ║
║                                        ║
║  로깅 없음: "모릅니다..."               ║
║  로깅 있음: "어제 14:23에 Cell 5가     ║
║            4.2V 도달 후 과열됐습니다"   ║
║                                        ║
╚════════════════════════════════════════╝
```

**필요한 이유**:
- 폴트 원인 분석
- 수명 예측 (사이클 카운트)
- 품질 보증 (Warranty claim)
- 성능 최적화

## 로깅 대상 데이터

### 주기적 데이터 (1분~1시간 주기)

```c
typedef struct __attribute__((packed)) {
    uint32_t timestamp;         // Unix timestamp
    uint16_t cell_voltage[24];  // 셀 전압 (mV)
    int8_t   temperature[4];    // 온도 (°C)
    uint16_t pack_voltage;      // 팩 전압 (0.1V)
    int16_t  pack_current;      // 팩 전류 (0.1A)
    uint8_t  soc;               // SOC (%)
    uint8_t  state;             // BMS 상태
    uint8_t  balancing_mask[4]; // 밸런싱 상태 (24비트)
    uint16_t crc16;             // 데이터 무결성
} log_periodic_t;               // 66 bytes
```

### 이벤트 데이터 (발생 시)

```c
typedef struct __attribute__((packed)) {
    uint32_t timestamp;         // 발생 시간
    uint8_t  event_type;        // 이벤트 종류
    uint8_t  severity;          // 심각도 (0~3)
    uint16_t param1;            // 파라미터 1
    uint16_t param2;            // 파라미터 2
    uint32_t param3;            // 파라미터 3
    uint16_t crc16;
} log_event_t;                  // 16 bytes

// 이벤트 타입
#define EVT_POWER_ON        0x01
#define EVT_POWER_OFF       0x02
#define EVT_CHARGE_START    0x10
#define EVT_CHARGE_END      0x11
#define EVT_DISCHARGE_START 0x12
#define EVT_DISCHARGE_END   0x13
#define EVT_FAULT_OV        0x20
#define EVT_FAULT_UV        0x21
#define EVT_FAULT_OT        0x22
#define EVT_FAULT_COMM      0x23
#define EVT_BALANCE_START   0x30
#define EVT_BALANCE_END     0x31
#define EVT_SOC_CALIBRATE   0x40
```

### 통계 데이터 (누적)

```c
typedef struct __attribute__((packed)) {
    uint32_t magic;             // 0xBMS_STAT
    uint32_t total_charge_ah;   // 총 충전량 (0.1Ah)
    uint32_t total_discharge_ah;// 총 방전량 (0.1Ah)
    uint16_t cycle_count;       // 충방전 사이클
    uint16_t full_charge_count; // 완충 횟수
    uint16_t fault_count[8];    // 폴트 종류별 횟수
    uint32_t operating_hours;   // 총 운영 시간 (초)
    uint32_t charging_hours;    // 충전 시간
    uint32_t discharging_hours; // 방전 시간
    int8_t   max_temp_ever;     // 역대 최고 온도
    int8_t   min_temp_ever;     // 역대 최저 온도
    uint16_t max_cell_mv_ever;  // 역대 최고 전압
    uint16_t min_cell_mv_ever;  // 역대 최저 전압
    uint16_t crc16;
} log_statistics_t;             // 52 bytes
```

## Flash 메모리 레이아웃

STM32F103VE: 512KB Flash

```
┌────────────────────────────────────┐ 0x08080000
│                                    │
│          Application               │
│         (가변 크기)                 │
│                                    │
├────────────────────────────────────┤ 0x08078000 (480KB)
│                                    │
│       Event Log (16KB)             │
│     1024 entries × 16B             │
│                                    │
├────────────────────────────────────┤ 0x08074000 (464KB)
│                                    │
│      Periodic Log (48KB)           │
│       ~745 entries × 66B           │
│     (1시간 주기 = 31일)             │
│                                    │
├────────────────────────────────────┤ 0x08068000 (416KB)
│                                    │
│      Statistics (2KB)              │
│     + Backup copy                  │
│                                    │
├────────────────────────────────────┤ 0x08067800 (414KB)
│                                    │
│      Configuration (2KB)           │
│     + Backup copy                  │
│                                    │
└────────────────────────────────────┘ 0x08067000 (412KB)
```

```c
// flash_layout.h

#define FLASH_CONFIG_ADDR       0x08067000
#define FLASH_CONFIG_SIZE       0x800       // 2KB
#define FLASH_STATS_ADDR        0x08067800
#define FLASH_STATS_SIZE        0x800       // 2KB
#define FLASH_LOG_PERIODIC_ADDR 0x08068000
#define FLASH_LOG_PERIODIC_SIZE 0xC000      // 48KB
#define FLASH_LOG_EVENT_ADDR    0x08074000
#define FLASH_LOG_EVENT_SIZE    0x4000      // 16KB
```

## 링 버퍼 구현

### 헤더 구조

```c
typedef struct __attribute__((packed)) {
    uint32_t magic;         // 0x4C4F4731 ("LOG1")
    uint32_t write_index;   // 다음 쓰기 위치
    uint32_t read_index;    // 다음 읽기 위치 (미사용 가능)
    uint32_t entry_count;   // 총 엔트리 수
    uint32_t entry_size;    // 엔트리 크기
    uint32_t max_entries;   // 최대 엔트리 수
    uint32_t wrap_count;    // 랩어라운드 횟수
    uint16_t crc16;
} log_header_t;
```

### 초기화

```c
// log_flash.c

#define LOG_MAGIC           0x4C4F4731

static log_header_t g_periodic_header;
static log_header_t g_event_header;

void Log_Init(void) {
    // 주기적 로그 헤더 로드
    memcpy(&g_periodic_header, (void *)FLASH_LOG_PERIODIC_ADDR, sizeof(log_header_t));
    
    if (g_periodic_header.magic != LOG_MAGIC) {
        // 초기화 필요
        Log_Format(FLASH_LOG_PERIODIC_ADDR, FLASH_LOG_PERIODIC_SIZE, 
                   sizeof(log_periodic_t));
    }
    
    // 이벤트 로그 헤더 로드
    memcpy(&g_event_header, (void *)FLASH_LOG_EVENT_ADDR, sizeof(log_header_t));
    
    if (g_event_header.magic != LOG_MAGIC) {
        Log_Format(FLASH_LOG_EVENT_ADDR, FLASH_LOG_EVENT_SIZE,
                   sizeof(log_event_t));
    }
}

void Log_Format(uint32_t base_addr, uint32_t size, uint32_t entry_size) {
    log_header_t header = {0};
    
    header.magic = LOG_MAGIC;
    header.write_index = 0;
    header.read_index = 0;
    header.entry_count = 0;
    header.entry_size = entry_size;
    header.max_entries = (size - sizeof(log_header_t)) / entry_size;
    header.wrap_count = 0;
    header.crc16 = CRC16_Calculate((uint8_t *)&header, sizeof(header) - 2);
    
    // 헤더 영역만 Erase
    Flash_ErasePage(base_addr);
    Flash_ProgramBuffer(base_addr, (uint8_t *)&header, sizeof(header));
}
```

### 쓰기

```c
HAL_StatusTypeDef Log_WriteEntry(log_header_t *header, uint32_t base_addr,
                                  void *entry, uint32_t entry_size) {
    // 엔트리 주소 계산
    uint32_t data_start = base_addr + sizeof(log_header_t);
    uint32_t entry_addr = data_start + (header->write_index * entry_size);
    
    // 페이지 경계 체크 (STM32F103: 2KB 페이지)
    uint32_t page_start = entry_addr & ~0x7FF;
    uint32_t page_end = (entry_addr + entry_size - 1) & ~0x7FF;
    
    Flash_Unlock();
    
    // 새 페이지 시작이면 Erase
    if ((entry_addr & 0x7FF) == 0 || page_start != page_end) {
        Flash_ErasePage(page_start);
        if (page_start != page_end) {
            Flash_ErasePage(page_end);
        }
    }
    
    // 엔트리 쓰기
    Flash_ProgramBuffer(entry_addr, (uint8_t *)entry, entry_size);
    
    // 인덱스 업데이트
    header->write_index++;
    if (header->write_index >= header->max_entries) {
        header->write_index = 0;
        header->wrap_count++;
    }
    header->entry_count++;
    header->crc16 = CRC16_Calculate((uint8_t *)header, sizeof(*header) - 2);
    
    // 헤더 업데이트 (첫 페이지)
    Flash_ErasePage(base_addr);
    Flash_ProgramBuffer(base_addr, (uint8_t *)header, sizeof(*header));
    
    Flash_Lock();
    
    return HAL_OK;
}
```

### 읽기

```c
HAL_StatusTypeDef Log_ReadEntry(log_header_t *header, uint32_t base_addr,
                                 uint32_t index, void *entry, uint32_t entry_size) {
    if (index >= header->entry_count) {
        return HAL_ERROR;
    }
    
    // 실제 인덱스 계산 (링 버퍼)
    uint32_t actual_index;
    if (header->wrap_count > 0) {
        // 랩어라운드 발생: write_index가 가장 오래된 데이터
        actual_index = (header->write_index + index) % header->max_entries;
    } else {
        actual_index = index;
    }
    
    uint32_t data_start = base_addr + sizeof(log_header_t);
    uint32_t entry_addr = data_start + (actual_index * entry_size);
    
    memcpy(entry, (void *)entry_addr, entry_size);
    
    return HAL_OK;
}
```

## 주기적 로깅

```c
// 1분 주기 로깅 타이머
void Log_PeriodicTask(void) {
    static uint32_t last_log_time = 0;
    uint32_t now = HAL_GetTick();
    
    // 1분 주기
    if (now - last_log_time < 60000) {
        return;
    }
    last_log_time = now;
    
    // 로그 엔트리 생성
    log_periodic_t entry = {0};
    
    entry.timestamp = RTC_GetUnixTime();
    
    // 셀 전압 복사
    for (int i = 0; i < 24; i++) {
        entry.cell_voltage[i] = g_bms.cell_voltage[i];
    }
    
    // 온도 복사
    for (int i = 0; i < 4; i++) {
        entry.temperature[i] = g_bms.temperature[i];
    }
    
    entry.pack_voltage = g_bms.pack_voltage_mv / 100;  // 0.1V 단위
    entry.pack_current = g_bms.pack_current_ma / 100;  // 0.1A 단위
    entry.soc = (uint8_t)g_bms.soc;
    entry.state = g_bms.state;
    
    // 밸런싱 상태 (24비트 → 3바이트)
    entry.balancing_mask[0] = g_bms.balance_mask & 0xFF;
    entry.balancing_mask[1] = (g_bms.balance_mask >> 8) & 0xFF;
    entry.balancing_mask[2] = (g_bms.balance_mask >> 16) & 0xFF;
    
    // CRC 계산
    entry.crc16 = CRC16_Calculate((uint8_t *)&entry, sizeof(entry) - 2);
    
    // Flash에 저장
    Log_WriteEntry(&g_periodic_header, FLASH_LOG_PERIODIC_ADDR,
                   &entry, sizeof(entry));
}
```

## 이벤트 로깅

```c
void Log_Event(uint8_t type, uint8_t severity, 
               uint16_t p1, uint16_t p2, uint32_t p3) {
    log_event_t entry = {0};
    
    entry.timestamp = RTC_GetUnixTime();
    entry.event_type = type;
    entry.severity = severity;
    entry.param1 = p1;
    entry.param2 = p2;
    entry.param3 = p3;
    entry.crc16 = CRC16_Calculate((uint8_t *)&entry, sizeof(entry) - 2);
    
    Log_WriteEntry(&g_event_header, FLASH_LOG_EVENT_ADDR,
                   &entry, sizeof(entry));
}

// 사용 예시
void BMS_OnFaultOccurred(uint8_t fault_type, uint8_t cell, uint16_t value) {
    Log_Event(EVT_FAULT_OV + fault_type, 
              SEVERITY_ERROR,
              cell,           // 셀 번호
              value,          // 측정값
              g_bms.state);   // 현재 상태
}

void BMS_OnChargeStart(void) {
    Log_Event(EVT_CHARGE_START, 
              SEVERITY_INFO,
              g_bms.soc,              // 시작 SOC
              g_bms.pack_voltage_mv,  // 팩 전압
              0);
}
```

## 통계 관리

```c
// 통계 업데이트 (100ms 주기)
void Log_UpdateStatistics(void) {
    static uint32_t last_update = 0;
    static float accumulated_charge = 0;
    static float accumulated_discharge = 0;
    
    uint32_t now = HAL_GetTick();
    float dt_hours = (now - last_update) / 3600000.0f;
    last_update = now;
    
    // 운영 시간
    g_stats.operating_hours++;
    
    // 충방전량 누적
    float current_a = g_bms.pack_current_ma / 1000.0f;
    
    if (current_a < 0) {  // 충전
        accumulated_charge += (-current_a) * dt_hours;
        g_stats.charging_hours++;
    } else if (current_a > 0) {  // 방전
        accumulated_discharge += current_a * dt_hours;
        g_stats.discharging_hours++;
    }
    
    // Ah 단위로 저장 (0.1Ah 단위)
    g_stats.total_charge_ah = (uint32_t)(accumulated_charge * 10);
    g_stats.total_discharge_ah = (uint32_t)(accumulated_discharge * 10);
    
    // 사이클 카운트 (80% DOD 기준)
    // 100Ah 팩이면 80Ah 방전 = 1 사이클
    g_stats.cycle_count = g_stats.total_discharge_ah / (g_bms.capacity_ah * 8);
    
    // 최대/최소 기록
    int8_t max_temp = BMS_GetMaxTemperature();
    int8_t min_temp = BMS_GetMinTemperature();
    uint16_t max_cell = BMS_GetMaxCellVoltage();
    uint16_t min_cell = BMS_GetMinCellVoltage();
    
    if (max_temp > g_stats.max_temp_ever) g_stats.max_temp_ever = max_temp;
    if (min_temp < g_stats.min_temp_ever) g_stats.min_temp_ever = min_temp;
    if (max_cell > g_stats.max_cell_mv_ever) g_stats.max_cell_mv_ever = max_cell;
    if (min_cell < g_stats.min_cell_mv_ever) g_stats.min_cell_mv_ever = min_cell;
}

// 통계 저장 (1시간 주기 또는 종료 시)
void Log_SaveStatistics(void) {
    g_stats.magic = 0x424D5353;  // "BMSS"
    g_stats.crc16 = CRC16_Calculate((uint8_t *)&g_stats, sizeof(g_stats) - 2);
    
    Flash_Unlock();
    Flash_ErasePage(FLASH_STATS_ADDR);
    Flash_ProgramBuffer(FLASH_STATS_ADDR, (uint8_t *)&g_stats, sizeof(g_stats));
    
    // 백업 복사본
    Flash_ErasePage(FLASH_STATS_ADDR + 0x800);
    Flash_ProgramBuffer(FLASH_STATS_ADDR + 0x800, (uint8_t *)&g_stats, sizeof(g_stats));
    Flash_Lock();
}
```

## Wear Leveling 고려

Flash는 쓰기 수명이 있다 (STM32: ~10,000회):

```c
// 쓰기 횟수 분산을 위한 간단한 Wear Leveling
typedef struct {
    uint8_t active_page;    // 현재 활성 페이지 (0~3)
    uint32_t write_counts[4]; // 페이지별 쓰기 횟수
} wear_level_t;

uint32_t GetNextWritePage(wear_level_t *wl) {
    // 가장 적게 쓴 페이지 선택
    uint32_t min_count = wl->write_counts[0];
    uint8_t min_page = 0;
    
    for (int i = 1; i < 4; i++) {
        if (wl->write_counts[i] < min_count) {
            min_count = wl->write_counts[i];
            min_page = i;
        }
    }
    
    wl->active_page = min_page;
    wl->write_counts[min_page]++;
    
    return FLASH_LOG_PERIODIC_ADDR + (min_page * 0x3000);
}
```

## 삽질: 페이지 경계

엔트리가 페이지 경계를 넘으면 문제:

```c
// 잘못된 코드
Flash_ErasePage(page_start);
// 엔트리가 두 페이지에 걸침 → 다음 페이지 데이터 손실!

// 올바른 코드
if (page_start != page_end) {
    Flash_ErasePage(page_start);
    Flash_ErasePage(page_end);  // 다음 페이지도 Erase
}
```

## 삽질: 전원 차단 중 쓰기

쓰기 중 전원 차단 → 데이터 손상:

```c
// 해결책: Write-Ahead Logging
void Log_SafeWrite(void *entry, uint32_t size) {
    // 1. 임시 영역에 먼저 쓰기
    Flash_ProgramBuffer(TEMP_ADDR, entry, size);
    
    // 2. Valid 마크
    Flash_ProgramHalfWord(TEMP_ADDR + size, 0xAA55);
    
    // 3. 실제 위치에 복사
    // (부팅 시 TEMP_ADDR에 valid 데이터 있으면 복구)
}
```

## CAN으로 로그 읽기

```c
// 진단 명령으로 로그 조회
void Log_HandleDiagRequest(uint8_t *data) {
    uint8_t cmd = data[0];
    
    switch (cmd) {
    case 0x01:  // 최근 이벤트 조회
        {
            log_event_t event;
            uint16_t index = (data[1] << 8) | data[2];
            
            if (Log_ReadEntry(&g_event_header, FLASH_LOG_EVENT_ADDR,
                              index, &event, sizeof(event)) == HAL_OK) {
                // CAN으로 응답 (여러 메시지로 분할)
                BMS_CAN_SendLogEntry(&event);
            }
        }
        break;
        
    case 0x02:  // 통계 조회
        BMS_CAN_SendStatistics(&g_stats);
        break;
        
    case 0x03:  // 로그 클리어
        Log_Format(FLASH_LOG_EVENT_ADDR, FLASH_LOG_EVENT_SIZE, sizeof(log_event_t));
        break;
    }
}
```

## 정리

| 항목 | 크기 | 주기 | 저장량 |
|------|------|------|--------|
| 주기적 데이터 | 66B | 1분 | 31일 |
| 이벤트 데이터 | 16B | 이벤트 | 1024개 |
| 통계 | 52B | 1시간 | 영구 |

**핵심 포인트**:
- 링 버퍼로 순환 저장
- CRC로 무결성 검증
- 백업 복사본 유지
- Wear Leveling 고려

**다음 글에서**: 진단 인터페이스 - CLI와 디버그 명령.

---

## 시리즈 네비게이션

**Part 6: 통신 & 진단편**
- [#17 - CAN 통신 프로토콜 설계](/posts/bms/ad7280a-bms-dev-17/)
- [#18 - SOC 추정 기초](/posts/bms/ad7280a-bms-dev-18/)
- **#19 - 데이터 로깅** ← 현재 글
- #20 - 진단 인터페이스

---

## 참고 자료

- [STM32 Flash Programming](https://www.st.com/resource/en/programming_manual/pm0075.pdf)
- [Flash Wear Leveling](https://www.embedded.com/flash-memory-wear-leveling/)
- [Ring Buffer Implementation](https://embedjournal.com/implementing-circular-buffer-embedded-c/)
