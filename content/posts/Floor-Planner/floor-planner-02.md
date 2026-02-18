---
title: "Floor Planner 개발기 #2 - 캔버스 라이브러리 선택"
date: 2024-12-15
tags: ["Floor Planner", "Canvas", "Konva", "Fabric.js"]
categories: ["개발기"]
series: ["Floor Planner 개발기"]
summary: "Fabric.js vs Konva vs PixiJS. 뭘 쓸까?"
---

평면도 도구니까 Canvas를 써야 한다. DOM으로는 한계가 있음.

근데 Canvas API를 직접 쓰면 귀찮은 게 많다. 드래그, 리사이즈, 회전... 다 직접 구현해야 함.

라이브러리를 쓰자.

---

## 후보들

### Fabric.js

가장 유명한 Canvas 라이브러리.

```js
const rect = new fabric.Rect({
  left: 100,
  top: 100,
  width: 50,
  height: 50
});
canvas.add(rect);
```

- 문서 많음, 예제 많음
- 오래됨 (안정적)
- 근데 Vue 바인딩이 좀 어색함

### Konva

Fabric.js 대안으로 많이 씀.

```js
const rect = new Konva.Rect({
  x: 100,
  y: 100,
  width: 50,
  height: 50
});
layer.add(rect);
```

- Stage → Layer → Shape 구조
- vue-konva가 있음 (공식)
- React용 react-konva도 있음

### PixiJS

게임 쪽에서 많이 씀. WebGL 기반.

- 성능 좋음
- 근데 2D 도형 그리기엔 오버스펙
- 게임 아니면 굳이?

---

## vue-konva 선택

결정적인 이유: **Vue 컴포넌트로 쓸 수 있음**.

```vue
<template>
  <v-stage :config="stageConfig">
    <v-layer>
      <v-rect :config="rectConfig" />
    </v-layer>
  </v-stage>
</template>
```

Fabric.js는 이런 식으로 못 씀. 명령형으로 canvas.add() 해야 함.

Konva는 선언적으로 쓸 수 있어서 Vue랑 잘 맞는다.

---

## 설치

```bash
npm install vue-konva konva
```

Nuxt 3에서는 플러그인으로 등록:

```ts
// plugins/konva.client.ts
import VueKonva from 'vue-konva';

export default defineNuxtPlugin((nuxtApp) => {
  nuxtApp.vueApp.use(VueKonva);
});
```

`.client.ts`인 이유: Konva는 브라우저에서만 동작함. SSR하면 에러남.

---

## 기본 구조

```vue
<v-stage :config="{ width: 800, height: 600 }">
  <v-layer>
    <!-- 배경 그리드 -->
  </v-layer>
  <v-layer>
    <!-- 방, 가구 -->
  </v-layer>
  <v-layer>
    <!-- UI (선택 박스, 치수) -->
  </v-layer>
</v-stage>
```

Layer를 나누면 성능에 좋다. 자주 바뀌는 것과 안 바뀌는 것 분리.

---

다음 글에서 좌표계 삽질.

[#3 - 좌표계 삽질](/posts/floor-planner/floor-planner-03/)
