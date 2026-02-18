---
title: "Floor Planner 개발기 #17 - Vitest로 Konva 테스트"
date: 2024-12-16
tags: ["Floor Planner", "Vitest", "Konva", "테스트"]
categories: ["개발기"]
series: ["Floor Planner 개발기"]
summary: "Canvas 기반 앱을 어떻게 테스트하나?"
---

테스트 코드 작성하려는데, Canvas가 문제다.

---

## 문제: jsdom에 Canvas 없음

Vitest는 기본적으로 jsdom 환경. 근데 jsdom에는 Canvas API가 없다.

```ts
// 이런 테스트가 안 됨
test('Stage 생성', () => {
  const stage = new Konva.Stage({
    container: 'container',
    width: 800,
    height: 600
  });
  // Error: Cannot read property 'getContext' of null
});
```

---

## 해결: canvas 패키지

```bash
npm install -D canvas
```

vitest.config.ts:

```ts
export default defineConfig({
  test: {
    environment: 'jsdom',
    setupFiles: ['./tests/setup.ts']
  }
});
```

tests/setup.ts:

```ts
import { vi } from 'vitest';

// Canvas 모킹
HTMLCanvasElement.prototype.getContext = vi.fn(() => ({
  fillRect: vi.fn(),
  clearRect: vi.fn(),
  getImageData: vi.fn(() => ({ data: [] })),
  putImageData: vi.fn(),
  createImageData: vi.fn(() => []),
  setTransform: vi.fn(),
  drawImage: vi.fn(),
  save: vi.fn(),
  restore: vi.fn(),
  scale: vi.fn(),
  rotate: vi.fn(),
  translate: vi.fn(),
  transform: vi.fn(),
  beginPath: vi.fn(),
  closePath: vi.fn(),
  moveTo: vi.fn(),
  lineTo: vi.fn(),
  stroke: vi.fn(),
  fill: vi.fn(),
  arc: vi.fn(),
  rect: vi.fn()
})) as any;
```

전부 모킹. 실제 렌더링은 안 되지만 에러는 안 남.

---

## 스토어 테스트

Konva 없이도 테스트 가능한 건 따로:

```ts
import { setActivePinia, createPinia } from 'pinia';
import { useFurnitureStore } from '@/stores/furniture';

describe('FurnitureStore', () => {
  beforeEach(() => {
    setActivePinia(createPinia());
  });
  
  test('가구 추가', () => {
    const store = useFurnitureStore();
    
    store.addFurniture({
      id: 'test-1',
      type: 'bed',
      name: '침대',
      x: 100,
      y: 100,
      width: 200,
      height: 150,
      rotation: 0,
      color: '#8B4513',
      shape: 'rect'
    });
    
    expect(store.items).toHaveLength(1);
    expect(store.items[0].type).toBe('bed');
  });
  
  test('가구 삭제', () => {
    const store = useFurnitureStore();
    store.addFurniture({ id: 'test-1', ... });
    
    store.deleteFurniture('test-1');
    
    expect(store.items).toHaveLength(0);
  });
});
```

---

## 유틸 함수 테스트

좌표 변환, 거리 계산 같은 건 Canvas 없이 테스트 가능:

```ts
import { calcDistance, getLShapePath } from '@/utils/geometry';

describe('Geometry Utils', () => {
  test('거리 계산', () => {
    const distance = calcDistance(
      { x: 0, y: 0 },
      { x: 3, y: 4 }
    );
    expect(distance).toBe(5);
  });
  
  test('L자형 Path 생성', () => {
    const path = getLShapePath(100, 100, 0.5, 'bottomRight');
    expect(path).toContain('M 0 0');
    expect(path).toContain('Z');
  });
});
```

---

## 컴포넌트 테스트

vue-test-utils + Canvas 모킹:

```ts
import { mount } from '@vue/test-utils';
import FurniturePanel from '@/components/FurniturePanel.vue';

test('가구 팔레트 렌더링', () => {
  const wrapper = mount(FurniturePanel);
  
  expect(wrapper.text()).toContain('침대');
  expect(wrapper.text()).toContain('책상');
});

test('가구 클릭 시 이벤트 발생', async () => {
  const wrapper = mount(FurniturePanel);
  
  await wrapper.find('[data-testid="furniture-bed"]').trigger('click');
  
  expect(wrapper.emitted('add')).toBeTruthy();
  expect(wrapper.emitted('add')[0]).toEqual(['bed']);
});
```

---

## 테스트 커버리지

```
--------------------|---------|----------|---------|---------|
File                | % Stmts | % Branch | % Funcs | % Lines |
--------------------|---------|----------|---------|---------|
stores/furniture.ts |   100   |    90    |   100   |   100   |
utils/geometry.ts   |   100   |   100    |   100   |   100   |
...                 |   ...   |    ...   |   ...   |   ...   |
--------------------|---------|----------|---------|---------|
All files           |    85   |    78    |    82   |    85   |
```

Canvas 렌더링 부분은 제외하고 85% 정도.

---

다음 글에서 Playwright E2E 테스트.

[#18 - Playwright E2E](/posts/floor-planner/floor-planner-18/)
