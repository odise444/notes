---
title: "Floor Planner 개발기 #15 - 저장/불러오기"
date: 2024-12-16
tags: ["Floor Planner", "LocalStorage", "JSON"]
categories: ["개발기"]
series: ["Floor Planner 개발기"]
summary: "작업 내용 날리면 안 된다. LocalStorage에 저장."
---

열심히 배치해놓고 브라우저 닫으면 다 날아간다. 저장 기능 필요.

---

## 저장할 데이터

```ts
interface ProjectData {
  version: string;
  room: Room | null;
  furniture: Furniture[];
  doors: Door[];
  measurements: Measurement[];
  backgroundImage: BackgroundImage | null;
}
```

버전 넣어두면 나중에 포맷 바뀔 때 마이그레이션 가능.

---

## LocalStorage 저장

```ts
const STORAGE_KEY = 'floor-planner-project';

function saveProject() {
  const data: ProjectData = {
    version: '1.0',
    room: room.value,
    furniture: furnitureStore.items,
    doors: doorStore.items,
    measurements: measurements.value,
    backgroundImage: backgroundImage.value
  };
  
  localStorage.setItem(STORAGE_KEY, JSON.stringify(data));
}
```

---

## 불러오기

```ts
function loadProject() {
  const json = localStorage.getItem(STORAGE_KEY);
  if (!json) return;
  
  try {
    const data: ProjectData = JSON.parse(json);
    
    room.value = data.room;
    furnitureStore.items = data.furniture;
    doorStore.items = data.doors;
    measurements.value = data.measurements;
    backgroundImage.value = data.backgroundImage;
    
    // 배경 이미지 다시 로드
    if (data.backgroundImage?.src) {
      reloadBackgroundImage(data.backgroundImage.src);
    }
  } catch (e) {
    console.error('Failed to load project:', e);
  }
}
```

---

## 자동 저장

매번 버튼 누르기 귀찮음. 변경될 때 자동 저장:

```ts
const debouncedSave = useDebounceFn(() => {
  saveProject();
}, 1000);

watch(
  [room, () => furnitureStore.items, () => doorStore.items, measurements, backgroundImage],
  () => {
    debouncedSave();
  },
  { deep: true }
);
```

1초 디바운스. 연속 변경은 묶어서.

---

## 앱 시작 시 로드

```ts
onMounted(() => {
  loadProject();
});
```

---

## 새 프로젝트

전부 초기화:

```ts
function newProject() {
  if (!confirm('현재 작업을 삭제하고 새로 시작할까요?')) return;
  
  room.value = null;
  furnitureStore.items = [];
  doorStore.items = [];
  measurements.value = [];
  backgroundImage.value = null;
  
  localStorage.removeItem(STORAGE_KEY);
}
```

---

## 파일로 내보내기

LocalStorage 말고 파일로도 저장하고 싶으면:

```ts
function exportProject() {
  const data: ProjectData = { ... };
  const json = JSON.stringify(data, null, 2);
  const blob = new Blob([json], { type: 'application/json' });
  
  const a = document.createElement('a');
  a.href = URL.createObjectURL(blob);
  a.download = 'floor-plan.json';
  a.click();
  
  URL.revokeObjectURL(a.href);
}
```

---

## 파일 불러오기

```ts
function importProject(file: File) {
  const reader = new FileReader();
  reader.onload = (e) => {
    const json = e.target?.result as string;
    const data = JSON.parse(json);
    // ... 데이터 적용
  };
  reader.readAsText(file);
}
```

---

## 용량 제한

LocalStorage는 보통 5MB 제한.

배경 이미지가 크면 문제될 수 있음. 이미지 압축하거나 IndexedDB 쓰는 게 나을 수도.

지금은 그냥 쓰고 있음.

---

다음 글에서 PNG 내보내기.

[#16 - PNG 내보내기](/posts/floor-planner/floor-planner-16/)
