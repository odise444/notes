---
title: "AD7280A BMS 개발기 #20 - 진단 인터페이스"
date: 2024-12-13
draft: false
tags: ["AD7280A", "BMS", "STM32", "UART", "CLI"]
categories: ["BMS 개발"]
series: ["AD7280A BMS 개발"]
summary: "개발하고 디버깅하려면 CLI가 필요하다. UART로 간단하게 만들었다."
---

개발 중에 상태 확인하려면 매번 디버거 붙이기 귀찮다. UART CLI를 만들어두면 편하다.

---

간단한 명령어들:

```
> status
State: IDLE
Pack: 72.4V, 0.2A
SOC: 85%
Temp: 28°C / 31°C

> cells
Cell 01: 3.201V
Cell 02: 3.198V
Cell 03: 3.203V
...
Cell 24: 3.195V
Min: 3.195V (Cell 24)
Max: 3.203V (Cell 03)
Delta: 8mV

> balance
Balancing: OFF
Target mask: 0x000000

> log
[14:23:01] EVT_CHARGE_START
[14:35:22] EVT_PERIODIC V=72.1V I=5.2A
[14:45:22] EVT_PERIODIC V=73.5V I=4.8A
...
```

---

구현은 간단하다.

```c
void CLI_Process(char *cmd) {
    if (strcmp(cmd, "status") == 0) {
        printf("State: %s\n", StateToString(bms_state));
        printf("Pack: %.1fV, %.1fA\n", pack_voltage, pack_current);
        printf("SOC: %d%%\n", soc);
    }
    else if (strcmp(cmd, "cells") == 0) {
        for (int i = 0; i < 24; i++) {
            printf("Cell %02d: %.3fV\n", i+1, cell_mv[i]/1000.0f);
        }
    }
    else if (strcmp(cmd, "reset") == 0) {
        NVIC_SystemReset();
    }
    else {
        printf("Unknown command\n");
    }
}
```

---

나중에 양산하면 CAN으로 진단하는 UDS 프로토콜도 추가했다. 근데 개발 단계에서는 UART CLI가 훨씬 편하다.

---

이걸로 기본 기능은 끝. 다음 글부터 고급 기능을 다룬다.

[#21 - SOH 추정](/posts/bms/ad7280a-bms-dev-21/)
