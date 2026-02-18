---
title: "Floor Planner 개발기 #5 - 줌/팬"
date: 2024-12-15
tags: ["Floor Planner", "Canvas", "Konva", "줌", "팬"]
categories: ["개발기"]
series: ["Floor Planner 개발기"]
summary: "휠로 줌, 드래그로 팬. 마우스 위치 기준 줌이 어렵다."
---

지도 앱처럼 휠로 줌, 드래그로 이동하고 싶었다.

---

## 팬 (이동)

빈 공간 드래그하면 캔버스 이동.

```ts
// Stage 설정
const stageConfig = {
  draggable: true
};
```

Konva Stage에 `draggable: true`만 주면 끝. 쉽다.

---

## 줌

휠 이벤트로 구현.

```ts
function handleWheel(e: KonvaEventObject<WheelEvent>) {
  e.evt.preventDefault();
  
  const stage = e.target.getStage();
  const oldScale = stage.scaleX();
  
  // 줌 방향
  const direction = e.evt.deltaY > 0 ? -1 : 1;
  const scaleBy = 1.1;
  const newScale = direction > 0 
    ? oldScale * scaleBy 
    : oldScale / scaleBy;
  
  stage.scale({ x: newScale, y: newScale });
}
```

이렇게만 하면 문제가 있다.

---

## 문제: 줌 중심

위 코드는 Stage 원점(0,0) 기준으로 줌된다.

마우스 위치 기준으로 줌해야 자연스럽다. 구글 맵처럼.

---

## 해결: 마우스 위치 기준 줌

```ts
function handleWheel(e: KonvaEventObject<WheelEvent>) {
  e.evt.preventDefault();
  
  const stage = e.target.getStage();
  const oldScale = stage.scaleX();
  const pointer = stage.getPointerPosition();
  
  // 마우스 위치의 캔버스 좌표
  const mousePointTo = {
    x: (pointer.x - stage.x()) / oldScale,
    y: (pointer.y - stage.y()) / oldScale
  };
  
  // 새 스케일
  const direction = e.evt.deltaY > 0 ? -1 : 1;
  const scaleBy = 1.1;
  const newScale = direction > 0 
    ? oldScale * scaleBy 
    : oldScale / scaleBy;
  
  // 스케일 제한
  const clampedScale = Math.max(0.1, Math.min(5, newScale));
  
  stage.scale({ x: clampedScale, y: clampedScale });
  
  // 마우스 위치가 그대로 유지되도록 Stage 위치 조정
  const newPos = {
    x: pointer.x - mousePointTo.x * clampedScale,
    y: pointer.y - mousePointTo.y * clampedScale
  };
  
  stage.position(newPos);
}
```

핵심은 마지막 부분. 줌 후에 마우스 아래 점이 그대로 있도록 Stage를 옮긴다.

---

## 줌 제한

무한히 줌되면 안 됨:

```ts
const MIN_SCALE = 0.1;  // 10%
const MAX_SCALE = 5;    // 500%

const clampedScale = Math.max(MIN_SCALE, Math.min(MAX_SCALE, newScale));
```

---

## 줌 리셋 버튼

100%로 돌아가기:

```ts
function resetZoom() {
  stage.scale({ x: 1, y: 1 });
  stage.position({ x: 0, y: 0 });
}
```

---

## 모바일 핀치 줌?

안 했다. 귀찮음.

터치 이벤트 처리하려면 코드가 꽤 늘어난다. 나중에.

---

다음 글에서 가구 시스템.

[#6 - 가구 시스템](/posts/floor-planner/floor-planner-06/)
