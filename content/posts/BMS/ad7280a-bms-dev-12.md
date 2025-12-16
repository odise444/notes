---
title: "AD7280A BMS 개발기 #12 - 밸런싱 발열 문제"
date: 2024-12-12
draft: false
tags: ["AD7280A", "BMS", "STM32", "밸런싱", "발열"]
categories: ["BMS 개발"]
series: ["AD7280A BMS 개발"]
summary: "24셀 동시 밸런싱하면 8W 발열. PCB가 엄청 뜨거워진다."
---

밸런싱 저항 하나당 0.33W. 24개 동시에 켜면?

```
0.33W × 24 = 7.9W
```

8W가 PCB에서 나온다. 손 못 댈 정도로 뜨거워진다.

---

처음엔 그냥 다 켰다가 PCB 온도가 80도 넘게 올라갔다. 저항 근처 패턴이 변색될 뻔했다.

해결책: 동시 밸런싱 개수 제한

```c
#define MAX_SIMULTANEOUS_BALANCE    8  // 동시에 최대 8셀만

void Balancing_ApplyWithLimit(uint32_t target_mask) {
    int count = __builtin_popcount(target_mask);
    
    if (count <= MAX_SIMULTANEOUS_BALANCE) {
        // 그대로 적용
        ApplyBalanceMask(target_mask);
    } else {
        // 우선순위 높은 셀만 선택
        uint32_t limited_mask = SelectHighPriorityCells(target_mask, MAX_SIMULTANEOUS_BALANCE);
        ApplyBalanceMask(limited_mask);
    }
}
```

8셀 제한하면 약 2.6W. 이 정도는 자연 방열로 감당 가능하다.

---

체커보드 패턴도 효과가 있다.

인접한 셀을 동시에 밸런싱하면 열이 집중된다. 홀수/짝수 번호를 번갈아 밸런싱하면 열이 분산된다.

```c
void Balancing_Checkerboard(uint32_t target_mask) {
    static bool odd_phase = true;
    
    uint32_t phase_mask = odd_phase ? 0x555555 : 0xAAAAAA;
    uint32_t active_mask = target_mask & phase_mask;
    
    ApplyBalanceMask(active_mask);
    odd_phase = !odd_phase;
}
```

이렇게 하면 최대 발열이 절반으로 줄어든다.

---

PCB 설계할 때 저항 배치도 중요하다. 다닥다닥 붙이면 안 되고 분산시켜야 한다.

그리고 저항 아래에 써멀 비아를 박으면 열이 뒷면으로 빠진다. 이건 #30 PCB 레이아웃 글에서 자세히 다룬다.

---

다음 글에서 과전압/저전압 알람 설정을 다룬다.

[#13 - 과전압/저전압 알람](/posts/bms/ad7280a-bms-dev-13/)
