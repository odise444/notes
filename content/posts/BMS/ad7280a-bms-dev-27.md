---
title: "AD7280A BMS 개발기 #27 - EMC 대응"
date: 2024-12-13
draft: false
tags: ["AD7280A", "BMS", "STM32", "EMC", "노이즈"]
categories: ["BMS 개발"]
series: ["AD7280A BMS 개발"]
summary: "인버터 옆에 두면 노이즈 지옥이다. 필터랑 실드로 버텼다."
---

BMS가 인버터 바로 옆에 있다. 인버터가 스위칭할 때 엄청난 노이즈가 발생한다.

---

증상: 모터 구동 중에 셀 전압이 ±50mV씩 튀었다.

```
모터 OFF: 3200, 3201, 3199, 3200, ...
모터 ON:  3150, 3248, 3167, 3253, ... (난리)
```

---

하드웨어 대책:

1. 전원 필터 추가
```
입력 ── 100µF ── 페라이트 ── 10µF ── VDD
```

2. 디커플링 캡 추가
```
각 IC VDD 핀에 100nF 세라믹 최대한 가깝게
```

3. 케이블 실드
```
셀 탭 케이블에 실드 추가, 한쪽만 GND 연결
```

---

소프트웨어 대책:

1. 중간값 필터
```c
uint16_t MedianFilter(uint16_t *samples, int n) {
    // 정렬 후 중간값 반환
    sort(samples, n);
    return samples[n/2];
}
```

2. 오버샘플링
```c
uint16_t ReadCellAvg(int cell) {
    uint32_t sum = 0;
    for (int i = 0; i < 8; i++) {
        sum += ReadCellRaw(cell);
    }
    return sum / 8;
}
```

3. 글리치 제거
```c
// 이전 값과 너무 다르면 무시
if (abs(new_value - old_value) > 100) {
    return old_value;  // 100mV 이상 변화는 노이즈로 간주
}
```

---

대책 적용 후 노이즈가 ±50mV → ±10mV로 줄었다. 완벽하진 않지만 동작에 문제없는 수준.

---

다음 글에서 양산 검사 지그를 다룬다.

[#28 - 양산 검사 지그](/posts/bms/ad7280a-bms-dev-28/)
