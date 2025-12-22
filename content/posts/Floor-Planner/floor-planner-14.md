---
title: "Floor Planner 개발기 #14 - 이미지 업로드"
date: 2024-12-16
tags: ["Floor Planner", "Konva", "이미지", "업로드"]
categories: ["개발기"]
series: ["Floor Planner 개발기"]
summary: "실제 평면도 사진을 배경에 깔고 따라 그리기."
---

부동산에서 받은 평면도 이미지가 있다. 이걸 배경에 깔고 위에 가구 배치하면 편할 것 같다.

![이미지 업로드](/imgs/fp-image-upload.png)

---

## 파일 선택

```vue
<input
  type="file"
  accept="image/png,image/jpeg,image/gif,image/webp"
  @change="handleFileSelect"
/>
```

---

## 이미지 로드

```ts
function handleFileSelect(e: Event) {
  const input = e.target as HTMLInputElement;
  const file = input.files?.[0];
  if (!file) return;
  
  const reader = new FileReader();
  reader.onload = (event) => {
    const dataUrl = event.target?.result as string;
    loadImage(dataUrl);
  };
  reader.readAsDataURL(file);
}

function loadImage(dataUrl: string) {
  const img = new Image();
  img.onload = () => {
    backgroundImage.value = {
      src: dataUrl,
      width: img.width,
      height: img.height,
      x: 0,
      y: 0,
      opacity: 0.5,
      scale: 1,
      locked: false
    };
  };
  img.src = dataUrl;
}
```

---

## Konva에 렌더링

```vue
<v-image
  v-if="backgroundImage"
  :config="{
    image: imageElement,
    x: backgroundImage.x,
    y: backgroundImage.y,
    width: backgroundImage.width * backgroundImage.scale,
    height: backgroundImage.height * backgroundImage.scale,
    opacity: backgroundImage.opacity,
    draggable: !backgroundImage.locked
  }"
/>
```

Konva Image는 HTMLImageElement가 필요:

```ts
const imageElement = ref<HTMLImageElement | null>(null);

watch(() => backgroundImage.value?.src, (src) => {
  if (!src) {
    imageElement.value = null;
    return;
  }
  
  const img = new Image();
  img.onload = () => {
    imageElement.value = img;
  };
  img.src = src;
});
```

---

## 투명도 조절

이미지가 너무 진하면 위에 그린 게 안 보임.

```vue
<input
  type="range"
  min="0.1"
  max="1"
  step="0.1"
  v-model.number="backgroundImage.opacity"
/>
```

슬라이더로 0.1 ~ 1.0 조절.

---

## 크기 조절

평면도 실제 치수랑 맞추려면 크기 조절 필요:

```vue
<input
  type="range"
  min="0.1"
  max="3"
  step="0.1"
  v-model.number="backgroundImage.scale"
/>
```

눈대중으로 방 크기 맞추면 됨.

---

## 잠금 기능

배경 이미지 실수로 움직이면 짜증남.

```ts
function toggleLock() {
  backgroundImage.value.locked = !backgroundImage.value.locked;
}
```

잠금 상태면 `draggable: false`.

---

## 삭제

```ts
function removeImage() {
  backgroundImage.value = null;
  imageElement.value = null;
}
```

---

## 레이어 순서

이미지는 가구 아래에 있어야 함:

```vue
<v-layer>
  <!-- 배경 이미지 (맨 아래) -->
  <v-image ... />
</v-layer>
<v-layer>
  <!-- 가구들 (위) -->
</v-layer>
```

Layer 순서로 조절.

---

다음 글에서 저장/불러오기.

[#15 - 저장/불러오기](/posts/floor-planner/floor-planner-15/)
