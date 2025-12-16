---
title: "AD7280A BMS 개발기 #10 - 패시브 밸런싱"
date: 2024-12-10
draft: false
tags: ["AD7280A", "BMS", "STM32", "밸런싱"]
categories: ["BMS 개발"]
series: ["AD7280A BMS 개발"]
summary: "높은 셀을 방전시켜서 맞추는 패시브 밸런싱. AD7280A에 스위치가 내장되어 있다."
---

배터리 셀들은 시간이 지나면 전압 편차가 생긴다. 이걸 맞춰주는 게 밸런싱이다.

패시브 밸런싱은 높은 셀을 저항으로 방전시켜서 낮은 셀에 맞추는 방식이다. 에너지를 열로 버리는 거라 효율은 안 좋지만, 회로가 간단하다.

AD7280A에는 셀당 밸런싱 스위치가 내장되어 있다.

---

회로 구성:

```
Cell+ ────┬──────── AD7280A Cell Input
          │
          │
       ┌──┴──┐
       │ 33Ω │ (방전 저항)
       └──┬──┘
          │
          ├──[SW]── AD7280A CB pin
          │
Cell- ────┴────────
```

SW를 켜면 33Ω 저항으로 전류가 흐르면서 셀이 방전된다.

```
방전 전류 = 3.3V / 33Ω ≈ 100mA
방전 전력 = 3.3V × 0.1A = 0.33W
```

---

밸런싱 제어:

```c
void AD7280A_SetBalance(uint8_t dev, uint8_t cell_mask) {
    // cell_mask: bit0=Cell1, bit1=Cell2, ... bit5=Cell6
    AD7280A_WriteReg(dev, REG_CELL_BALANCE, cell_mask);
}

// 예: Device 0의 Cell 1, 3 밸런싱 ON
AD7280A_SetBalance(0, 0b00000101);
```

---

처음에 밸런싱이 안 됐다.

레지스터 쓰고 확인해보면 값은 들어갔는데 실제로 전류가 안 흘렀다. 삽질하다 보니 CB_TIMER 설정이 필요했다.

```c
// CONTROL_HB에 CB_TIMER 설정 필요
AD7280A_WriteReg(dev, REG_CONTROL_HB, 0x1F);  // CB_TIMER = max

// 그 다음 밸런싱 ON
AD7280A_WriteReg(dev, REG_CELL_BALANCE, cell_mask);
```

CB_TIMER가 0이면 밸런싱이 바로 꺼진다. 최대값(31)으로 설정하면 약 36초 동안 유지된다.

---

밸런싱 저항 선정도 중요하다.

```
저항이 너무 작으면: 전류 크고 → 발열 심함
저항이 너무 크면: 전류 작고 → 밸런싱 시간 오래 걸림

일반적으로 50~100mA 정도로 설계
33Ω~68Ω 사이가 적당
```

우리는 33Ω으로 했는데 좀 뜨거웠다. 47Ω이 나았을 것 같다.

---

다음 글에서 밸런싱 알고리즘을 구현한다.

[#11 - 밸런싱 알고리즘](/posts/bms/ad7280a-bms-dev-11/)
