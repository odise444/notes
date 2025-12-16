---
title: "AD7280A BMS 개발기 #15 - 알람 상태 읽기"
date: 2024-01-29
draft: false
tags: ["AD7280A", "BMS", "STM32", "알람", "레지스터"]
categories: ["BMS 개발"]
series: ["AD7280A BMS 개발"]
summary: "Alert 떴는데 뭐가 문제인지 모르겠다. 알람 레지스터 파헤치기."
---

Alert 핀이 떨어졌다. 이제 뭐가 문제인지 알아내야 한다.

AD7280A에 알람 관련 레지스터가 여러 개 있다.

---

## Alert 레지스터들

| 레지스터 | 주소 | 내용 |
|---------|------|------|
| ALERT | 0x18 | 셀 알람 상태 |
| CELL_OV_STATUS | 0x19 | 어떤 셀이 OV인지 |
| CELL_UV_STATUS | 0x1A | 어떤 셀이 UV인지 |
| AUX_ALERT | 0x1B | AUX 채널 알람 |

---

## ALERT 레지스터

셀별 알람 발생 여부:

```c
uint8_t alert = AD7280A_ReadRegister(dev, AD7280A_ALERT);

// 비트 해석
if (alert & 0x01) printf("Cell 1 알람\n");
if (alert & 0x02) printf("Cell 2 알람\n");
if (alert & 0x04) printf("Cell 3 알람\n");
if (alert & 0x08) printf("Cell 4 알람\n");
if (alert & 0x10) printf("Cell 5 알람\n");
if (alert & 0x20) printf("Cell 6 알람\n");
```

근데 이건 OV인지 UV인지 구분이 안 된다.

---

## OV/UV STATUS 레지스터

어떤 조건인지 구분:

```c
uint8_t ov_status = AD7280A_ReadRegister(dev, AD7280A_CELL_OV_STATUS);
uint8_t uv_status = AD7280A_ReadRegister(dev, AD7280A_CELL_UV_STATUS);

for (int cell = 0; cell < 6; cell++) {
    if (ov_status & (1 << cell)) {
        printf("Cell %d: 과전압\n", cell + 1);
    }
    if (uv_status & (1 << cell)) {
        printf("Cell %d: 저전압\n", cell + 1);
    }
}
```

---

## 알람 상태 구조체

정보를 구조체로 정리:

```c
typedef struct {
    bool ov[24];      // 과전압
    bool uv[24];      // 저전압
    bool aux_ov[8];   // AUX 과전압 (온도 등)
    bool aux_uv[8];   // AUX 저전압
} BMS_AlarmStatus_t;

void BMS_ReadAlarmStatus(BMS_AlarmStatus_t *status) {
    memset(status, 0, sizeof(BMS_AlarmStatus_t));
    
    for (int dev = 0; dev < 4; dev++) {
        uint8_t ov = AD7280A_ReadRegister(dev, AD7280A_CELL_OV_STATUS);
        uint8_t uv = AD7280A_ReadRegister(dev, AD7280A_CELL_UV_STATUS);
        
        for (int cell = 0; cell < 6; cell++) {
            int idx = dev * 6 + cell;
            status->ov[idx] = (ov & (1 << cell)) != 0;
            status->uv[idx] = (uv & (1 << cell)) != 0;
        }
    }
}
```

---

## 폴링 vs 인터럽트

인터럽트로 빠르게 감지하고, 주기적 폴링으로 상태 확인:

```c
void BMS_Task(void) {
    // 인터럽트로 잡힌 알람 처리
    if (g_alert_flag) {
        g_alert_flag = false;
        BMS_HandleAlert();
    }
    
    // 주기적 폴링 (1초마다)
    static uint32_t last_poll = 0;
    if (HAL_GetTick() - last_poll > 1000) {
        last_poll = HAL_GetTick();
        
        BMS_AlarmStatus_t status;
        BMS_ReadAlarmStatus(&status);
        BMS_CheckAlarms(&status);
    }
}
```

---

## 로그 출력

디버깅용 알람 로그:

```c
void BMS_LogAlarms(BMS_AlarmStatus_t *status) {
    for (int i = 0; i < 24; i++) {
        if (status->ov[i]) {
            printf("[ALARM] Cell %d OV: %dmV\n", i, g_cells_mv[i]);
        }
        if (status->uv[i]) {
            printf("[ALARM] Cell %d UV: %dmV\n", i, g_cells_mv[i]);
        }
    }
}
```

---

## 정리

- ALERT: 알람 발생 여부
- OV_STATUS: 과전압 셀
- UV_STATUS: 저전압 셀
- 인터럽트 + 폴링 병행

---

다음은 보호 로직 통합. FET 제어까지.

[#16 - 보호 로직](/posts/bms/ad7280a-bms-dev-16/)
