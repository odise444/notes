---
title: "Floor Planner 개발기 #4 - 무한 그리드"
date: 2024-12-15
tags: ["Floor Planner", "Canvas", "Konva", "그리드"]
categories: ["개발기"]
series: ["Floor Planner 개발기"]
summary: "그리드 선 수천 개 그렸다가 렉 걸림."
---

평면도엔 그리드 배경이 필요하다. 눈금 있어야 배치하기 편함.

![빈 캔버스](/imgs/fp-empty.png)

---

## 첫 시도: 전부 그리기

단순하게 생각했다.

```ts
const lines = [];
for (let x = 0; x < 10000; x += 10) {
  lines.push({ points: [x, 0, x, 10000] });
}
for (let y = 0; y < 10000; y += 10) {
  lines.push({ points: [0, y, 10000, y] });
}
```

10px 간격으로 선 2000개.

**렉 걸림.** 줌/팬할 때마다 버벅거린다.

---

## 해결: viewport만 그리기

화면에 보이는 영역만 그리면 된다.

```ts
function getVisibleArea(stage: Konva.Stage) {
  const transform = stage.getAbsoluteTransform().copy().invert();
  
  const topLeft = transform.point({ x: 0, y: 0 });
  const bottomRight = transform.point({ 
    x: stage.width(), 
    y: stage.height() 
  });
  
  return { topLeft, bottomRight };
}
```

---

## 그리드 그리기

```ts
function generateGridLines(area: VisibleArea, spacing: number) {
  const lines = [];
  
  // 시작점을 spacing 배수로 맞춤
  const startX = Math.floor(area.topLeft.x / spacing) * spacing;
  const startY = Math.floor(area.topLeft.y / spacing) * spacing;
  
  // 세로선
  for (let x = startX; x <= area.bottomRight.x; x += spacing) {
    lines.push({
      points: [x, area.topLeft.y, x, area.bottomRight.y],
      stroke: '#ddd',
      strokeWidth: 1
    });
  }
  
  // 가로선
  for (let y = startY; y <= area.bottomRight.y; y += spacing) {
    lines.push({
      points: [area.topLeft.x, y, area.bottomRight.x, y],
      stroke: '#ddd',
      strokeWidth: 1
    });
  }
  
  return lines;
}
```

이제 화면에 보이는 선만 그린다. 수십 개 정도.

---

## 줌 레벨별 간격

줌 아웃하면 선이 너무 빽빽해진다.

```ts
function getGridSpacing(scale: number) {
  if (scale > 1.5) return 10;   // 줌 인: 10px
  if (scale > 0.5) return 50;   // 보통: 50px
  return 100;                    // 줌 아웃: 100px
}
```

---

## 메인 그리드 강조

50px마다 진한 선:

```ts
const isMainLine = (pos: number) => pos % 50 === 0;

lines.push({
  points: [...],
  stroke: isMainLine(x) ? '#bbb' : '#eee',
  strokeWidth: isMainLine(x) ? 1 : 0.5
});
```

---

## 성능 팁

그리드는 자주 안 바뀌니까 별도 Layer에 두고 캐싱:

```ts
gridLayer.cache();
```

줌/팬할 때만 다시 그리면 됨.

---

다음 글에서 줌/팬 구현.

[#5 - 줌/팬](/posts/floor-planner/floor-planner-05/)
