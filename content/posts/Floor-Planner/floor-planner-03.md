---
title: "Floor Planner 개발기 #3 - 좌표계 삽질"
date: 2024-12-15
tags: ["Floor Planner", "Canvas", "Konva", "좌표계"]
categories: ["개발기"]
series: ["Floor Planner 개발기"]
summary: "스크린 좌표, 캔버스 좌표, 실제 치수. 뭐가 이렇게 많아."
---

캔버스 작업하면서 제일 헷갈리는 게 좌표계다.

---

## 세 가지 좌표

### 1. 스크린 좌표

마우스 이벤트에서 받는 좌표. `event.clientX`, `event.clientY`.

브라우저 창 기준이다.

### 2. 캔버스 좌표

Stage 내부 좌표. Shape의 x, y가 이거다.

줌/팬 하면 스크린 좌표랑 달라진다.

### 3. 실제 치수 (cm)

방 크기 300cm x 400cm 이런 거.

캔버스에서는 1cm = 1px로 쓰거나, 스케일 적용해서 쓰거나.

---

## 문제 상황

줌 200%에서 마우스 클릭했다.

```js
// 마우스 위치
const screenX = event.clientX;  // 400

// 근데 캔버스에서 실제 위치는?
const canvasX = ???  // 200이어야 함
```

줌 2배니까 스크린 400 = 캔버스 200.

---

## 변환 공식

Konva가 변환 함수를 제공한다.

```js
const stage = stageRef.value.getStage();
const pointer = stage.getPointerPosition();  // 스크린 좌표

// 캔버스 좌표로 변환
const transform = stage.getAbsoluteTransform().copy().invert();
const canvasPos = transform.point(pointer);
```

`getAbsoluteTransform().invert()`가 핵심.

---

## 유틸 함수로 빼기

매번 쓰기 귀찮으니까:

```ts
function screenToCanvas(stage: Konva.Stage, screenPos: { x: number, y: number }) {
  const transform = stage.getAbsoluteTransform().copy().invert();
  return transform.point(screenPos);
}

function canvasToScreen(stage: Konva.Stage, canvasPos: { x: number, y: number }) {
  const transform = stage.getAbsoluteTransform().copy();
  return transform.point(canvasPos);
}
```

---

## cm 변환

실제 치수 표시할 때:

```ts
const SCALE = 1;  // 1px = 1cm

function pxToCm(px: number) {
  return px / SCALE;
}

function cmToPx(cm: number) {
  return cm * SCALE;
}
```

지금은 1:1이라 의미없지만, 나중에 스케일 바꿀 때 유용함.

---

## 삽질 포인트

처음에 `event.offsetX` 썼다가 삽질함.

```js
// 이거 쓰면 안 됨
const x = event.offsetX;

// Stage의 container 기준이라 줌/팬 반영 안 됨
```

`getPointerPosition()` + 변환이 정답.

---

다음 글에서 무한 그리드 구현.

[#4 - 무한 그리드](/posts/floor-planner/floor-planner-04/)
