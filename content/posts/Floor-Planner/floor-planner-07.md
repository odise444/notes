---
title: "Floor Planner 개발기 #7 - 다양한 모양"
date: 2024-12-15
tags: ["Floor Planner", "Konva", "Canvas", "도형"]
categories: ["개발기"]
series: ["Floor Planner 개발기"]
summary: "사각형, 원형, 타원형까지는 쉬웠다. L자형에서 멘붕."
---

가구가 다 사각형은 아니다.

원형 테이블, 타원형 러그, L자형 소파도 있음.

---

## 사각형

제일 쉽다.

```vue
<v-rect
  :config="{
    x: furniture.x,
    y: furniture.y,
    width: furniture.width,
    height: furniture.height,
    fill: furniture.color,
    draggable: true
  }"
/>
```

---

## 원형

```vue
<v-circle
  :config="{
    x: furniture.x + furniture.width / 2,
    y: furniture.y + furniture.height / 2,
    radius: Math.min(furniture.width, furniture.height) / 2,
    fill: furniture.color,
    draggable: true
  }"
/>
```

주의: Circle은 중심점 기준. Rect는 좌상단 기준.

좌표 변환 신경 써야 함.

---

## 타원형

```vue
<v-ellipse
  :config="{
    x: furniture.x + furniture.width / 2,
    y: furniture.y + furniture.height / 2,
    radiusX: furniture.width / 2,
    radiusY: furniture.height / 2,
    fill: furniture.color,
    draggable: true
  }"
/>
```

타원도 중심점 기준.

---

## L자형... 문제의 시작

L자형 소파나 책상이 있다.

```
┌───────┐
│       │
│   ┌───┘
│   │
└───┘
```

이걸 어떻게 그리지?

---

## Path로 그리기

Konva Path 사용:

```ts
function getLShapePath(width: number, height: number, ratio = 0.5) {
  const w = width;
  const h = height;
  const cutW = w * ratio;
  const cutH = h * ratio;
  
  return `M 0 0 
          L ${w} 0 
          L ${w} ${h - cutH} 
          L ${w - cutW} ${h - cutH} 
          L ${w - cutW} ${h} 
          L 0 ${h} 
          Z`;
}
```

```vue
<v-path
  :config="{
    x: furniture.x,
    y: furniture.y,
    data: getLShapePath(furniture.width, furniture.height),
    fill: furniture.color,
    draggable: true
  }"
/>
```

---

## L자형 방향

L이 어느 쪽으로 꺾이는지도 필요함.

```ts
type LShapeDirection = 'topLeft' | 'topRight' | 'bottomLeft' | 'bottomRight';
```

방향별로 Path가 다름:

```ts
function getLShapePath(w: number, h: number, ratio: number, direction: LShapeDirection) {
  const cutW = w * ratio;
  const cutH = h * ratio;
  
  switch (direction) {
    case 'bottomRight':
      return `M 0 0 L ${w} 0 L ${w} ${h - cutH} L ${w - cutW} ${h - cutH} L ${w - cutW} ${h} L 0 ${h} Z`;
    case 'bottomLeft':
      return `M 0 0 L ${w} 0 L ${w} ${h} L ${cutW} ${h} L ${cutW} ${h - cutH} L 0 ${h - cutH} Z`;
    case 'topRight':
      return `M 0 0 L ${w - cutW} 0 L ${w - cutW} ${cutH} L ${w} ${cutH} L ${w} ${h} L 0 ${h} Z`;
    case 'topLeft':
      return `M ${cutW} 0 L ${w} 0 L ${w} ${h} L 0 ${h} L 0 ${cutH} L ${cutW} ${cutH} Z`;
  }
}
```

머리 아프다.

---

## 비율 조절

L자의 꺾인 부분 비율도 조절 가능하게:

```ts
interface LShapeFurniture extends Furniture {
  shape: 'lshape';
  lshapeDirection: LShapeDirection;
  lshapeRatioX: number;  // 가로 비율 (0.3 ~ 0.7)
  lshapeRatioY: number;  // 세로 비율 (0.3 ~ 0.7)
}
```

편집 폼에서 슬라이더로 조절.

---

## 컴포넌트 분기

shape에 따라 다른 컴포넌트:

```vue
<template>
  <v-rect v-if="furniture.shape === 'rect'" ... />
  <v-circle v-else-if="furniture.shape === 'circle'" ... />
  <v-ellipse v-else-if="furniture.shape === 'ellipse'" ... />
  <v-path v-else-if="furniture.shape === 'lshape'" ... />
</template>
```

---

다음 글에서 Konva Transformer.

[#8 - Konva Transformer](/posts/floor-planner/floor-planner-08/)
