---
title: "Floor Planner 개발기 #13 - Undo/Redo"
date: 2024-12-16
tags: ["Floor Planner", "Vue", "Undo", "Command Pattern"]
categories: ["개발기"]
series: ["Floor Planner 개발기"]
summary: "실수로 삭제했다. Ctrl+Z!"
---

삭제 버튼 잘못 눌렀다. 되돌리고 싶다.

Undo/Redo는 필수 기능.

---

## 방법 1: 전체 상태 저장

가장 단순한 방법.

```ts
const history = ref<State[]>([]);
const historyIndex = ref(-1);

function saveState() {
  // 현재 상태를 히스토리에 추가
  const state = JSON.parse(JSON.stringify(currentState.value));
  
  // 현재 위치 이후 히스토리 삭제
  history.value = history.value.slice(0, historyIndex.value + 1);
  history.value.push(state);
  historyIndex.value++;
}

function undo() {
  if (historyIndex.value > 0) {
    historyIndex.value--;
    currentState.value = JSON.parse(JSON.stringify(history.value[historyIndex.value]));
  }
}

function redo() {
  if (historyIndex.value < history.value.length - 1) {
    historyIndex.value++;
    currentState.value = JSON.parse(JSON.stringify(history.value[historyIndex.value]));
  }
}
```

단점: 상태가 크면 메모리 많이 씀.

---

## 방법 2: Command 패턴

변경 내용만 저장.

```ts
interface Command {
  execute(): void;
  undo(): void;
}

class AddFurnitureCommand implements Command {
  constructor(private furniture: Furniture) {}
  
  execute() {
    store.addFurniture(this.furniture);
  }
  
  undo() {
    store.deleteFurniture(this.furniture.id);
  }
}

class DeleteFurnitureCommand implements Command {
  constructor(private furniture: Furniture) {}
  
  execute() {
    store.deleteFurniture(this.furniture.id);
  }
  
  undo() {
    store.addFurniture(this.furniture);
  }
}
```

장점: 메모리 적게 씀.
단점: 모든 액션에 Command 클래스 만들어야 함.

---

## 나는 방법 1 선택

가구 몇 개 없어서 전체 상태 저장해도 됨.

```ts
// 상태 변경될 때마다 저장
watch(
  () => store.items,
  () => {
    saveState();
  },
  { deep: true }
);
```

---

## 깊은 복사 주의

```ts
// 얕은 복사 - 안 됨
const state = { ...currentState.value };

// 깊은 복사 - 됨
const state = JSON.parse(JSON.stringify(currentState.value));
```

`JSON.parse(JSON.stringify())`가 제일 간단. 함수나 순환 참조 없으면 잘 됨.

structuredClone도 있음:

```ts
const state = structuredClone(currentState.value);
```

---

## 히스토리 크기 제한

무한히 저장하면 메모리 터짐:

```ts
const MAX_HISTORY = 50;

function saveState() {
  // ...
  if (history.value.length > MAX_HISTORY) {
    history.value.shift();
    historyIndex.value--;
  }
}
```

---

## 연속 변경 묶기

드래그 중에 매 프레임 저장하면 히스토리가 폭발함.

드래그 끝날 때만 저장:

```ts
function onDragEnd() {
  saveState();
}

// 드래그 중에는 저장 안 함
function onDragMove() {
  // 위치만 업데이트, saveState 안 함
}
```

---

## 디바운스

또는 디바운스:

```ts
const debouncedSave = useDebounceFn(() => {
  saveState();
}, 500);
```

500ms 내 연속 변경은 하나로 묶임.

---

다음 글에서 이미지 업로드.

[#14 - 이미지 업로드](/posts/floor-planner/floor-planner-14/)
