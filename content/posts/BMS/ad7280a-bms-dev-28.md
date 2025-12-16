---
title: "AD7280A BMS 개발기 #28 - 양산 검사 지그"
date: 2024-02-11
draft: false
tags: ["AD7280A", "BMS", "STM32", "양산", "EOL테스트"]
categories: ["BMS 개발"]
series: ["AD7280A BMS 개발"]
summary: "하나씩 손으로 테스트하면 하루에 10개도 힘들다. 자동화 검사 지그 만들기."
---

양산 들어가면 하루에 수십 대 테스트해야 한다.

손으로 하면 안 된다. 검사 지그가 필요하다.

---

## EOL 테스트 항목

End-Of-Line 테스트. 출하 전 검사.

```
1. 전원 인가 → 부팅 확인
2. 셀 전압 측정 정확도
3. 전류 센싱 정확도
4. 온도 센싱 확인
5. 밸런싱 동작
6. 알람 동작
7. CAN 통신
8. 시리얼 번호 기록
```

---

## 지그 구성

```
┌─────────────────────────────────────────┐
│              EOL 테스트 지그             │
├─────────────────────────────────────────┤
│                                         │
│  [기준 전압원] ─── [릴레이 매트릭스]    │
│       │                  │              │
│       └────────┬─────────┘              │
│                │                        │
│            [DUT 소켓]                   │
│                │                        │
│  [전자부하] ───┤                        │
│                │                        │
│  [USB-CAN] ────┤                        │
│                │                        │
│  [USB-UART] ───┘                        │
│                                         │
│           [PC + 테스트 SW]              │
│                                         │
└─────────────────────────────────────────┘
```

---

## 기준 전압원

셀 입력에 정확한 기준 전압 인가.

```python
# Python 테스트 스크립트
def test_cell_accuracy():
    ref_voltages = [3.000, 3.200, 3.400, 3.600]
    
    for ref in ref_voltages:
        voltage_source.set_voltage(ref)
        time.sleep(0.5)
        
        measured = dut.read_cell_voltage(0)
        error = abs(measured - ref * 1000)  # mV
        
        if error > 5:  # 5mV 허용
            return FAIL
    
    return PASS
```

---

## 릴레이 매트릭스

24셀 각각에 기준 전압 연결/분리.

```python
def connect_cell(cell_num, voltage):
    relay.disconnect_all()
    relay.connect(cell_num)
    voltage_source.set_voltage(voltage)
```

---

## 테스트 시퀀스

```python
def eol_test(serial_number):
    results = {}
    
    # 1. 전원 인가
    power.on()
    time.sleep(2)
    results['boot'] = check_boot()
    
    # 2. 셀 전압 정확도
    results['cell_accuracy'] = test_cell_accuracy()
    
    # 3. 전류 센싱
    results['current_sense'] = test_current_sense()
    
    # 4. 온도 센싱
    results['temp_sense'] = test_temp_sense()
    
    # 5. 밸런싱
    results['balance'] = test_balance()
    
    # 6. 알람
    results['alarm'] = test_alarm()
    
    # 7. CAN 통신
    results['can'] = test_can_comm()
    
    # 8. 결과 기록
    save_results(serial_number, results)
    
    power.off()
    
    return all(r == PASS for r in results.values())
```

---

## 밸런싱 테스트

```python
def test_balance():
    # 밸런싱 ON 명령
    dut.set_balance(cell=0, on=True)
    time.sleep(0.5)
    
    # 밸런싱 저항에 전류 흐르는지 확인
    current = ammeter.read()
    
    dut.set_balance(cell=0, on=False)
    
    # 100mA ± 20% 
    if 80 < current < 120:
        return PASS
    return FAIL
```

---

## 알람 테스트

```python
def test_alarm():
    # 과전압 인가
    voltage_source.set_voltage(3.8)  # OV threshold 3.7V
    time.sleep(0.5)
    
    status = dut.read_status()
    
    voltage_source.set_voltage(3.3)  # 정상으로 복귀
    
    if status['ov_fault']:
        return PASS
    return FAIL
```

---

## 테스트 시간

항목별 시간:

```
부팅: 3초
셀 정확도 (4포인트 × 24셀): 60초
전류 센싱: 10초
온도 센싱: 5초
밸런싱 (24셀): 30초
알람: 10초
CAN: 5초

총: 약 2분
```

수동으로 하면 30분. 자동화하면 2분.

---

## 합격/불합격 판정

```python
def judge_result(results):
    # 전체 PASS면 합격
    if all(r == PASS for r in results.values()):
        label_printer.print_pass(serial_number)
        return "PASS"
    else:
        # 불량 항목 표시
        failed = [k for k, v in results.items() if v == FAIL]
        label_printer.print_fail(serial_number, failed)
        return "FAIL"
```

---

## 정리

- 기준 전압원 + 릴레이 매트릭스
- Python 스크립트로 자동화
- 2분/대 → 하루 200대 가능
- 결과 DB 저장, 추적성 확보

지그 만드는 데 2주 걸렸지만, 양산 들어가면 본전 뽑는다.

---

Part 8 실전 적용편 끝.

다음은 하드웨어 설계편.

[#29 - 회로도 설계](/posts/bms/ad7280a-bms-dev-29/)
