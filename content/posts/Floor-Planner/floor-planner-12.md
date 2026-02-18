---
title: "Floor Planner 개발기 #12 - 키보드 단축키"
date: 2024-12-16
tags: ["Floor Planner", "Vue", "키보드", "단축키"]
categories: ["개발기"]
series: ["Floor Planner 개발기"]
summary: "Delete로 삭제, R로 회전, 화살표로 이동."
---

마우스만으로 조작하면 느리다. 단축키 지원하자.

---

## 단축키 목록

| 키 | 기능 |
|---|---|
| Delete | 선택된 객체 삭제 |
| R | 90도 회전 |
| Enter | 편집 폼 열기 |
| Escape | 닫기 / 선택 해제 |
| ↑↓←→ | 1px 이동 |
| Shift + 화살표 | 10px 이동 |
| M | 측정 모드 토글 |
| D | 문 열림 방향 |
| H | 문 경첩 위치 |
| Ctrl+Z | Undo |
| Ctrl+Y | Redo |

---

## 전역 이벤트 리스너

```ts
onMounted(() => {
  window.addEventListener('keydown', handleKeyDown);
});

onUnmounted(() => {
  window.removeEventListener('keydown', handleKeyDown);
});
```

컴포넌트 마운트 시 등록, 언마운트 시 해제.

---

## 핸들러

```ts
function handleKeyDown(e: KeyboardEvent) {
  // 입력 중이면 무시
  if (isInputFocused()) return;
  
  // Ctrl/Cmd 조합
  if (e.ctrlKey || e.metaKey) {
    if (e.key === 'z') {
      e.preventDefault();
      undo();
    }
    if (e.key === 'y') {
      e.preventDefault();
      redo();
    }
    return;
  }
  
  // 단일 키
  switch (e.key) {
    case 'Delete':
    case 'Backspace':
      deleteSelected();
      break;
    case 'r':
    case 'R':
      rotateSelected(90);
      break;
    case 'Enter':
      openEditForm();
      break;
    case 'Escape':
      closeAll();
      break;
    case 'm':
    case 'M':
      toggleMeasureMode();
      break;
    case 'ArrowUp':
      moveSelected(0, e.shiftKey ? -10 : -1);
      break;
    case 'ArrowDown':
      moveSelected(0, e.shiftKey ? 10 : 1);
      break;
    case 'ArrowLeft':
      moveSelected(e.shiftKey ? -10 : -1, 0);
      break;
    case 'ArrowRight':
      moveSelected(e.shiftKey ? 10 : 1, 0);
      break;
  }
}
```

---

## 입력 중 무시

텍스트 입력 중에 단축키 작동하면 안 됨:

```ts
function isInputFocused() {
  const activeElement = document.activeElement;
  const tagName = activeElement?.tagName.toLowerCase();
  
  return tagName === 'input' || tagName === 'textarea';
}
```

---

## 회전

```ts
function rotateSelected(angle: number) {
  if (!selectedId.value) return;
  
  const furniture = store.getFurniture(selectedId.value);
  if (furniture) {
    store.updateFurniture(selectedId.value, {
      rotation: (furniture.rotation + angle) % 360
    });
  }
}
```

90도씩 회전. 360 넘으면 0으로.

---

## 이동

```ts
function moveSelected(dx: number, dy: number) {
  if (!selectedId.value) return;
  
  const furniture = store.getFurniture(selectedId.value);
  if (furniture) {
    store.updateFurniture(selectedId.value, {
      x: furniture.x + dx,
      y: furniture.y + dy
    });
  }
}
```

Shift 누르면 10px, 아니면 1px.

---

## 문 전용 단축키

문 선택됐을 때만:

```ts
if (selectedDoor.value) {
  if (e.key === 'd' || e.key === 'D') {
    toggleDoorDirection();
  }
  if (e.key === 'h' || e.key === 'H') {
    toggleDoorHinge();
  }
}
```

---

## 충돌 방지

브라우저 기본 단축키랑 겹치면 안 됨.

- `Ctrl+S`: 저장 아님 (브라우저가 가로챔)
- `Ctrl+Z`: Undo는 괜찮음

`e.preventDefault()` 안 하면 브라우저 동작이 우선.

---

다음 글에서 Undo/Redo 구현.

[#13 - Undo/Redo](/posts/floor-planner/floor-planner-13/)
