---
title: "Floor Planner 개발기 #9 - 회전된 객체"
date: 2024-12-15
tags: ["Floor Planner", "Konva", "회전", "바운딩박스"]
categories: ["개발기"]
series: ["Floor Planner 개발기"]
summary: "회전하면 바운딩 박스가 이상해진다."
---

가구를 45도 회전시켰다.

근데 뭔가 이상하다.

---

## 문제

100x50 사각형을 45도 회전하면?

```
원본:
┌────────────┐
│            │  50px
└────────────┘
    100px

45도 회전 후 바운딩 박스:
┌──────────────────┐
│    ╱╲            │
│   ╱  ╲           │  ~106px
│  ╱    ╲          │
│ ╱______╲         │
└──────────────────┘
      ~106px
```

회전하면 바운딩 박스가 커진다.

---

## getClientRect()

Konva에서 회전된 객체의 실제 바운딩 박스:

```ts
const rect = node.getClientRect();
// { x, y, width, height } - 회전 반영된 값
```

원래 width/height와 다르다.

---

## 충돌 감지 문제

가구끼리 겹치는지 확인하려면?

```ts
function isOverlapping(a: Furniture, b: Furniture) {
  // 단순 AABB 충돌
  return !(
    a.x + a.width < b.x ||
    a.x > b.x + b.width ||
    a.y + a.height < b.y ||
    a.y > b.y + b.height
  );
}
```

회전 안 했으면 이게 맞는데, 회전하면 틀림.

---

## 해결 1: AABB 확장

회전된 바운딩 박스로 체크:

```ts
function isOverlapping(nodeA: Konva.Node, nodeB: Konva.Node) {
  const rectA = nodeA.getClientRect();
  const rectB = nodeB.getClientRect();
  
  return !(
    rectA.x + rectA.width < rectB.x ||
    rectA.x > rectB.x + rectB.width ||
    rectA.y + rectA.height < rectB.y ||
    rectA.y > rectB.y + rectB.height
  );
}
```

근데 이건 정확하진 않음. 실제론 안 겹치는데 겹친다고 나올 수 있다.

---

## 해결 2: SAT 알고리즘

정확한 충돌 감지하려면 SAT (Separating Axis Theorem).

```ts
function getCorners(node: Konva.Node) {
  // 회전된 네 꼭짓점 좌표 계산
  const transform = node.getAbsoluteTransform();
  const points = [
    { x: 0, y: 0 },
    { x: node.width(), y: 0 },
    { x: node.width(), y: node.height() },
    { x: 0, y: node.height() }
  ];
  
  return points.map(p => transform.point(p));
}

function satCollision(cornersA: Point[], cornersB: Point[]) {
  // ... SAT 구현
}
```

복잡하다. 평면도 도구에선 오버스펙.

---

## 타협: AABB로 충분

어차피 가구 겹쳐도 에러는 아님. 사용자가 알아서 조절하면 됨.

그냥 AABB 체크만 하고, 살짝 겹치는 건 무시하기로 했다.

충돌 감지는 "대충 경고" 정도로만 쓰면 됨.

---

## 회전 중심점

Konva는 기본적으로 좌상단이 회전 중심.

중앙 기준으로 회전하려면:

```ts
{
  offsetX: width / 2,
  offsetY: height / 2,
  x: x + width / 2,
  y: y + height / 2
}
```

offset 설정하면 그 점이 기준이 됨.

---

다음 글에서 문 시스템.

[#10 - 문 시스템](/posts/floor-planner/floor-planner-10/)
