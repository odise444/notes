---
title: "Floor Planner 개발기 #8 - Konva Transformer"
date: 2024-12-15
tags: ["Floor Planner", "Konva", "Transformer", "리사이즈"]
categories: ["개발기"]
series: ["Floor Planner 개발기"]
summary: "선택하면 핸들 나오고, 드래그로 크기 조절."
---

가구 선택하면 이런 게 나와야 한다:

![선택 상태](/imgs/fp-selected.png)

모서리 핸들 드래그해서 크기 조절. 위에 동그라미로 회전.

---

## Konva Transformer

Konva에 Transformer라는 게 있다. 이거 쓰면 됨.

```vue
<v-transformer
  ref="transformerRef"
  :config="{
    nodes: selectedNodes,
    keepRatio: false,
    rotateEnabled: true,
    borderStroke: '#0066ff',
    anchorStroke: '#0066ff',
    anchorFill: '#ffffff',
    anchorSize: 10
  }"
/>
```

---

## 선택한 노드 연결

Transformer에 어떤 Shape을 붙일지 알려줘야 함:

```ts
const transformerRef = ref();

watch(selectedId, (id) => {
  const transformer = transformerRef.value?.getNode();
  if (!transformer) return;
  
  if (id) {
    const node = stage.findOne(`#${id}`);
    transformer.nodes([node]);
  } else {
    transformer.nodes([]);
  }
  
  transformer.getLayer().batchDraw();
});
```

Shape에 id 붙여놓고 `findOne`으로 찾는다.

---

## 크기 조절 후 동기화

Transformer로 크기 바꾸면 Shape의 scale이 바뀐다. width/height가 아니라.

이게 좀 헷갈린다.

```ts
function onTransformEnd(e: KonvaEventObject<Event>) {
  const node = e.target;
  
  // scale을 width/height에 반영
  const scaleX = node.scaleX();
  const scaleY = node.scaleY();
  
  store.updateFurniture(node.id(), {
    x: node.x(),
    y: node.y(),
    width: node.width() * scaleX,
    height: node.height() * scaleY,
    rotation: node.rotation()
  });
  
  // scale 초기화
  node.scaleX(1);
  node.scaleY(1);
}
```

---

## 종횡비 유지

`keepRatio: true`면 Shift 안 눌러도 비율 유지.

근데 가구는 비율 자유롭게 바꾸고 싶어서 `false`.

---

## 회전 핸들

기본으로 위에 동그라미가 나온다.

```ts
rotateEnabled: true,
rotationSnaps: [0, 45, 90, 135, 180, 225, 270, 315]  // 45도 스냅
```

`rotationSnaps` 주면 45도 단위로 스냅됨. 정확히 90도 맞추기 편함.

---

## 최소 크기 제한

너무 작아지면 안 됨:

```ts
boundBoxFunc: (oldBox, newBox) => {
  const minSize = 20;
  if (newBox.width < minSize || newBox.height < minSize) {
    return oldBox;
  }
  return newBox;
}
```

---

## 스타일링

기본 스타일 별로라서 바꿨다:

```ts
{
  borderStroke: '#0066ff',
  borderStrokeWidth: 1,
  anchorStroke: '#0066ff',
  anchorFill: '#ffffff',
  anchorSize: 8,
  anchorCornerRadius: 2,
  rotateAnchorOffset: 30,
  enabledAnchors: ['top-left', 'top-right', 'bottom-left', 'bottom-right']
}
```

`enabledAnchors`로 모서리만 활성화. 변 중간 핸들은 안 씀.

---

다음 글에서 회전된 객체 처리.

[#9 - 회전된 객체](/posts/floor-planner/floor-planner-09/)
