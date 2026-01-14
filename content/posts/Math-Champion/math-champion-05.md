---
title: "수학 챔피언 개발기 #5 - 숫자 키패드 UI"
date: 2026-01-14
tags: ["React", "UI", "터치", "키패드"]
categories: ["개발기"]
series: ["수학 챔피언 개발기"]
summary: "폰 키보드 대신 직접 만든 키패드."
---

폰 기본 키보드는 아이한테 불편하다.

- 숫자 키보드로 전환해야 함
- 키가 작음
- 불필요한 버튼 많음

직접 키패드 만들자.

---

## 레이아웃

전화기 스타일 3x4:

```
[1] [2] [3]
[4] [5] [6]
[7] [8] [9]
[⌫] [0] [제출]
```

---

## 구현

```jsx
<div className="grid grid-cols-3 gap-2 max-w-64 mx-auto">
  {[1, 2, 3, 4, 5, 6, 7, 8, 9].map(num => (
    <button 
      key={num} 
      onClick={() => setUserAnswer(prev => String(prev) + num)}
      className="py-4 text-2xl font-bold bg-white border-2 border-gray-200 rounded-xl btn-3d"
    >
      {num}
    </button>
  ))}
  
  {/* 지우기 */}
  <button 
    onClick={() => setUserAnswer(prev => String(prev).slice(0, -1))}
    className="py-4 text-xl font-bold bg-rose-50 border-2 border-rose-200 text-rose-500 rounded-xl btn-3d"
  >
    ⌫
  </button>
  
  {/* 0 */}
  <button 
    onClick={() => setUserAnswer(prev => String(prev) + '0')}
    className="py-4 text-2xl font-bold bg-white border-2 border-gray-200 rounded-xl btn-3d"
  >
    0
  </button>
  
  {/* 제출 */}
  <button 
    onClick={checkAnswer} 
    disabled={!userAnswer}
    className="py-4 text-sm font-bold bg-gradient-to-r from-emerald-400 to-green-500 text-white rounded-xl btn-3d disabled:opacity-50"
  >
    제출
  </button>
</div>
```

---

## 터치 최적화

버튼 크기 충분히 크게:

```css
py-4  /* padding-y: 1rem = 16px */
```

손가락으로 누르기 편함.

---

## 시각적 피드백

누르면 반응:

```css
.btn-3d {
  box-shadow: 0 4px 0 rgba(0,0,0,0.2);
  transition: all 0.1s ease;
}
.btn-3d:active {
  transform: translateY(3px);
  box-shadow: 0 1px 0 rgba(0,0,0,0.2);
}
```

---

## 입력 표시

세로 계산식처럼:

```jsx
<div className="bg-amber-50 border-4 border-amber-200 rounded-2xl p-4">
  <div className="text-right font-mono">
    <div className="text-4xl font-bold">
      {String(problem.num1).padStart(4, '\u00A0')}
    </div>
    <div className="text-4xl font-bold flex items-center justify-end">
      <span className="text-orange-500 mr-2">{problem.op}</span>
      {String(problem.num2).padStart(4, '\u00A0')}
    </div>
    <div className="border-t-4 border-gray-800 mt-2 pt-2">
      <div className="text-4xl font-bold text-blue-500">
        {userAnswer ? String(userAnswer).padStart(4, '\u00A0') : '\u00A0\u00A0\u00A0?'}
      </div>
    </div>
  </div>
</div>
```

```
    7
  + 8
  ───
   15  ← 내 답
```

학교에서 배운 방식 그대로.

---

## 숫자 스피너 제거

input type="number" 쓰면 스피너 나옴. 거슬림.

```css
input[type="number"]::-webkit-inner-spin-button,
input[type="number"]::-webkit-outer-spin-button {
  -webkit-appearance: none;
  margin: 0;
}
input[type="number"] { 
  -moz-appearance: textfield; 
}
```

근데 우린 input 안 쓰고 키패드로 직접 입력해서 상관없음.

---

다음 글에서 타이머 구현.

[#6 - 타이머 구현](/posts/math-champion/math-champion-06/)
