---
title: "AD7280A BMS 개발기 번외 #4 - NMC에서 LFP로 갈아타기"
date: 2024-12-15
draft: false
tags: ["AD7280A", "BMS", "NMC", "LFP", "배터리"]
categories: ["BMS 개발"]
series: ["AD7280A BMS 개발"]
summary: "'기존 BMS를 LFP 버전으로 만들어주세요.' 셀 수만 바꾸면 되는 줄 알았다."
---

영업팀에서 전화 왔다. "기존 BMS 있잖아요, LFP 버전으로 만들 수 있어요?"

기존 제품은 14S NMC (51.8V). 새 요청은 24S LFP (76.8V).

"셀 수만 바꾸면 되는 거 아닌가요?"

1주일이면 될 줄 알았다. 2개월 걸렸다.

---

## NMC vs LFP

겉보기엔 둘 다 리튬 배터리다. 근데 특성이 완전히 다르다.

| 항목 | NMC | LFP |
|------|-----|-----|
| 공칭 전압 | 3.7V | 3.2V |
| 만충 전압 | 4.2V | 3.65V |
| 방전 종지 | 3.0V | 2.5V |
| OCV 커브 | 가파름 | 평평함 |

OCV 커브가 핵심이다.

NMC는 전압만 봐도 SOC를 대충 안다. 4.0V면 80% 정도, 3.5V면 20% 정도.

LFP는 20%~80% 구간이 거의 평평하다. 3.28V가 30%일 수도 있고 60%일 수도 있다. 전압으로 SOC를 모른다.

---

## 부품 교체

첫 번째 문제. 전압 범위가 다르다.

```
14S NMC: 42~59V
24S LFP: 60~88V
```

72V 시스템을 88V까지 올리니까 부품들이 터지기 시작했다.

80V 정격 MOSFET에 88V 걸리면? 터진다. 63V 정격 전해캡? 터진다.

BOM 절반을 갈아엎었다. MOSFET 150V로, 캡 100V로, DC-DC도 입력 범위 넓은 걸로.

---

## SOC 추정 재작업

이게 진짜 문제였다.

기존 NMC용 SOC 알고리즘:

```c
// 쿨롱 카운팅 + OCV 보정
if (current < 0.5A) {
    soc_ocv = OCV_LookupSOC(voltage);
    soc = soc * 0.9 + soc_ocv * 0.1;  // OCV 보정 자주
}
```

LFP에서는 이게 안 먹힌다. OCV 테이블에서 3.28V 찾으면 SOC가 25%? 40%? 60%? 전부 비슷하다.

LFP용으로 바꿨다:

```c
if (voltage > 3.55 || voltage < 2.8) {
    // 양 끝 구간에서만 OCV 보정
    soc_ocv = OCV_LookupSOC(voltage);
    soc = soc * 0.7 + soc_ocv * 0.3;
} else {
    // 가운데 구간은 쿨롱 카운팅만 믿는다
}
```

OCV 테이블도 새로 만들어야 했다. 완충→완방 3번 반복해서 실측. 72시간.

---

## 밸런싱 전략

NMC는 셀 간 30mV 차이면 SOC 5% 차이다. 적극적으로 밸런싱해야 한다.

LFP는 평평한 구간에서 30mV 차이가 SOC 0.5% 차이일 수도 있다. 그 구간에서 밸런싱은 의미 없다.

```c
if (min_v > 3.15 && max_v < 3.45) {
    // 평평한 구간 - 임계값 높게
    if (delta > 50mV) DoBalancing();
} else {
    // 양 끝 구간 - 임계값 낮게  
    if (delta > 20mV) DoBalancing();
}
```

충전 막판(3.5V 이상)에서는 적극적으로 밸런싱해서 모든 셀이 3.65V에 도달하게 했다.

---

## 충전 종료

NMC는 CC-CV 충전. CV 구간에서 전류가 서서히 줄어든다.

LFP는 CV 구간이 거의 없다. CC 충전만으로 95%까지 간다.

```c
// LFP 충전 종료
if (max_cell >= 3.65V) return true;
if (max_cell >= 3.60V && current < 0.05C) return true;
```

---

## 정리

NMC → LFP 전환에 필요한 것:

- 부품 전압 정격 전체 재검토
- OCV 테이블 새로 측정
- SOC 알고리즘 재설계
- 밸런싱 전략 변경
- 충전 종료 조건 변경

"파라미터 몇 개 바꾸기"가 아니라 "알고리즘 재설계"였다.

배터리 화학이 다르면 BMS도 다르다.

---

## 시리즈 완결

여기까지 AD7280A BMS 개발기.

```
Part 1~3: 기초 (AD7280A 이해, SPI, 레지스터)
Part 4~6: 핵심 (밸런싱, 알람, SOC)
Part 7~8: 고급 (SOH, 칼만필터, EMC)
Part 9: 하드웨어 (회로도, PCB, 부품, 납땜)
번외: 실무 (단종, 커널드라이버, 중국산, LFP전환)
```

총 36개 글.

읽어주셔서 감사합니다.

---

## 전체 시리즈 목차

- [Part 1: 기초편](/posts/bms/ad7280a-bms-dev-1/)
- [Part 2: 드라이버편](/posts/bms/ad7280a-bms-dev-4/)
- [Part 3: 측정편](/posts/bms/ad7280a-bms-dev-7/)
- [Part 4: 밸런싱편](/posts/bms/ad7280a-bms-dev-10/)
- [Part 5: 알람편](/posts/bms/ad7280a-bms-dev-13/)
- [Part 6: 통신편](/posts/bms/ad7280a-bms-dev-17/)
- [Part 7: 고급편](/posts/bms/ad7280a-bms-dev-21/)
- [Part 8: 실전편](/posts/bms/ad7280a-bms-dev-25/)
- [Part 9: 하드웨어편](/posts/bms/ad7280a-bms-dev-29/)
- [번외편](/posts/bms/ad7280a-bms-dev-b1/)
