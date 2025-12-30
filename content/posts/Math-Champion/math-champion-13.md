---
title: "수학 챔피언 개발기 #13 - 모바일 최적화"
date: 2024-12-20
tags: ["React", "PWA", "모바일", "반응형"]
categories: ["개발기"]
series: ["수학 챔피언 개발기"]
summary: "폰에서 앱처럼 쓸 수 있게."
---

아이들은 폰으로 한다.

모바일 최적화 필수.

---

## 뷰포트 설정

확대/축소 막기:

```html
<meta name="viewport" content="width=device-width, initial-scale=1.0, maximum-scale=1.0, user-scalable=no">
```

---

## PWA 메타 태그

홈 화면 추가하면 앱처럼:

```html
<meta name="apple-mobile-web-app-capable" content="yes">
<meta name="apple-mobile-web-app-status-bar-style" content="default">
<meta name="theme-color" content="#f97316">
```

---

## 간단한 Manifest

인라인으로:

```html
<link rel="manifest" href="data:application/json,{
  &quot;name&quot;:&quot;수학 챔피언&quot;,
  &quot;short_name&quot;:&quot;수학&quot;,
  &quot;display&quot;:&quot;standalone&quot;,
  &quot;background_color&quot;:&quot;%23fef3c7&quot;,
  &quot;theme_color&quot;:&quot;%23f97316&quot;
}">
```

파일 따로 안 만들어도 됨.

---

## 터치 하이라이트 제거

터치할 때 파란색 박스 안 뜨게:

```css
* { 
  -webkit-tap-highlight-color: transparent; 
}
```

---

## 스크롤 바운스 막기

당겼다 놓으면 튕기는 거:

```css
body { 
  overscroll-behavior: none; 
}
```

---

## 반응형 레이아웃

최대 너비 제한:

```jsx
<div className="min-h-screen p-3">
  <div className="max-w-md mx-auto">
    {/* 내용 */}
  </div>
</div>
```

태블릿이나 PC에서도 적당한 크기.

---

## 큰 터치 영역

버튼 크기 충분히:

```jsx
<button className="py-4 text-2xl ...">
  {num}
</button>
```

`py-4` = padding-y 1rem = 16px × 2 = 32px

손가락으로 누르기 편함.

---

## 폰트 크기

작은 화면에서도 보이게:

- 제목: `text-2xl` ~ `text-3xl`
- 숫자: `text-4xl`
- 본문: `text-sm` ~ `text-base`

---

## 스크롤 영역

리더보드, 오답 노트 길어지면:

```jsx
<div className="space-y-2 max-h-80 overflow-y-auto">
  {items.map(...)}
</div>
```

`max-h-80`으로 높이 제한, `overflow-y-auto`로 스크롤.

---

## 홈 화면 추가

iPhone:
1. Safari에서 열기
2. 공유 버튼
3. "홈 화면에 추가"

Android:
1. Chrome에서 열기
2. 메뉴 → "앱 설치" 또는 "홈 화면에 추가"

브라우저 주소창 없이 전체 화면.

---

다음 글에서 난이도 시스템.

[#14 - 난이도 시스템](/posts/math-champion/math-champion-14/)
