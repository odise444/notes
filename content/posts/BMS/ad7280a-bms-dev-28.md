---
title: "AD7280A BMS 개발기 #28 - 양산 검사 지그"
date: 2024-12-13
draft: false
tags: ["AD7280A", "BMS", "STM32", "양산", "지그"]
categories: ["BMS 개발"]
series: ["AD7280A BMS 개발"]
summary: "한 대 테스트하는 데 2시간 걸렸다. 자동화해서 5분으로 줄였다."
---

개발할 때는 하나씩 꼼꼼히 테스트했다. 한 대당 2시간.

양산하면 하루에 50대씩 검사해야 한다. 자동화 지그가 필요했다.

---

검사 항목:

1. 전원 인가 / 소비 전류
2. SPI 통신
3. CAN 통신
4. 셀 전압 정확도 (기준 전압 vs 측정값)
5. 온도 센서 연결
6. 밸런싱 동작
7. 알람 동작

---

지그 구성:

```
PC (Python GUI)
    │ USB
지그 컨트롤러 (STM32)
    │
    ├── 전원 공급 (전류 측정 포함)
    ├── 모의 셀 전압 (DAC로 생성)
    ├── SPI 연결
    ├── CAN 연결
    └── 테스트 포인트 프로브
```

모의 배터리 대신 DAC로 셀 전압을 흉내낸다. 각 채널에 원하는 전압을 인가하고 BMS가 제대로 읽는지 확인.

---

테스트 시퀀스:

```python
def test_bms(serial_number):
    results = {}
    
    # 1. 전원/통신
    results['power'] = test_power_consumption()  # < 50mA?
    results['spi'] = test_spi_communication()
    results['can'] = test_can_communication()
    
    # 2. 셀 전압 정확도
    for v in [2.5, 3.0, 3.2, 3.4, 3.6]:
        set_all_cells(v)
        measured = read_bms_cells()
        for i, m in enumerate(measured):
            if abs(m - v*1000) > 5:  # 5mV 허용
                results[f'cell_{i}'] = 'FAIL'
    
    # 3. 밸런싱
    enable_balance(cell=0)
    current = measure_balance_current(cell=0)
    results['balance'] = 'PASS' if 80 < current < 120 else 'FAIL'
    
    # 4. 알람
    set_cell(0, 3.8)  # 과전압 유발
    alert = check_alert()
    results['ov_alert'] = 'PASS' if alert else 'FAIL'
    
    return results
```

---

검사 시간: 2시간 → 5분

하루 검사 가능 수량: 10대 → 100대

---

이걸로 실전 적용편 끝. 다음은 하드웨어 설계편이다.

[#29 - 회로도 설계](/posts/bms/ad7280a-bms-dev-29/)
