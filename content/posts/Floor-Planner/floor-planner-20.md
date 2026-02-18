---
title: "Floor Planner 개발기 #20 - Vue Reactivity + Konva 충돌"
date: 2024-12-16
tags: ["Floor Planner", "Vue", "Konva", "Reactivity", "shallowRef"]
categories: ["개발기"]
series: ["Floor Planner 개발기"]
summary: "ref()로 Konva 노드 감쌌다가 성능 폭망."
---

어느 순간부터 앱이 느려졌다. 원인 찾느라 한참 걸렸다.

---

## 증상

- 드래그가 버벅거림
- 메모리 사용량 계속 증가
- CPU 100%

---

## 원인

Konva Stage를 ref로 감쌌다:

```ts
const stage = ref<Konva.Stage | null>(null);
```

Vue의 ref()는 객체를 깊은 반응성으로 감싼다.

Konva Stage는 내부에 수많은 프로퍼티가 있음. children, attrs, parent, 이벤트 핸들러...

이걸 전부 Proxy로 감싸버리니까 느려진 거다.

---

## 해결: shallowRef

```ts
// 안 좋음
const stage = ref<Konva.Stage | null>(null);

// 좋음
const stage = shallowRef<Konva.Stage | null>(null);
```

`shallowRef`는 최상위만 반응성. 내부 프로퍼티는 건드리지 않음.

---

## Layer, Node도 마찬가지

```ts
const transformerRef = shallowRef<Konva.Transformer | null>(null);
const layerRef = shallowRef<Konva.Layer | null>(null);
```

Konva 객체는 전부 shallowRef.

---

## 데이터와 노드 분리

Konva 노드를 직접 저장하지 말고, 데이터만 저장:

```ts
// 안 좋음
const items = ref<Konva.Rect[]>([]);

// 좋음
const itemsData = ref<FurnitureData[]>([]);
// Konva 노드는 렌더링 시 생성됨
```

스토어에는 순수 데이터만. Konva 노드는 vue-konva가 관리.

---

## computed 주의

```ts
// 문제 가능
const selectedNode = computed(() => {
  return stage.value?.findOne(`#${selectedId.value}`);
});
```

매번 findOne 호출. 캐싱 안 됨.

```ts
// 개선
const selectedNode = shallowRef<Konva.Node | null>(null);

watch(selectedId, (id) => {
  selectedNode.value = id 
    ? stage.value?.findOne(`#${id}`) 
    : null;
});
```

---

## watch의 deep 옵션

```ts
// 주의: 성능 문제 가능
watch(store.items, () => { ... }, { deep: true });
```

배열 안 객체 하나 바뀌어도 콜백 실행됨. 필요한 경우만 deep 쓰기.

---

## markRaw

절대 반응성 필요 없는 객체:

```ts
import { markRaw } from 'vue';

const konvaStage = markRaw(new Konva.Stage({ ... }));
```

`markRaw`로 감싸면 Vue가 무시함.

---

## 정리

| 상황 | 사용 |
|------|------|
| Konva 노드 참조 | shallowRef |
| 순수 데이터 | ref |
| 절대 반응성 불필요 | markRaw |
| 깊은 감시 필요할 때만 | watch + deep |

---

## 결과

shallowRef로 바꾸니까 다시 빨라졌다.

Vue + Canvas 라이브러리 조합할 때 반응성 범위 조심해야 한다.

---

## 시리즈 끝

20개 글에 걸쳐 Floor Planner 개발 과정을 정리했다.

요약:
- vue-konva로 Canvas 앱 만들기
- 좌표계, 줌/팬, 그리드
- 가구/문 시스템
- 키보드 단축키, Undo/Redo
- 저장/불러오기, 이미지 내보내기
- 테스트 (Vitest + Playwright)
- 성능 최적화

직접 써보기: [Floor Planner](/floor-planner/)

---

[#1 - 왜 만들게 됐나](/posts/floor-planner/floor-planner-01/)로 돌아가기
