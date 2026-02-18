---
title: "Floor Planner 개발기 #19 - 성능 최적화"
date: 2024-12-16
tags: ["Floor Planner", "Konva", "성능", "최적화"]
categories: ["개발기"]
series: ["Floor Planner 개발기"]
summary: "가구 많아지면 느려진다. 최적화 방법들."
---

가구 20개 넘어가니까 드래그가 버벅거린다.

---

## Layer 분리

Konva에서 Layer는 별도 Canvas. Layer 안 내용이 바뀌면 그 Layer만 다시 그림.

```vue
<v-stage>
  <v-layer ref="gridLayer">
    <!-- 그리드: 거의 안 바뀜 -->
  </v-layer>
  <v-layer ref="backgroundLayer">
    <!-- 배경 이미지: 거의 안 바뀜 -->
  </v-layer>
  <v-layer ref="mainLayer">
    <!-- 방, 가구, 문: 자주 바뀜 -->
  </v-layer>
  <v-layer ref="uiLayer">
    <!-- Transformer, 측정선: 자주 바뀜 -->
  </v-layer>
</v-stage>
```

그리드 드래그한다고 가구까지 다시 그릴 필요 없음.

---

## batchDraw

여러 변경을 한 번에:

```ts
// 안 좋음: 매번 그림
items.forEach(item => {
  item.x(item.x() + 10);
  layer.draw();  // 불필요한 반복
});

// 좋음: 한 번에
items.forEach(item => {
  item.x(item.x() + 10);
});
layer.batchDraw();  // 한 번만
```

`batchDraw()`는 requestAnimationFrame으로 다음 프레임에 그림.

---

## 캐싱

복잡한 Shape은 캐싱:

```ts
complexShape.cache();
```

비트맵으로 캐싱해서 매번 다시 그리지 않음.

변경 후에는 `clearCache()` 해야 함.

---

## 드래그 중 업데이트 최소화

드래그할 때마다 스토어 업데이트하면 느림:

```ts
// 안 좋음
function onDragMove(e) {
  store.updateFurniture(id, {
    x: e.target.x(),
    y: e.target.y()
  });  // 매 프레임 스토어 업데이트
}

// 좋음
function onDragMove(e) {
  // 아무것도 안 함. Konva가 알아서 움직임.
}

function onDragEnd(e) {
  store.updateFurniture(id, {
    x: e.target.x(),
    y: e.target.y()
  });  // 드래그 끝날 때만
}
```

---

## 불필요한 리렌더링 방지

Vue의 반응성이 너무 민감하면 문제:

```ts
// 모든 가구가 리렌더링됨
const items = computed(() => store.items);

// 개별 가구만 리렌더링
const getItem = (id: string) => computed(() => 
  store.items.find(item => item.id === id)
);
```

---

## Transformer 최적화

Transformer가 무거움. 선택된 것만 붙이기:

```ts
// 안 좋음: 모든 가구에 Transformer
<v-transformer v-for="item in items" ... />

// 좋음: 선택된 것만
<v-transformer :nodes="selectedNodes" />
```

---

## requestAnimationFrame

연속 업데이트는 RAF로:

```ts
let rafId: number;

function onMouseMove(e) {
  if (rafId) return;
  
  rafId = requestAnimationFrame(() => {
    updatePosition(e);
    rafId = 0;
  });
}
```

---

## 결과

| 상황 | 최적화 전 | 최적화 후 |
|------|----------|----------|
| 가구 30개 드래그 | 20 FPS | 55 FPS |
| 줌/팬 | 버벅 | 부드러움 |

---

다음 글에서 Vue Reactivity와 Konva 충돌.

[#20 - Vue Reactivity + Konva 충돌](/posts/floor-planner/floor-planner-20/)
