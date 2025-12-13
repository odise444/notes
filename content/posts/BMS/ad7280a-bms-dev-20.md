---
title: "AD7280A BMS 개발 삽질기 #20 - 진단 인터페이스"
date: 2024-12-13
draft: false
tags: ["AD7280A", "BMS", "STM32", "진단", "CLI", "디버그"]
categories: ["BMS 개발"]
series: ["AD7280A BMS 개발"]
summary: "개발과 현장 디버깅을 위한 CLI 명령어 체계. UART와 CAN 진단 인터페이스 구현."
---

## 지난 글 요약

[지난 글](/posts/bms/ad7280a-bms-dev-19/)에서 데이터 로깅을 구현했다. Flash 저장, 링 버퍼, 통계 관리. 이제 **진단 인터페이스**로 디버깅을 편하게 하자.

## 왜 진단 인터페이스가 필요한가?

```
┌─ 현장에서... ────────────────────────────┐
│                                          │
│  "BMS가 이상해요. 뭐가 문제죠?"           │
│                                          │
│  진단 없음: ST-Link 연결... 소스 분석...  │
│            → 반나절 소요                  │
│                                          │
│  진단 있음: "status" 명령 입력            │
│            → 10초 만에 원인 파악          │
│                                          │
└──────────────────────────────────────────┘
```

**용도**:
- 개발 중 디버깅
- 생산 라인 테스트
- 현장 트러블슈팅
- 파라미터 조정

## 진단 채널

### UART (개발/생산용)

```
┌─────────┐    USB-UART    ┌────────────┐
│   PC    │◄──────────────►│    BMS     │
│ (터미널) │    115200bps   │  (UART1)   │
└─────────┘                └────────────┘
```

### CAN (현장용)

```
┌─────────┐      CAN       ┌────────────┐
│  진단기  │◄──────────────►│    BMS     │
│ (PCAN)  │    500kbps     │            │
└─────────┘                └────────────┘
```

## UART CLI 구현

### 명령어 파서

```c
// cli.c

#define CLI_MAX_CMD_LEN     64
#define CLI_MAX_ARGS        8
#define CLI_PROMPT          "BMS> "

typedef struct {
    const char *name;
    const char *help;
    void (*handler)(int argc, char *argv[]);
} cli_command_t;

static char g_cmd_buffer[CLI_MAX_CMD_LEN];
static uint8_t g_cmd_index = 0;

// 명령어 테이블
static const cli_command_t g_commands[] = {
    {"help",     "Show all commands",           cmd_help},
    {"status",   "Show BMS status",             cmd_status},
    {"cell",     "Show cell voltages",          cmd_cell},
    {"temp",     "Show temperatures",           cmd_temp},
    {"balance",  "Control balancing",           cmd_balance},
    {"fet",      "Control FETs",                cmd_fet},
    {"fault",    "Show/clear faults",           cmd_fault},
    {"log",      "Log operations",              cmd_log},
    {"stats",    "Show statistics",             cmd_stats},
    {"config",   "Configuration",               cmd_config},
    {"test",     "Self-test",                   cmd_test},
    {"reset",    "System reset",                cmd_reset},
    {"debug",    "Debug commands",              cmd_debug},
    {NULL, NULL, NULL}
};
```

### 명령어 처리

```c
void CLI_ProcessChar(char c) {
    if (c == '\r' || c == '\n') {
        // 명령어 실행
        g_cmd_buffer[g_cmd_index] = '\0';
        if (g_cmd_index > 0) {
            CLI_ExecuteCommand(g_cmd_buffer);
        }
        g_cmd_index = 0;
        printf(CLI_PROMPT);
    }
    else if (c == '\b' || c == 0x7F) {
        // 백스페이스
        if (g_cmd_index > 0) {
            g_cmd_index--;
            printf("\b \b");
        }
    }
    else if (g_cmd_index < CLI_MAX_CMD_LEN - 1) {
        g_cmd_buffer[g_cmd_index++] = c;
        printf("%c", c);  // 에코
    }
}

void CLI_ExecuteCommand(const char *cmd) {
    char *argv[CLI_MAX_ARGS];
    int argc = 0;
    char *token;
    char cmd_copy[CLI_MAX_CMD_LEN];
    
    strcpy(cmd_copy, cmd);
    
    // 토큰 분리
    token = strtok(cmd_copy, " ");
    while (token && argc < CLI_MAX_ARGS) {
        argv[argc++] = token;
        token = strtok(NULL, " ");
    }
    
    if (argc == 0) return;
    
    // 명령어 검색
    for (int i = 0; g_commands[i].name != NULL; i++) {
        if (strcmp(argv[0], g_commands[i].name) == 0) {
            g_commands[i].handler(argc, argv);
            return;
        }
    }
    
    printf("Unknown command: %s\r\n", argv[0]);
    printf("Type 'help' for available commands\r\n");
}
```

## 명령어 구현

### help - 도움말

```c
void cmd_help(int argc, char *argv[]) {
    printf("\r\n=== BMS CLI Commands ===\r\n");
    
    for (int i = 0; g_commands[i].name != NULL; i++) {
        printf("  %-10s - %s\r\n", 
               g_commands[i].name, 
               g_commands[i].help);
    }
    printf("\r\n");
}
```

### status - 상태 조회

```c
void cmd_status(int argc, char *argv[]) {
    printf("\r\n=== BMS Status ===\r\n");
    printf("State:      %s\r\n", BMS_StateToString(g_bms.state));
    printf("SOC:        %d%%\r\n", (int)g_bms.soc);
    printf("SOH:        %d%%\r\n", (int)g_bms.soh);
    printf("Pack V:     %.1fV\r\n", g_bms.pack_voltage_mv / 1000.0f);
    printf("Pack I:     %.2fA\r\n", g_bms.pack_current_ma / 1000.0f);
    printf("Max Cell:   %dmV (Cell %d)\r\n", 
           BMS_GetMaxCellVoltage(), BMS_GetMaxCellIndex() + 1);
    printf("Min Cell:   %dmV (Cell %d)\r\n",
           BMS_GetMinCellVoltage(), BMS_GetMinCellIndex() + 1);
    printf("Delta:      %dmV\r\n", 
           BMS_GetMaxCellVoltage() - BMS_GetMinCellVoltage());
    printf("Max Temp:   %d°C\r\n", BMS_GetMaxTemperature());
    printf("Min Temp:   %d°C\r\n", BMS_GetMinTemperature());
    printf("Charge:     %s\r\n", g_bms.charge_allowed ? "Allowed" : "Blocked");
    printf("Discharge:  %s\r\n", g_bms.discharge_allowed ? "Allowed" : "Blocked");
    printf("Balancing:  %s\r\n", g_bms.balancing_active ? "Active" : "Inactive");
    printf("Fault:      %s\r\n", g_bms.fault ? "YES" : "No");
    printf("\r\n");
}
```

### cell - 셀 전압

```c
void cmd_cell(int argc, char *argv[]) {
    printf("\r\n=== Cell Voltages ===\r\n");
    
    for (int dev = 0; dev < 4; dev++) {
        printf("Device %d:\r\n", dev);
        for (int cell = 0; cell < 6; cell++) {
            int idx = dev * 6 + cell;
            uint16_t mv = g_bms.cell_voltage[idx];
            char bal = (g_bms.balance_mask & (1 << idx)) ? '*' : ' ';
            
            printf("  Cell %2d: %4dmV %c", idx + 1, mv, bal);
            
            // 상태 표시
            if (mv >= 3600) printf(" [HIGH]");
            else if (mv <= 2800) printf(" [LOW]");
            
            printf("\r\n");
        }
    }
    
    printf("\r\n* = Balancing active\r\n\r\n");
}
```

### balance - 밸런싱 제어

```c
void cmd_balance(int argc, char *argv[]) {
    if (argc < 2) {
        printf("Usage: balance <start|stop|status|set>\r\n");
        printf("  balance start        - Start auto balancing\r\n");
        printf("  balance stop         - Stop balancing\r\n");
        printf("  balance status       - Show balancing status\r\n");
        printf("  balance set <mask>   - Manual set (hex)\r\n");
        return;
    }
    
    if (strcmp(argv[1], "start") == 0) {
        g_bms.balance_enabled = true;
        printf("Balancing enabled\r\n");
    }
    else if (strcmp(argv[1], "stop") == 0) {
        g_bms.balance_enabled = false;
        ad7280a_stop_balancing(&g_ad7280a);
        printf("Balancing disabled\r\n");
    }
    else if (strcmp(argv[1], "status") == 0) {
        printf("Balancing: %s\r\n", g_bms.balance_enabled ? "Enabled" : "Disabled");
        printf("Active:    %s\r\n", g_bms.balancing_active ? "Yes" : "No");
        printf("Mask:      0x%06X\r\n", g_bms.balance_mask);
        
        printf("Active cells: ");
        for (int i = 0; i < 24; i++) {
            if (g_bms.balance_mask & (1 << i)) {
                printf("%d ", i + 1);
            }
        }
        printf("\r\n");
    }
    else if (strcmp(argv[1], "set") == 0 && argc >= 3) {
        uint32_t mask = strtoul(argv[2], NULL, 16);
        ad7280a_set_balance_manual(&g_ad7280a, mask);
        printf("Balance mask set to 0x%06X\r\n", mask);
    }
}
```

### fault - 폴트 관리

```c
void cmd_fault(int argc, char *argv[]) {
    if (argc < 2) {
        // 현재 폴트 표시
        printf("\r\n=== Fault Status ===\r\n");
        printf("Current Fault: %s\r\n", g_bms.fault ? "YES" : "No");
        
        if (g_bms.fault) {
            printf("Fault Code:    0x%04X\r\n", g_bms.fault_code);
            printf("Fault Cell:    %d\r\n", g_bms.fault_cell);
            printf("Fault Value:   %d\r\n", g_bms.fault_value);
        }
        
        printf("\r\nFault History:\r\n");
        for (int i = 0; i < 8; i++) {
            if (g_stats.fault_count[i] > 0) {
                printf("  %s: %d times\r\n", 
                       Fault_TypeToString(i), 
                       g_stats.fault_count[i]);
            }
        }
        return;
    }
    
    if (strcmp(argv[1], "clear") == 0) {
        BMS_ClearFault();
        printf("Faults cleared\r\n");
    }
    else if (strcmp(argv[1], "test") == 0 && argc >= 3) {
        // 폴트 테스트 (개발용)
        uint8_t type = atoi(argv[2]);
        BMS_TriggerTestFault(type);
        printf("Test fault triggered: %d\r\n", type);
    }
}
```

### log - 로그 조회

```c
void cmd_log(int argc, char *argv[]) {
    if (argc < 2) {
        printf("Usage: log <event|periodic|clear|export>\r\n");
        return;
    }
    
    if (strcmp(argv[1], "event") == 0) {
        // 최근 이벤트 10개
        int count = (argc >= 3) ? atoi(argv[2]) : 10;
        
        printf("\r\n=== Recent Events ===\r\n");
        for (int i = 0; i < count; i++) {
            log_event_t event;
            if (Log_ReadEntry(&g_event_header, FLASH_LOG_EVENT_ADDR,
                              g_event_header.entry_count - 1 - i,
                              &event, sizeof(event)) == HAL_OK) {
                printf("%10u | %-15s | %d | %d | %d | %u\r\n",
                       event.timestamp,
                       Event_TypeToString(event.event_type),
                       event.severity,
                       event.param1,
                       event.param2,
                       event.param3);
            }
        }
    }
    else if (strcmp(argv[1], "periodic") == 0) {
        // 최근 주기적 로그 5개
        printf("\r\n=== Recent Periodic Logs ===\r\n");
        for (int i = 0; i < 5; i++) {
            log_periodic_t entry;
            if (Log_ReadEntry(&g_periodic_header, FLASH_LOG_PERIODIC_ADDR,
                              g_periodic_header.entry_count - 1 - i,
                              &entry, sizeof(entry)) == HAL_OK) {
                printf("Time: %u, SOC: %d%%, V: %d, I: %d\r\n",
                       entry.timestamp,
                       entry.soc,
                       entry.pack_voltage,
                       entry.pack_current);
            }
        }
    }
    else if (strcmp(argv[1], "clear") == 0) {
        printf("Clear all logs? (y/n): ");
        // 확인 후 클리어
    }
    else if (strcmp(argv[1], "export") == 0) {
        // CSV 형식으로 출력
        printf("timestamp,soc,pack_v,pack_i,max_cell,min_cell\r\n");
        for (uint32_t i = 0; i < g_periodic_header.entry_count; i++) {
            log_periodic_t entry;
            Log_ReadEntry(&g_periodic_header, FLASH_LOG_PERIODIC_ADDR,
                          i, &entry, sizeof(entry));
            printf("%u,%d,%d,%d,%d,%d\r\n",
                   entry.timestamp,
                   entry.soc,
                   entry.pack_voltage,
                   entry.pack_current,
                   BMS_GetMaxFromArray(entry.cell_voltage, 24),
                   BMS_GetMinFromArray(entry.cell_voltage, 24));
        }
    }
}
```

### stats - 통계

```c
void cmd_stats(int argc, char *argv[]) {
    printf("\r\n=== BMS Statistics ===\r\n");
    printf("Total Charge:    %.1f Ah\r\n", g_stats.total_charge_ah / 10.0f);
    printf("Total Discharge: %.1f Ah\r\n", g_stats.total_discharge_ah / 10.0f);
    printf("Cycle Count:     %d\r\n", g_stats.cycle_count);
    printf("Full Charges:    %d\r\n", g_stats.full_charge_count);
    printf("Operating Time:  %d hours\r\n", g_stats.operating_hours / 3600);
    printf("Charging Time:   %d hours\r\n", g_stats.charging_hours / 3600);
    printf("Discharge Time:  %d hours\r\n", g_stats.discharging_hours / 3600);
    printf("Max Temp Ever:   %d°C\r\n", g_stats.max_temp_ever);
    printf("Min Temp Ever:   %d°C\r\n", g_stats.min_temp_ever);
    printf("Max Cell Ever:   %dmV\r\n", g_stats.max_cell_mv_ever);
    printf("Min Cell Ever:   %dmV\r\n", g_stats.min_cell_mv_ever);
    printf("\r\n");
}
```

### test - 자가진단

```c
void cmd_test(int argc, char *argv[]) {
    printf("\r\n=== Self-Test ===\r\n");
    
    // 1. AD7280A 통신 테스트
    printf("AD7280A Communication... ");
    if (ad7280a_selftest(&g_ad7280a) == HAL_OK) {
        printf("OK\r\n");
    } else {
        printf("FAIL\r\n");
    }
    
    // 2. 전압 범위 체크
    printf("Cell Voltage Range... ");
    bool volt_ok = true;
    for (int i = 0; i < 24; i++) {
        if (g_bms.cell_voltage[i] < 2000 || g_bms.cell_voltage[i] > 4000) {
            volt_ok = false;
            break;
        }
    }
    printf("%s\r\n", volt_ok ? "OK" : "FAIL");
    
    // 3. 온도 범위 체크
    printf("Temperature Range... ");
    bool temp_ok = true;
    for (int i = 0; i < 4; i++) {
        if (g_bms.temperature[i] < -40 || g_bms.temperature[i] > 80) {
            temp_ok = false;
            break;
        }
    }
    printf("%s\r\n", temp_ok ? "OK" : "FAIL");
    
    // 4. CAN 통신 테스트
    printf("CAN Communication... ");
    if (BMS_CAN_SelfTest() == HAL_OK) {
        printf("OK\r\n");
    } else {
        printf("FAIL\r\n");
    }
    
    // 5. Flash 테스트
    printf("Flash Memory... ");
    if (Flash_SelfTest() == HAL_OK) {
        printf("OK\r\n");
    } else {
        printf("FAIL\r\n");
    }
    
    printf("\r\n");
}
```

### debug - 디버그 명령

```c
void cmd_debug(int argc, char *argv[]) {
    if (argc < 2) {
        printf("Usage: debug <reg|raw|dump|trace>\r\n");
        return;
    }
    
    if (strcmp(argv[1], "reg") == 0) {
        // AD7280A 레지스터 직접 읽기
        if (argc >= 4) {
            uint8_t dev = atoi(argv[2]);
            uint8_t reg = strtoul(argv[3], NULL, 16);
            uint16_t value;
            
            ad7280a_read_register(&g_ad7280a, dev, reg, &value);
            printf("Dev %d Reg 0x%02X = 0x%04X\r\n", dev, reg, value);
        }
    }
    else if (strcmp(argv[1], "raw") == 0) {
        // Raw ADC 값
        printf("Raw ADC Values:\r\n");
        for (int i = 0; i < 24; i++) {
            printf("Cell %2d: %d (0x%03X)\r\n", 
                   i + 1, 
                   g_bms.raw_adc[i],
                   g_bms.raw_adc[i]);
        }
    }
    else if (strcmp(argv[1], "dump") == 0) {
        // 메모리 덤프
        if (argc >= 4) {
            uint32_t addr = strtoul(argv[2], NULL, 16);
            uint32_t len = strtoul(argv[3], NULL, 10);
            
            printf("Memory dump at 0x%08X:\r\n", addr);
            for (uint32_t i = 0; i < len; i++) {
                if (i % 16 == 0) printf("%08X: ", addr + i);
                printf("%02X ", *(uint8_t *)(addr + i));
                if (i % 16 == 15) printf("\r\n");
            }
            printf("\r\n");
        }
    }
    else if (strcmp(argv[1], "trace") == 0) {
        // 실시간 트레이스 토글
        g_trace_enabled = !g_trace_enabled;
        printf("Trace: %s\r\n", g_trace_enabled ? "ON" : "OFF");
    }
}
```

## CAN 진단 프로토콜

### 진단 메시지 구조

```c
// CAN ID 0x500: 진단 요청
// CAN ID 0x501: 진단 응답

typedef struct {
    uint8_t service_id;     // 서비스 ID
    uint8_t sub_function;   // 서브 함수
    uint8_t data[6];        // 데이터
} diag_message_t;

// 서비스 ID (UDS 기반 간소화)
#define DIAG_SID_READ_DATA      0x22
#define DIAG_SID_WRITE_DATA     0x2E
#define DIAG_SID_IO_CONTROL     0x2F
#define DIAG_SID_ROUTINE        0x31
#define DIAG_SID_TESTER_PRESENT 0x3E

// Data Identifier (DID)
#define DID_BMS_STATUS          0x0100
#define DID_CELL_VOLTAGE_1      0x0200
#define DID_CELL_VOLTAGE_2      0x0201
#define DID_PACK_INFO           0x0210
#define DID_TEMPERATURE         0x0220
#define DID_SOC_SOH             0x0230
#define DID_STATISTICS          0x0300
#define DID_FAULT_MEMORY        0x0400
```

### 진단 요청 처리

```c
void Diag_ProcessRequest(uint8_t *data, uint8_t len) {
    uint8_t response[8] = {0};
    uint8_t resp_len = 0;
    
    uint8_t sid = data[0];
    
    switch (sid) {
    case DIAG_SID_READ_DATA:
        {
            uint16_t did = (data[1] << 8) | data[2];
            resp_len = Diag_ReadData(did, response);
        }
        break;
        
    case DIAG_SID_IO_CONTROL:
        {
            uint16_t did = (data[1] << 8) | data[2];
            uint8_t control = data[3];
            resp_len = Diag_IOControl(did, control, response);
        }
        break;
        
    case DIAG_SID_ROUTINE:
        {
            uint16_t rid = (data[1] << 8) | data[2];
            resp_len = Diag_Routine(rid, &data[3], response);
        }
        break;
        
    case DIAG_SID_TESTER_PRESENT:
        response[0] = sid + 0x40;  // Positive response
        resp_len = 1;
        break;
        
    default:
        response[0] = 0x7F;  // Negative response
        response[1] = sid;
        response[2] = 0x11;  // Service not supported
        resp_len = 3;
        break;
    }
    
    BMS_CAN_Transmit(CAN_ID_BMS_DIAG + 1, response, resp_len);
}

uint8_t Diag_ReadData(uint16_t did, uint8_t *response) {
    response[0] = DIAG_SID_READ_DATA + 0x40;
    response[1] = did >> 8;
    response[2] = did & 0xFF;
    
    switch (did) {
    case DID_BMS_STATUS:
        response[3] = g_bms.state;
        response[4] = g_bms.soc;
        response[5] = g_bms.fault ? 1 : 0;
        return 6;
        
    case DID_SOC_SOH:
        response[3] = g_bms.soc;
        response[4] = g_bms.soh;
        return 5;
        
    // ... 기타 DID
    }
    
    return 3;  // DID만 반환 (데이터 없음)
}
```

## 실시간 모니터링

```c
// 100ms 주기로 자동 출력 (trace 모드)
void CLI_TraceOutput(void) {
    if (!g_trace_enabled) return;
    
    static uint32_t last_trace = 0;
    if (HAL_GetTick() - last_trace < 100) return;
    last_trace = HAL_GetTick();
    
    // 한 줄 업데이트
    printf("\r[%s] SOC:%3d%% V:%5.1fV I:%+6.2fA T:%+3d°C dV:%3dmV  ",
           BMS_StateToString(g_bms.state),
           (int)g_bms.soc,
           g_bms.pack_voltage_mv / 1000.0f,
           g_bms.pack_current_ma / 1000.0f,
           BMS_GetMaxTemperature(),
           BMS_GetMaxCellVoltage() - BMS_GetMinCellVoltage());
}
```

## 삽질: printf 리다이렉션

UART로 printf 사용하려면:

```c
// syscalls.c 또는 main.c
int _write(int file, char *ptr, int len) {
    HAL_UART_Transmit(&huart1, (uint8_t *)ptr, len, HAL_MAX_DELAY);
    return len;
}

// 또는 직접 구현
void CLI_Printf(const char *fmt, ...) {
    char buffer[256];
    va_list args;
    va_start(args, fmt);
    int len = vsnprintf(buffer, sizeof(buffer), fmt, args);
    va_end(args);
    
    HAL_UART_Transmit(&huart1, (uint8_t *)buffer, len, 100);
}
```

## 삽질: 비동기 수신

인터럽트 기반 수신:

```c
static uint8_t g_rx_byte;

void CLI_Init(void) {
    // 1바이트 인터럽트 수신 시작
    HAL_UART_Receive_IT(&huart1, &g_rx_byte, 1);
}

void HAL_UART_RxCpltCallback(UART_HandleTypeDef *huart) {
    if (huart == &huart1) {
        CLI_ProcessChar(g_rx_byte);
        // 다음 바이트 수신 준비
        HAL_UART_Receive_IT(&huart1, &g_rx_byte, 1);
    }
}
```

## 정리

### CLI 명령어 요약

| 명령어 | 설명 |
|--------|------|
| help | 도움말 |
| status | BMS 상태 |
| cell | 셀 전압 |
| temp | 온도 |
| balance | 밸런싱 제어 |
| fet | FET 제어 |
| fault | 폴트 관리 |
| log | 로그 조회 |
| stats | 통계 |
| config | 설정 |
| test | 자가진단 |
| debug | 디버그 |

### CAN 진단 서비스

| SID | 기능 |
|-----|------|
| 0x22 | 데이터 읽기 |
| 0x2E | 데이터 쓰기 |
| 0x2F | IO 제어 |
| 0x31 | 루틴 제어 |
| 0x3E | Tester Present |

**Part 6 완료!** 🎉

---

## 시리즈 네비게이션

**Part 6: 통신 & 진단편** ✅
- [#17 - CAN 통신 프로토콜 설계](/posts/bms/ad7280a-bms-dev-17/)
- [#18 - SOC 추정 기초](/posts/bms/ad7280a-bms-dev-18/)
- [#19 - 데이터 로깅](/posts/bms/ad7280a-bms-dev-19/)
- **#20 - 진단 인터페이스** ← 현재 글

**다음**: Part 7 - 고급 기능편 (SOH, 칼만필터, 프리차지, 절연모니터링)

---

## 참고 자료

- [UDS Protocol (ISO 14229)](https://www.iso.org/standard/72439.html)
- [Embedded CLI Design](https://interrupt.memfault.com/blog/building-a-cli)
- [STM32 UART Tutorial](https://www.st.com/resource/en/application_note/an3155.pdf)
