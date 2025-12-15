---
title: "AD7280A BMS 개발 삽질기 #28 - 양산 검사 지그"
date: 2024-12-14
draft: false
tags: ["AD7280A", "BMS", "STM32", "양산", "EOL테스트", "지그"]
categories: ["BMS 개발"]
series: ["AD7280A BMS 개발"]
summary: "개발 완료! 이제 양산이다. 모든 BMS 보드를 어떻게 검사할까? 자동화된 EOL 테스트 지그 개발기."
---

## 지난 글 요약

[지난 글](/posts/bms/ad7280a-bms-dev-27/)에서 EMC 대응을 다뤘다. 이제 개발이 완료됐다. **양산**으로 가자. 수백, 수천 대를 어떻게 검사할까?

## 양산 검사의 필요성

```
개발 1대 vs 양산 1000대:

개발:
┌─────────────────────────────────┐
│  꼼꼼히 손으로 확인             │
│  시간: 2시간/대                 │
│  문제 발생 → 직접 디버깅        │
└─────────────────────────────────┘

양산:
┌─────────────────────────────────┐
│  1000대 × 2시간 = 2000시간?     │
│  = 250일 = 불가능!              │
│                                 │
│  자동화 필수!                   │
│  목표: 5분/대                   │
└─────────────────────────────────┘
```

## EOL 테스트란?

**EOL (End Of Line)** = 생산 라인 끝에서 하는 검사

```
생산 라인:
                                          
PCB 제작 → SMT 실장 → 검사 → 조립 → [EOL 테스트] → 출하
                                    ↑
                              여기서 불량 걸러내기
```

## 검사 항목 정의

### BMS EOL 테스트 항목

```c
typedef struct {
    // 전기적 테스트
    bool power_supply;          // 전원 공급 정상
    bool current_draw;          // 소비 전류 정상 범위
    
    // 통신 테스트
    bool spi_ad7280a;          // AD7280A SPI 통신
    bool can_bus;              // CAN 통신
    bool uart_debug;           // UART 디버그
    
    // 기능 테스트
    bool cell_voltage_read;    // 셀 전압 읽기
    bool cell_voltage_accuracy; // 셀 전압 정확도
    bool temperature_read;     // 온도 측정
    bool balancing_function;   // 밸런싱 기능
    bool overvoltage_alert;    // 과전압 경보
    bool undervoltage_alert;   // 저전압 경보
    bool current_sensing;      // 전류 센싱
    
    // 절연/안전 테스트
    bool isolation_resistance; // 절연 저항
    bool relay_function;       // 릴레이 동작
    
    // 펌웨어
    bool firmware_version;     // FW 버전 확인
    bool flash_crc;           // Flash CRC
    
    // 전체 결과
    bool overall_pass;
    
} eol_test_result_t;
```

### 테스트 시간 목표

| 항목 | 시간 | 비고 |
|------|------|------|
| 보드 장착 | 30초 | 수동 |
| 전원/소비전류 | 10초 | 자동 |
| 통신 테스트 | 10초 | 자동 |
| 셀 전압 | 30초 | 자동 |
| 밸런싱 | 60초 | 자동 |
| 알람 테스트 | 20초 | 자동 |
| 절연 테스트 | 30초 | 자동 |
| 결과 기록 | 10초 | 자동 |
| 보드 탈거 | 30초 | 수동 |
| **합계** | **약 4분** | |

## 지그 하드웨어 설계

### 시스템 구성

```
                    EOL 테스트 지그 구성
┌─────────────────────────────────────────────────────┐
│                                                     │
│  ┌──────────┐                                       │
│  │   PC     │ ← 테스트 프로그램                     │
│  │ (Python) │                                       │
│  └────┬─────┘                                       │
│       │ USB                                         │
│       ▼                                             │
│  ┌──────────┐     ┌──────────┐     ┌──────────┐    │
│  │ 지그     │     │ 모의     │     │  DUT     │    │
│  │ 컨트롤러 │────▶│ 배터리   │────▶│  (BMS)   │    │
│  │ (STM32)  │     │ 시뮬레이터│     │          │    │
│  └──────────┘     └──────────┘     └──────────┘    │
│       │                                  │          │
│       │           ┌──────────┐          │          │
│       └──────────▶│ 계측기   │◀─────────┘          │
│                   │ (DMM/오실)│                     │
│                   └──────────┘                     │
│                                                     │
└─────────────────────────────────────────────────────┘

DUT = Device Under Test (피검사 장치)
```

### 모의 배터리 시뮬레이터

```
24채널 전압 출력 (셀 시뮬레이션):
┌─────────────────────────────────────────┐
│                                         │
│  ┌─────┐  ┌─────┐  ┌─────┐             │
│  │DAC 1│  │DAC 2│  │DAC 3│  ... × 24   │
│  │3.2V │  │3.2V │  │3.2V │             │
│  └──┬──┘  └──┬──┘  └──┬──┘             │
│     │        │        │                 │
│     ▼        ▼        ▼                 │
│  C0────C1────C2────C3──── ... ────C24   │
│                                         │
│  기능:                                  │
│  • 각 채널 독립 전압 설정               │
│  • 범위: 2.0V ~ 4.0V                    │
│  • 정밀도: ±1mV                         │
│  • 전류 싱크: 100mA (밸런싱 테스트용)   │
│                                         │
└─────────────────────────────────────────┘
```

### 지그 인터페이스 보드

```c
// 지그 컨트롤러 GPIO 할당
typedef struct {
    // DUT 전원 제어
    GPIO_TypeDef *pwr_port;
    uint16_t pwr_pin;
    
    // DAC 제어 (SPI)
    SPI_HandleTypeDef *dac_spi;
    GPIO_TypeDef *dac_cs_port;
    uint16_t dac_cs_pins[24];
    
    // DUT 통신
    CAN_HandleTypeDef *dut_can;
    UART_HandleTypeDef *dut_uart;
    
    // 계측
    ADC_HandleTypeDef *current_adc;
    
    // 상태 LED
    GPIO_TypeDef *led_port;
    uint16_t led_pass;
    uint16_t led_fail;
    uint16_t led_testing;
    
} jig_hw_t;
```

## 지그 펌웨어

### 메인 테스트 시퀀스

```c
// eol_test.c

eol_test_result_t EOL_RunFullTest(void) {
    eol_test_result_t result = {0};
    
    printf("\n");
    printf("╔════════════════════════════════════╗\n");
    printf("║       BMS EOL TEST START           ║\n");
    printf("╚════════════════════════════════════╝\n\n");
    
    // 1. 전원 테스트
    printf("[1/8] Power Supply Test... ");
    result.power_supply = Test_PowerSupply();
    result.current_draw = Test_CurrentDraw();
    printf("%s\n", result.power_supply ? "PASS" : "FAIL");
    
    // 2. 통신 테스트
    printf("[2/8] Communication Test... ");
    result.spi_ad7280a = Test_SPI_AD7280A();
    result.can_bus = Test_CAN();
    result.uart_debug = Test_UART();
    printf("%s\n", (result.spi_ad7280a && result.can_bus) ? "PASS" : "FAIL");
    
    // 3. 셀 전압 테스트
    printf("[3/8] Cell Voltage Test... ");
    result.cell_voltage_read = Test_CellVoltageRead();
    result.cell_voltage_accuracy = Test_CellVoltageAccuracy();
    printf("%s\n", result.cell_voltage_accuracy ? "PASS" : "FAIL");
    
    // 4. 온도 테스트
    printf("[4/8] Temperature Test... ");
    result.temperature_read = Test_Temperature();
    printf("%s\n", result.temperature_read ? "PASS" : "FAIL");
    
    // 5. 밸런싱 테스트
    printf("[5/8] Balancing Test... ");
    result.balancing_function = Test_Balancing();
    printf("%s\n", result.balancing_function ? "PASS" : "FAIL");
    
    // 6. 알람 테스트
    printf("[6/8] Alarm Test... ");
    result.overvoltage_alert = Test_OVAlert();
    result.undervoltage_alert = Test_UVAlert();
    printf("%s\n", (result.overvoltage_alert && result.undervoltage_alert) ? "PASS" : "FAIL");
    
    // 7. 전류 센싱 테스트
    printf("[7/8] Current Sensing Test... ");
    result.current_sensing = Test_CurrentSensing();
    printf("%s\n", result.current_sensing ? "PASS" : "FAIL");
    
    // 8. 절연/릴레이 테스트
    printf("[8/8] Safety Test... ");
    result.relay_function = Test_Relay();
    printf("%s\n", result.relay_function ? "PASS" : "FAIL");
    
    // 펌웨어 확인
    result.firmware_version = Test_FirmwareVersion();
    result.flash_crc = Test_FlashCRC();
    
    // 종합 판정
    result.overall_pass = 
        result.power_supply &&
        result.current_draw &&
        result.spi_ad7280a &&
        result.can_bus &&
        result.cell_voltage_read &&
        result.cell_voltage_accuracy &&
        result.temperature_read &&
        result.balancing_function &&
        result.overvoltage_alert &&
        result.undervoltage_alert &&
        result.current_sensing &&
        result.relay_function &&
        result.firmware_version;
    
    // 결과 표시
    printf("\n");
    if (result.overall_pass) {
        printf("╔════════════════════════════════════╗\n");
        printf("║          ✓ ALL TESTS PASSED        ║\n");
        printf("╚════════════════════════════════════╝\n");
        LED_SetPass();
    } else {
        printf("╔════════════════════════════════════╗\n");
        printf("║          ✗ TEST FAILED             ║\n");
        printf("╚════════════════════════════════════╝\n");
        LED_SetFail();
        EOL_PrintFailureDetails(&result);
    }
    
    return result;
}
```

### 셀 전압 정확도 테스트

```c
bool Test_CellVoltageAccuracy(void) {
    const float test_voltages[] = {2.5f, 3.0f, 3.2f, 3.4f, 3.6f};
    const float tolerance = 5.0f;  // ±5mV
    
    bool all_pass = true;
    
    for (int v = 0; v < 5; v++) {
        float set_voltage = test_voltages[v];
        
        // 모든 DAC를 동일 전압으로 설정
        for (int ch = 0; ch < 24; ch++) {
            DAC_SetVoltage(ch, set_voltage);
        }
        
        HAL_Delay(100);  // 안정화 대기
        
        // DUT에서 측정값 읽기 (CAN 통신)
        float measured[24];
        DUT_ReadCellVoltages(measured);
        
        // 정확도 확인
        for (int ch = 0; ch < 24; ch++) {
            float error = fabsf(measured[ch] - set_voltage * 1000.0f);
            
            if (error > tolerance) {
                printf("\n  FAIL: Cell %d @ %.3fV: error %.1fmV", 
                       ch, set_voltage, error);
                all_pass = false;
            }
        }
    }
    
    return all_pass;
}
```

### 밸런싱 기능 테스트

```c
bool Test_Balancing(void) {
    // 초기 상태: 모든 셀 3.3V
    for (int ch = 0; ch < 24; ch++) {
        DAC_SetVoltage(ch, 3.3f);
    }
    
    // 전류 측정 초기화
    float initial_current[24];
    for (int ch = 0; ch < 24; ch++) {
        initial_current[ch] = DAC_GetSinkCurrent(ch);
    }
    
    // 밸런싱 ON 명령 (홀수 셀만)
    uint32_t balance_mask = 0x555555;  // 0,2,4,6... 셀
    DUT_SetBalancing(balance_mask);
    
    HAL_Delay(500);  // 밸런싱 안정화
    
    // 전류 측정
    bool pass = true;
    for (int ch = 0; ch < 24; ch++) {
        float current = DAC_GetSinkCurrent(ch);
        float expected = (balance_mask & (1 << ch)) ? 50.0f : 0.0f;  // 50mA 예상
        
        // 밸런싱 ON인 채널: 30~70mA 범위
        if (balance_mask & (1 << ch)) {
            if (current < 30.0f || current > 70.0f) {
                printf("\n  FAIL: Cell %d balancing current: %.1fmA", ch, current);
                pass = false;
            }
        }
        // 밸런싱 OFF인 채널: 5mA 미만
        else {
            if (current > 5.0f) {
                printf("\n  FAIL: Cell %d leakage: %.1fmA", ch, current);
                pass = false;
            }
        }
    }
    
    // 밸런싱 OFF
    DUT_SetBalancing(0);
    
    return pass;
}
```

### 알람 테스트

```c
bool Test_OVAlert(void) {
    // 정상 전압 설정
    for (int ch = 0; ch < 24; ch++) {
        DAC_SetVoltage(ch, 3.3f);
    }
    HAL_Delay(100);
    
    // Alert 상태 확인 (정상이어야 함)
    if (DUT_GetAlertStatus() != 0) {
        printf("\n  FAIL: False alert in normal condition");
        return false;
    }
    
    // 과전압 조건 생성 (Cell 0만)
    DAC_SetVoltage(0, 3.70f);  // OV 임계값 초과
    HAL_Delay(100);
    
    // Alert 발생 확인
    uint32_t alert = DUT_GetAlertStatus();
    if (!(alert & ALERT_OV_CELL0)) {
        printf("\n  FAIL: OV alert not triggered for Cell 0");
        return false;
    }
    
    // 복구
    DAC_SetVoltage(0, 3.3f);
    HAL_Delay(100);
    
    // Alert 해제 확인
    DUT_ClearAlerts();
    if (DUT_GetAlertStatus() & ALERT_OV_CELL0) {
        printf("\n  FAIL: OV alert not cleared");
        return false;
    }
    
    return true;
}
```

## PC 테스트 프로그램

### Python GUI

```python
# eol_tester.py

import tkinter as tk
from tkinter import ttk
import serial
import json
from datetime import datetime

class EOLTester:
    def __init__(self):
        self.root = tk.Tk()
        self.root.title("BMS EOL Tester v1.0")
        self.root.geometry("800x600")
        
        self.serial_port = None
        self.test_count = 0
        self.pass_count = 0
        
        self.setup_ui()
        
    def setup_ui(self):
        # 시리얼 포트 선택
        frame_serial = ttk.LabelFrame(self.root, text="Connection")
        frame_serial.pack(fill="x", padx=10, pady=5)
        
        ttk.Label(frame_serial, text="Port:").pack(side="left", padx=5)
        self.port_combo = ttk.Combobox(frame_serial, values=["COM3", "COM4", "COM5"])
        self.port_combo.pack(side="left", padx=5)
        
        ttk.Button(frame_serial, text="Connect", command=self.connect).pack(side="left", padx=5)
        
        # 테스트 제어
        frame_control = ttk.LabelFrame(self.root, text="Test Control")
        frame_control.pack(fill="x", padx=10, pady=5)
        
        ttk.Label(frame_control, text="Serial:").pack(side="left", padx=5)
        self.serial_entry = ttk.Entry(frame_control, width=20)
        self.serial_entry.pack(side="left", padx=5)
        
        self.start_btn = ttk.Button(frame_control, text="START TEST", 
                                     command=self.start_test)
        self.start_btn.pack(side="left", padx=20)
        
        # 결과 표시
        frame_result = ttk.LabelFrame(self.root, text="Test Result")
        frame_result.pack(fill="both", expand=True, padx=10, pady=5)
        
        self.result_text = tk.Text(frame_result, height=20)
        self.result_text.pack(fill="both", expand=True, padx=5, pady=5)
        
        # 통계
        frame_stats = ttk.Frame(self.root)
        frame_stats.pack(fill="x", padx=10, pady=5)
        
        self.stats_label = ttk.Label(frame_stats, text="Tests: 0 | Pass: 0 | Fail: 0 | Yield: --%")
        self.stats_label.pack()
        
        # 대형 결과 표시
        self.result_label = tk.Label(self.root, text="READY", 
                                      font=("Arial", 48, "bold"),
                                      bg="gray", fg="white")
        self.result_label.pack(fill="x", padx=10, pady=10)
        
    def connect(self):
        port = self.port_combo.get()
        try:
            self.serial_port = serial.Serial(port, 115200, timeout=10)
            self.result_text.insert("end", f"Connected to {port}\n")
        except Exception as e:
            self.result_text.insert("end", f"Connection failed: {e}\n")
            
    def start_test(self):
        if not self.serial_port:
            return
            
        serial_num = self.serial_entry.get()
        if not serial_num:
            serial_num = f"AUTO_{datetime.now().strftime('%Y%m%d%H%M%S')}"
            
        self.result_label.config(text="TESTING...", bg="yellow", fg="black")
        self.root.update()
        
        # 테스트 시작 명령
        self.serial_port.write(b"TEST_START\n")
        
        # 결과 수신
        result_json = ""
        while True:
            line = self.serial_port.readline().decode().strip()
            self.result_text.insert("end", line + "\n")
            self.result_text.see("end")
            self.root.update()
            
            if line.startswith("RESULT:"):
                result_json = line[7:]
                break
                
        # 결과 파싱
        result = json.loads(result_json)
        
        # 표시 업데이트
        self.test_count += 1
        if result["overall_pass"]:
            self.pass_count += 1
            self.result_label.config(text="PASS", bg="green", fg="white")
        else:
            self.result_label.config(text="FAIL", bg="red", fg="white")
            
        # 통계 업데이트
        fail_count = self.test_count - self.pass_count
        yield_rate = (self.pass_count / self.test_count) * 100
        self.stats_label.config(
            text=f"Tests: {self.test_count} | Pass: {self.pass_count} | "
                 f"Fail: {fail_count} | Yield: {yield_rate:.1f}%"
        )
        
        # 로그 저장
        self.save_log(serial_num, result)
        
    def save_log(self, serial_num, result):
        timestamp = datetime.now().strftime("%Y-%m-%d %H:%M:%S")
        log_entry = {
            "timestamp": timestamp,
            "serial": serial_num,
            "result": result
        }
        
        with open("eol_log.json", "a") as f:
            f.write(json.dumps(log_entry) + "\n")
            
    def run(self):
        self.root.mainloop()

if __name__ == "__main__":
    app = EOLTester()
    app.run()
```

## 테스트 리포트

### 자동 리포트 생성

```python
# report_generator.py

def generate_report(test_results):
    """일일 테스트 리포트 생성"""
    
    report = f"""
╔══════════════════════════════════════════════════════════════╗
║                    BMS EOL TEST REPORT                       ║
╠══════════════════════════════════════════════════════════════╣
║  Date: {test_results['date']}                                 
║  Total Tests: {test_results['total']}                         
║  Passed: {test_results['passed']}                             
║  Failed: {test_results['failed']}                             
║  Yield: {test_results['yield']:.1f}%                          
╠══════════════════════════════════════════════════════════════╣
║  FAILURE BREAKDOWN:                                          ║
"""
    
    for failure_type, count in test_results['failures'].items():
        report += f"║  - {failure_type}: {count}\n"
        
    report += """╚══════════════════════════════════════════════════════════════╝
"""
    
    return report
```

### 불량 분석

```c
// 불량 코드 정의
typedef enum {
    FAIL_NONE = 0,
    FAIL_POWER_SHORT,           // 전원 단락
    FAIL_POWER_OPEN,            // 전원 단선
    FAIL_SPI_TIMEOUT,           // SPI 타임아웃
    FAIL_SPI_CRC,               // SPI CRC 에러
    FAIL_CELL_ACCURACY,         // 셀 전압 정확도
    FAIL_CELL_OPEN,             // 셀 탭 단선
    FAIL_NTC_OPEN,              // NTC 단선
    FAIL_NTC_SHORT,             // NTC 단락
    FAIL_BALANCE_OPEN,          // 밸런싱 안 됨
    FAIL_BALANCE_SHORT,         // 밸런싱 단락
    FAIL_OV_ALERT,              // OV 알람 미동작
    FAIL_UV_ALERT,              // UV 알람 미동작
    FAIL_CAN_ERROR,             // CAN 에러
    FAIL_RELAY,                 // 릴레이 불량
    FAIL_FW_VERSION,            // 펌웨어 버전 불일치
} fail_code_t;

// 불량 코드별 원인 힌트
const char* GetFailureHint(fail_code_t code) {
    switch (code) {
    case FAIL_POWER_SHORT:
        return "Check for solder bridges on power traces";
    case FAIL_SPI_TIMEOUT:
        return "Check AD7280A soldering, crystal";
    case FAIL_CELL_OPEN:
        return "Check cell tap connector soldering";
    case FAIL_BALANCE_OPEN:
        return "Check balance resistor, FET";
    // ...
    default:
        return "Unknown failure";
    }
}
```

## Part 8 완료!

```
╔════════════════════════════════════════════════════════════╗
║                    PART 8 COMPLETE!                        ║
╠════════════════════════════════════════════════════════════╣
║                                                            ║
║  실전 적용편 완료:                                          ║
║                                                            ║
║  #25 - 72V 팩 첫 연결     : 안전한 연결 절차               ║
║  #26 - 현장 트러블슈팅    : 실전 문제 해결                 ║
║  #27 - EMC 대응           : 노이즈와의 전쟁                ║
║  #28 - 양산 검사 지그     : EOL 테스트 자동화              ║
║                                                            ║
╚════════════════════════════════════════════════════════════╝
```

## 정리

| 항목 | 내용 |
|------|------|
| 테스트 시간 | 약 4분/대 |
| 검사 항목 | 15개 |
| 합격 기준 | 전 항목 PASS |
| 데이터 기록 | JSON 로그 |
| 리포트 | 일일 자동 생성 |

**양산 지그 핵심**:
1. 자동화로 시간 단축
2. 반복성 확보
3. 데이터 기록/추적
4. 불량 원인 분석

---

## 시리즈 네비게이션

**Part 8: 실전 적용편** ✅
- [#25 - 72V 팩 첫 연결](/posts/bms/ad7280a-bms-dev-25/)
- [#26 - 현장 트러블슈팅](/posts/bms/ad7280a-bms-dev-26/)
- [#27 - EMC 대응](/posts/bms/ad7280a-bms-dev-27/)
- **#28 - 양산 검사 지그** ← 현재 글

**다음**: Part 9 - 하드웨어 설계편 (회로도, PCB, 부품, 제작)

---

## 참고 자료

- [Production Testing Best Practices](https://www.ni.com/en/solutions/test-measurement.html)
- [Automated Test Equipment](https://www.keysight.com/us/en/solutions/manufacturing-test.html)
- [EOL Testing for Automotive](https://www.kistler.com/en/applications/industrial-automation/end-of-line-testing/)
