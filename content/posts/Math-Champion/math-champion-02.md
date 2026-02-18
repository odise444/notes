---
title: "수학 챔피언 개발기 #2 - 기술 스택 선택"
date: 2025-12-14
tags:
  - React
  - CDN
  - Hugo
  - Tailwind
categories:
  - 개발기
series:
  - 수학 챔피언 개발기
summary: 빌드 없이 React 쓰는 법.
---

아이용 앱이라 빠르게 만들고 싶다.

복잡한 설정 없이.

---

## 선택지

1. **React + Vite** - 정석, 근데 빌드 필요
2. **Vue + Nuxt** - Floor Planner에서 씀
3. **Vanilla JS** - 가볍지만 상태 관리 귀찮
4. **React CDN** - 빌드 없이 React

React CDN으로 가자. HTML 한 파일로 끝.

---

## React CDN 방식

```html
<!DOCTYPE html>
<html>
<head>
  <script src="https://unpkg.com/react@18/umd/react.production.min.js"></script>
  <script src="https://unpkg.com/react-dom@18/umd/react-dom.production.min.js"></script>
  <script src="https://unpkg.com/@babel/standalone/babel.min.js"></script>
  <script src="https://cdn.tailwindcss.com"></script>
</head>
<body>
  <div id="root"></div>
  <script type="text/babel">
    // 여기에 React 코드
    function App() {
      return <h1>Hello!</h1>;
    }
    ReactDOM.createRoot(document.getElementById('root')).render(<App />);
  </script>
</body>
</html>
```

빌드 없이 JSX 사용 가능.

---

## 장점

- **파일 하나** - index.html만 있으면 됨
- **즉시 반영** - 저장하면 바로 확인
- **배포 간단** - Hugo static 폴더에 복사
- **유지보수 쉬움** - 한 파일에 다 있음

---

## 단점

- **번들 최적화 없음** - 프로덕션엔 부적합
- **Babel 런타임** - 초기 로딩 약간 느림
- **모듈 시스템 없음** - import/export 제한

근데 간단한 앱이라 상관없음.

---

## Hugo static 폴더

```
notes/
├── static/
│   └── math/
│       └── index.html   ← 여기에
├── content/
└── hugo.toml
```

빌드하면 `https://mcu2cloud.kr/math/` 로 접근 가능.

---

## Tailwind CDN

스타일링도 CDN으로:

```html
<script src="https://cdn.tailwindcss.com"></script>
```

클래스만 쓰면 됨:

```jsx
<button className="bg-orange-400 text-white py-3 rounded-xl">
  시작하기
</button>
```

---

## 폰트

아이용이니까 귀여운 폰트:

```html
<style>
  @import url('https://fonts.googleapis.com/css2?family=Jua&family=Gaegu:wght@400;700&display=swap');
  * { font-family: 'Jua', sans-serif; }
  .handwriting { font-family: 'Gaegu', cursive; }
</style>
```

- **Jua**: 둥글둥글 귀여운 폰트
- **Gaegu**: 손글씨 느낌

---

다음 글에서 기본 구조 잡기.

[#3 - 기본 구조 잡기](/posts/math-champion/math-champion-03/)
