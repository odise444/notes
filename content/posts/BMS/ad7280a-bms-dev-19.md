---
title: "AD7280A BMS 개발기 #19 - 데이터 로깅"
date: 2024-02-02
draft: false
tags: ["AD7280A", "BMS", "STM32", "로깅", "Flash"]
categories: ["BMS 개발"]
series: ["AD7280A BMS 개발"]
summary: "문제 생기면 원인 찾아야 한다. 데이터 로깅 필수."
---

현장에서 문제 생기면 "그때 뭐가 어땠는데?"라고 묻는다.

기록이 없으면 답을 못 한다. 로깅 필수.

---

## 뭘 기록할까

1. **이벤트 로그**: 알람, 상태 변화, 폴트
2. **주기적 데이터**: 전압, 전류, 온도, SOC
3. **통계**: 사이클 수, 최대/최소값

---

## 저장 위치

옵션:
- 내부 Flash: 용량 작음, 쓰기 횟수 제한
- 외부 EEPROM: 느림, 용량 작음
- SD 카드: 용량 큼, 탈착 가능
- 외부 Flash (W25Q): 빠름, 용량 적당

나는 내부 Flash + 외부 SPI Flash 조합.

- 내부 Flash: 설정값, 통계 (자주 안 바뀜)
- 외부 Flash: 이벤트 로그, 주기 데이터

---

## 링 버퍼

Flash는 쓰기 횟수 제한이 있다. 한 곳만 계속 쓰면 수명 줄어든다.

링 버퍼로 분산:

```c
#define LOG_SECTOR_SIZE  4096
#define LOG_SECTOR_COUNT 64    // 256KB

typedef struct {
    uint32_t write_ptr;
    uint32_t read_ptr;
    uint32_t count;
} LogBuffer_t;

void Log_Write(uint8_t *data, uint16_t len) {
    uint32_t addr = LOG_BASE + (g_log.write_ptr % (LOG_SECTOR_SIZE * LOG_SECTOR_COUNT));
    
    Flash_Write(addr, data, len);
    
    g_log.write_ptr += len;
    g_log.count++;
    
    // 섹터 가득 차면 다음 섹터 erase
    if ((g_log.write_ptr % LOG_SECTOR_SIZE) == 0) {
        uint32_t next_sector = g_log.write_ptr / LOG_SECTOR_SIZE;
        Flash_EraseSector(LOG_BASE + next_sector * LOG_SECTOR_SIZE);
    }
}
```

---

## 이벤트 로그 구조

```c
typedef struct __attribute__((packed)) {
    uint32_t timestamp;    // ms since boot
    uint8_t  event_type;   // 이벤트 종류
    uint8_t  data[3];      // 추가 데이터
} EventLog_t;  // 8 bytes

typedef enum {
    EVT_BOOT = 0,
    EVT_FAULT,
    EVT_FAULT_CLEAR,
    EVT_STATE_CHANGE,
    EVT_OV_ALARM,
    EVT_UV_ALARM,
    EVT_OT_ALARM,
    // ...
} EventType_t;
```

이벤트 발생 시:

```c
void Log_Event(EventType_t type, uint8_t d1, uint8_t d2, uint8_t d3) {
    EventLog_t log;
    log.timestamp = HAL_GetTick();
    log.event_type = type;
    log.data[0] = d1;
    log.data[1] = d2;
    log.data[2] = d3;
    
    Log_Write((uint8_t*)&log, sizeof(log));
}

// 사용 예
Log_Event(EVT_OV_ALARM, cell_index, voltage_mv >> 8, voltage_mv & 0xFF);
```

---

## 주기 데이터

1분마다 상태 스냅샷:

```c
typedef struct __attribute__((packed)) {
    uint32_t timestamp;
    uint16_t pack_voltage;
    int16_t  pack_current;
    uint8_t  soc;
    int8_t   max_temp;
    uint16_t max_cell_mv;
    uint16_t min_cell_mv;
} PeriodicLog_t;  // 14 bytes

void Log_Periodic(void) {
    static uint32_t last_log = 0;
    
    if (HAL_GetTick() - last_log > 60000) {  // 1분
        last_log = HAL_GetTick();
        
        PeriodicLog_t log;
        log.timestamp = HAL_GetTick();
        log.pack_voltage = g_bms.pack_voltage_mv / 10;
        log.pack_current = g_bms.pack_current_ma / 10;
        log.soc = g_bms.soc;
        log.max_temp = g_bms.max_temp / 10;
        log.max_cell_mv = g_bms.max_cell_mv;
        log.min_cell_mv = g_bms.min_cell_mv;
        
        Log_Write((uint8_t*)&log, sizeof(log));
    }
}
```

---

## 로그 읽기

UART로 덤프:

```c
void Log_Dump(void) {
    uint32_t addr = LOG_BASE;
    EventLog_t log;
    
    printf("=== Event Log ===\n");
    
    for (int i = 0; i < g_log.count; i++) {
        Flash_Read(addr, (uint8_t*)&log, sizeof(log));
        
        printf("[%lu] Type:%d Data:%02X %02X %02X\n",
            log.timestamp, log.event_type,
            log.data[0], log.data[1], log.data[2]);
        
        addr += sizeof(log);
    }
}
```

---

## 정리

- 링 버퍼로 Flash 수명 관리
- 이벤트 로그: 알람, 상태 변화
- 주기 로그: 1분마다 스냅샷
- UART/CAN으로 덤프

---

다음은 진단 인터페이스.

[#20 - 진단 인터페이스](/posts/bms/ad7280a-bms-dev-20/)
