---
title: "AD7280A BMS 개발기 #19 - 데이터 로깅"
date: 2024-12-13
draft: false
tags: ["AD7280A", "BMS", "STM32", "Flash", "로깅"]
categories: ["BMS 개발"]
series: ["AD7280A BMS 개발"]
summary: "나중에 문제 생기면 원인을 찾아야 한다. Flash에 이력을 저장하자."
---

"어제 배터리가 갑자기 꺼졌는데 왜 그런 거죠?"

로깅이 없으면 답을 못 한다. 로깅이 있으면 "어제 14:23에 Cell 5가 2.4V로 떨어졌습니다"라고 답할 수 있다.

---

STM32 내부 Flash에 저장한다. 외부 EEPROM을 쓸 수도 있는데, 내부 Flash가 간단하다.

링 버퍼 구조로 만든다. 가득 차면 오래된 데이터부터 덮어쓴다.

```c
#define LOG_SECTOR_START    0x08060000  // Flash 섹터 주소
#define LOG_SECTOR_SIZE     0x20000     // 128KB
#define LOG_ENTRY_SIZE      32

typedef struct {
    uint32_t timestamp;
    uint8_t  event_type;
    uint8_t  severity;
    uint16_t cell_voltages[6];  // 대표값만
    int8_t   temperatures[2];
    int16_t  current;
    uint8_t  soc;
    uint8_t  reserved[5];
} LogEntry_t;
```

---

Flash 쓰기는 좀 귀찮다. 쓰기 전에 지워야 하고, 섹터 단위로만 지울 수 있다.

```c
void Log_WriteEntry(LogEntry_t *entry) {
    // 다음 쓸 위치 계산
    uint32_t addr = LOG_SECTOR_START + (log_index * LOG_ENTRY_SIZE);
    
    // 섹터 경계 넘으면 지우기
    if (addr >= LOG_SECTOR_START + LOG_SECTOR_SIZE) {
        HAL_FLASH_Unlock();
        FLASH_Erase_Sector(FLASH_SECTOR_7, VOLTAGE_RANGE_3);
        HAL_FLASH_Lock();
        log_index = 0;
        addr = LOG_SECTOR_START;
    }
    
    // 쓰기
    HAL_FLASH_Unlock();
    uint32_t *data = (uint32_t *)entry;
    for (int i = 0; i < LOG_ENTRY_SIZE / 4; i++) {
        HAL_FLASH_Program(FLASH_TYPEPROGRAM_WORD, addr + i*4, data[i]);
    }
    HAL_FLASH_Lock();
    
    log_index++;
}
```

---

이벤트 종류:

```c
#define EVT_POWER_ON        0x01
#define EVT_POWER_OFF       0x02
#define EVT_FAULT_OV        0x10
#define EVT_FAULT_UV        0x11
#define EVT_FAULT_OT        0x12
#define EVT_CHARGE_START    0x20
#define EVT_CHARGE_END      0x21
#define EVT_PERIODIC        0x80  // 주기적 기록 (1분마다)
```

---

Wear Leveling도 고려해야 한다. Flash는 쓰기 횟수 제한이 있다 (보통 10만 회). 같은 위치에 계속 쓰면 빨리 닳는다.

링 버퍼 구조면 자연스럽게 분산되긴 하는데, 더 신경 쓰려면 별도 처리가 필요하다.

---

다음 글에서 진단 인터페이스를 다룬다.

[#20 - 진단 인터페이스](/posts/bms/ad7280a-bms-dev-20/)
