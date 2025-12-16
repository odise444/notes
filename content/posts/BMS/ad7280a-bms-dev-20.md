---
title: "AD7280A BMS 개발기 #20 - 진단 인터페이스"
date: 2024-02-03
draft: false
tags: ["AD7280A", "BMS", "STM32", "진단", "CLI"]
categories: ["BMS 개발"]
series: ["AD7280A BMS 개발"]
summary: "개발 중에도, 현장에서도 BMS 상태를 봐야 한다. 진단 인터페이스 만들기."
---

개발할 때마다 디버거 연결하기 귀찮다. 간단한 명령으로 상태 보고 설정할 수 있으면 좋겠다.

UART CLI 만들었다.

---

## CLI 구조

```c
typedef struct {
    const char *cmd;
    void (*handler)(int argc, char **argv);
    const char *help;
} CLI_Command_t;

const CLI_Command_t commands[] = {
    {"help",   CLI_Help,     "Show commands"},
    {"status", CLI_Status,   "Show BMS status"},
    {"cells",  CLI_Cells,    "Show cell voltages"},
    {"temp",   CLI_Temp,     "Show temperatures"},
    {"log",    CLI_Log,      "Show/clear logs"},
    {"bal",    CLI_Balance,  "Balance control"},
    {"reset",  CLI_Reset,    "Reset BMS"},
    {NULL, NULL, NULL}
};
```

---

## 명령 파싱

```c
void CLI_Process(char *line) {
    char *argv[8];
    int argc = 0;
    
    // 공백으로 분리
    char *token = strtok(line, " ");
    while (token && argc < 8) {
        argv[argc++] = token;
        token = strtok(NULL, " ");
    }
    
    if (argc == 0) return;
    
    // 명령 찾기
    for (int i = 0; commands[i].cmd; i++) {
        if (strcmp(argv[0], commands[i].cmd) == 0) {
            commands[i].handler(argc, argv);
            return;
        }
    }
    
    printf("Unknown command: %s\n", argv[0]);
}
```

---

## status 명령

```c
void CLI_Status(int argc, char **argv) {
    printf("=== BMS Status ===\n");
    printf("State: %s\n", state_names[g_bms.state]);
    printf("Pack: %.1fV, %.1fA\n", 
        g_bms.pack_voltage_mv / 1000.0f,
        g_bms.pack_current_ma / 1000.0f);
    printf("SOC: %d%%\n", g_bms.soc);
    printf("Temp: %d~%d C\n", g_bms.min_temp / 10, g_bms.max_temp / 10);
    printf("Cells: %dmV ~ %dmV (delta %dmV)\n",
        g_bms.min_cell_mv, g_bms.max_cell_mv,
        g_bms.max_cell_mv - g_bms.min_cell_mv);
}
```

출력:

```
=== BMS Status ===
State: NORMAL
Pack: 72.3V, 15.2A
SOC: 85%
Temp: 28~32 C
Cells: 3312mV ~ 3348mV (delta 36mV)
```

---

## cells 명령

```c
void CLI_Cells(int argc, char **argv) {
    printf("=== Cell Voltages ===\n");
    for (int i = 0; i < 24; i++) {
        char bal = g_balancing[i] ? '*' : ' ';
        printf("C%02d: %4dmV %c", i + 1, g_cells_mv[i], bal);
        if ((i + 1) % 4 == 0) printf("\n");
    }
}
```

출력:

```
=== Cell Voltages ===
C01: 3312mV   C02: 3320mV   C03: 3315mV   C04: 3348mV *
C05: 3318mV   C06: 3325mV   C07: 3330mV   C08: 3342mV *
...
```

`*` 표시가 밸런싱 중인 셀.

---

## bal 명령

수동 밸런싱 제어:

```c
void CLI_Balance(int argc, char **argv) {
    if (argc < 2) {
        printf("Usage: bal <on|off|auto> [cell]\n");
        return;
    }
    
    if (strcmp(argv[1], "on") == 0 && argc >= 3) {
        int cell = atoi(argv[2]);
        if (cell >= 1 && cell <= 24) {
            g_balancing[cell - 1] = true;
            printf("Cell %d balance ON\n", cell);
        }
    }
    else if (strcmp(argv[1], "off") == 0) {
        for (int i = 0; i < 24; i++) g_balancing[i] = false;
        printf("All balance OFF\n");
    }
    else if (strcmp(argv[1], "auto") == 0) {
        g_balance_mode = BALANCE_AUTO;
        printf("Auto balance mode\n");
    }
}
```

---

## 시리얼 수신

인터럽트로 한 줄씩 받기:

```c
char rx_buf[64];
int rx_idx = 0;

void HAL_UART_RxCpltCallback(UART_HandleTypeDef *huart) {
    char c = rx_char;
    
    if (c == '\r' || c == '\n') {
        rx_buf[rx_idx] = '\0';
        if (rx_idx > 0) {
            CLI_Process(rx_buf);
        }
        rx_idx = 0;
        printf("> ");  // 프롬프트
    } else {
        if (rx_idx < sizeof(rx_buf) - 1) {
            rx_buf[rx_idx++] = c;
        }
    }
    
    HAL_UART_Receive_IT(huart, &rx_char, 1);
}
```

---

## CAN 진단

UDS(Unified Diagnostic Services) 간략 버전:

```c
void CAN_DiagHandler(uint8_t *data) {
    uint8_t service = data[0];
    
    switch (service) {
        case 0x22:  // Read Data By ID
            CAN_DiagReadData(data[1], data[2]);
            break;
        case 0x2E:  // Write Data By ID
            CAN_DiagWriteData(data[1], data[2], &data[3]);
            break;
        case 0x19:  // Read DTC
            CAN_DiagReadDTC();
            break;
        case 0x14:  // Clear DTC
            CAN_DiagClearDTC();
            break;
    }
}
```

---

## 정리

- UART CLI: 개발/디버깅용
- 명령: status, cells, temp, log, bal, reset
- CAN 진단: UDS 기반

현장 가서 노트북 연결하고 `status` 치면 바로 상태 파악.

---

Part 6 통신 & 진단편 끝.

다음은 고급 기능.

[#21 - SOH 추정](/posts/bms/ad7280a-bms-dev-21/)
