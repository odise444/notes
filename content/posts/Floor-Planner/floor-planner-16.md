---
title: "Floor Planner 개발기 #16 - PNG 내보내기"
date: 2024-12-16
tags: ["Floor Planner", "Konva", "PNG", "내보내기"]
categories: ["개발기"]
series: ["Floor Planner 개발기"]
summary: "완성된 평면도를 이미지로 저장."
---

배치 끝났으면 이미지로 뽑아서 저장하고 싶다.

---

## Konva toDataURL

Konva Stage를 이미지로 변환:

```ts
function exportImage() {
  const stage = stageRef.value?.getStage();
  if (!stage) return;
  
  const dataUrl = stage.toDataURL({
    pixelRatio: 2,  // 해상도 (2배)
    mimeType: 'image/png'
  });
  
  // 다운로드
  const a = document.createElement('a');
  a.href = dataUrl;
  a.download = 'floor-plan.png';
  a.click();
}
```

`pixelRatio: 2`면 2배 해상도. 선명함.

---

## 문제: 배경이 투명

기본적으로 Canvas 배경이 투명이라 PNG도 투명 배경.

흰 배경 원하면:

```ts
function exportImage() {
  const stage = stageRef.value?.getStage();
  
  // 임시로 흰 배경 추가
  const bgRect = new Konva.Rect({
    x: 0,
    y: 0,
    width: stage.width(),
    height: stage.height(),
    fill: 'white'
  });
  
  const layer = stage.getLayers()[0];
  layer.add(bgRect);
  bgRect.moveToBottom();
  
  const dataUrl = stage.toDataURL({ ... });
  
  // 배경 제거
  bgRect.destroy();
  
  // 다운로드
  downloadImage(dataUrl);
}
```

---

## 특정 영역만 내보내기

전체 캔버스 말고 방 영역만:

```ts
function exportRoomArea() {
  if (!room.value) return;
  
  const stage = stageRef.value?.getStage();
  const dataUrl = stage.toDataURL({
    x: room.value.x - 20,      // 여백
    y: room.value.y - 20,
    width: room.value.width + 40,
    height: room.value.height + 40,
    pixelRatio: 2
  });
  
  downloadImage(dataUrl);
}
```

---

## 해상도 선택

사용자가 해상도 선택하게:

```vue
<select v-model="exportScale">
  <option :value="1">1x (기본)</option>
  <option :value="2">2x (고해상도)</option>
  <option :value="4">4x (인쇄용)</option>
</select>
```

```ts
const dataUrl = stage.toDataURL({
  pixelRatio: exportScale.value
});
```

---

## JPEG도 지원

```ts
function exportAsJpeg() {
  const dataUrl = stage.toDataURL({
    mimeType: 'image/jpeg',
    quality: 0.9  // JPEG 품질
  });
  
  // ...
}
```

JPEG는 용량 작지만 투명 지원 안 함.

---

## 선택된 객체 하이라이트 제거

내보내기 전에 선택 표시 숨기기:

```ts
function exportImage() {
  // Transformer 숨기기
  const transformer = transformerRef.value?.getNode();
  const wasVisible = transformer?.visible();
  transformer?.visible(false);
  
  // 내보내기
  const dataUrl = stage.toDataURL({ ... });
  
  // 다시 표시
  transformer?.visible(wasVisible);
  
  downloadImage(dataUrl);
}
```

---

## 미리보기

다운로드 전에 미리보기 보여주기:

```ts
function showPreview() {
  const dataUrl = stage.toDataURL({ pixelRatio: 1 });
  previewImage.value = dataUrl;
  showPreviewModal.value = true;
}
```

모달에서 확인 후 다운로드.

---

다음 글에서 Vitest로 Konva 테스트.

[#17 - Vitest로 Konva 테스트](/posts/floor-planner/floor-planner-17/)
