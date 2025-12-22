---
title: "Floor Planner 개발기 #18 - Playwright E2E"
date: 2024-12-16
tags: ["Floor Planner", "Playwright", "E2E", "테스트"]
categories: ["개발기"]
series: ["Floor Planner 개발기"]
summary: "Canvas 앱 E2E 테스트. 드래그 테스트가 까다롭다."
---

단위 테스트로는 한계가 있다. 실제 브라우저에서 동작하는지 E2E 테스트.

---

## Playwright 설정

```bash
npm install -D @playwright/test
npx playwright install
```

playwright.config.ts:

```ts
export default defineConfig({
  testDir: './e2e',
  webServer: {
    command: 'npm run dev',
    url: 'http://localhost:3000',
    reuseExistingServer: true
  }
});
```

---

## 문제: Canvas 요소 선택

일반 DOM이면 `page.click('button')`으로 클릭. 근데 Canvas 안의 요소는?

Canvas는 하나의 `<canvas>` 태그. 안에 뭐가 있는지 Playwright가 모름.

---

## 해결 1: 좌표 클릭

Canvas 특정 위치 클릭:

```ts
test('가구 선택', async ({ page }) => {
  await page.goto('/');
  
  // 가구 추가
  await page.click('[data-testid="add-bed"]');
  
  // 캔버스에서 가구 위치 클릭
  const canvas = page.locator('canvas');
  await canvas.click({ position: { x: 200, y: 200 } });
  
  // 선택 확인
  await expect(page.locator('[data-testid="edit-form"]')).toBeVisible();
});
```

좌표 하드코딩은 별로지만 Canvas에선 어쩔 수 없음.

---

## 해결 2: data-testid 활용

Canvas 위에 투명 div 올려서 테스트용 마커:

```vue
<div class="relative">
  <canvas ... />
  
  <!-- 테스트용 오버레이 -->
  <div
    v-for="furniture in store.items"
    :key="furniture.id"
    :data-testid="`furniture-${furniture.id}`"
    class="absolute pointer-events-none"
    :style="{
      left: `${furniture.x}px`,
      top: `${furniture.y}px`,
      width: `${furniture.width}px`,
      height: `${furniture.height}px`
    }"
  />
</div>
```

```ts
test('가구 선택', async ({ page }) => {
  await page.click('[data-testid="furniture-test-1"]');
});
```

프로덕션에서는 이 div 숨기면 됨.

---

## 드래그 테스트

가구 드래그 이동:

```ts
test('가구 드래그 이동', async ({ page }) => {
  await page.goto('/');
  await page.click('[data-testid="add-bed"]');
  
  const canvas = page.locator('canvas');
  
  // 드래그
  await canvas.dragTo(canvas, {
    sourcePosition: { x: 200, y: 200 },
    targetPosition: { x: 400, y: 300 }
  });
  
  // 위치 변경 확인
  // ...
});
```

---

## 키보드 테스트

```ts
test('R 키로 회전', async ({ page }) => {
  await page.goto('/');
  await page.click('[data-testid="add-bed"]');
  
  // 가구 선택
  const canvas = page.locator('canvas');
  await canvas.click({ position: { x: 200, y: 200 } });
  
  // R 키
  await page.keyboard.press('r');
  
  // 회전 확인 (어떻게?)
});
```

회전 확인은... 스토어 값 체크하거나 스크린샷 비교.

---

## 스크린샷 비교

```ts
test('초기 화면', async ({ page }) => {
  await page.goto('/');
  await expect(page).toHaveScreenshot('initial.png');
});
```

첫 실행 시 스크린샷 저장. 이후 실행에서 비교.

픽셀 차이 있으면 실패.

---

## 테스트 결과

```
Running 39 tests using 4 workers

  ✓ 초기 화면 렌더링 (1.2s)
  ✓ 가구 추가 (0.8s)
  ✓ 가구 선택 (0.6s)
  ✓ 가구 드래그 (1.1s)
  ✓ 가구 삭제 (0.5s)
  ...

  39 passed (45s)
```

---

다음 글에서 성능 최적화.

[#19 - 성능 최적화](/posts/floor-planner/floor-planner-19/)
