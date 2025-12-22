---
title: "Floor Planner 개발기 #6 - 가구 시스템"
date: 2024-12-15
tags: ["Floor Planner", "Vue", "Konva", "가구"]
categories: ["개발기"]
series: ["Floor Planner 개발기"]
summary: "가구 팔레트에서 드래그해서 배치하기."
---

이제 가구를 배치해보자.

![가구 팔레트](/imgs/fp-furniture-panel.png)

---

## 가구 타입 정의

```ts
interface Furniture {
  id: string;
  type: FurnitureType;
  name: string;
  x: number;
  y: number;
  width: number;
  height: number;
  rotation: number;
  color: string;
  shape: 'rect' | 'circle' | 'ellipse' | 'lshape';
}

type FurnitureType = 
  | 'bed' 
  | 'desk' 
  | 'chair' 
  | 'sofa' 
  | 'table' 
  | 'wardrobe'
  | 'bookshelf'
  | 'tv'
  | 'refrigerator'
  | 'washingMachine';
```

10종류. 더 추가할 수 있지만 일단 이 정도.

---

## 기본값

가구마다 기본 크기가 다르다:

```ts
const FURNITURE_DEFAULTS: Record<FurnitureType, Partial<Furniture>> = {
  bed: { width: 200, height: 150, color: '#8B4513' },
  desk: { width: 120, height: 60, color: '#D2691E' },
  chair: { width: 50, height: 50, color: '#A0522D' },
  sofa: { width: 200, height: 80, color: '#708090' },
  table: { width: 150, height: 90, color: '#DEB887' },
  wardrobe: { width: 120, height: 60, color: '#8B0000' },
  bookshelf: { width: 100, height: 30, color: '#CD853F' },
  tv: { width: 120, height: 10, color: '#2F4F4F' },
  refrigerator: { width: 70, height: 70, color: '#C0C0C0' },
  washingMachine: { width: 60, height: 60, color: '#E0E0E0' }
};
```

침대는 200x150cm, 의자는 50x50cm.

---

## 드래그 앤 드롭

팔레트에서 캔버스로 드래그.

### HTML5 DnD vs Konva DnD

처음엔 HTML5 드래그 앤 드롭 썼다.

```html
<div draggable="true" @dragstart="onDragStart">
  침대
</div>
```

근데 문제: 캔버스에 drop할 때 좌표 계산이 복잡함.

### 해결: 클릭으로 변경

그냥 팔레트 클릭하면 캔버스 중앙에 배치되게 바꿨다.

```ts
function addFurniture(type: FurnitureType) {
  const defaults = FURNITURE_DEFAULTS[type];
  const center = getViewportCenter();
  
  const furniture: Furniture = {
    id: generateId(),
    type,
    name: type,
    x: center.x - defaults.width / 2,
    y: center.y - defaults.height / 2,
    ...defaults
  };
  
  store.addFurniture(furniture);
}
```

배치 후 드래그해서 위치 조정하면 됨.

---

## Pinia 스토어

```ts
export const useFurnitureStore = defineStore('furniture', () => {
  const items = ref<Furniture[]>([]);
  const selectedId = ref<string | null>(null);
  
  function addFurniture(furniture: Furniture) {
    items.value.push(furniture);
    selectedId.value = furniture.id;
  }
  
  function updateFurniture(id: string, updates: Partial<Furniture>) {
    const item = items.value.find(f => f.id === id);
    if (item) {
      Object.assign(item, updates);
    }
  }
  
  function deleteFurniture(id: string) {
    const index = items.value.findIndex(f => f.id === id);
    if (index !== -1) {
      items.value.splice(index, 1);
      if (selectedId.value === id) {
        selectedId.value = null;
      }
    }
  }
  
  return { items, selectedId, addFurniture, updateFurniture, deleteFurniture };
});
```

---

## 캔버스에 렌더링

```vue
<v-layer>
  <template v-for="furniture in store.items" :key="furniture.id">
    <FurnitureShape
      :config="furniture"
      @click="selectFurniture(furniture.id)"
      @dragend="onDragEnd"
    />
  </template>
</v-layer>
```

`FurnitureShape` 컴포넌트에서 shape에 따라 다르게 렌더링.

---

다음 글에서 다양한 모양 구현.

[#7 - 다양한 모양](/posts/floor-planner/floor-planner-07/)
