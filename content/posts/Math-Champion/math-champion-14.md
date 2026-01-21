---
title: "수학 챔피언 개발기 #14 - 난이도 시스템"
date: 2025-12-20
tags:
  - React
  - 난이도
  - 게임설계
categories:
  - 개발기
series:
  - 수학 챔피언 개발기
summary: 쉬움부터 마스터까지, 성장에 맞게.
---

처음엔 쉬운 거부터. 익숙해지면 난이도 올리기.

---

## 5단계 난이도

```jsx
const DIFFICULTY_SETTINGS = {
  easy:   { maxNum: 10,   mulMax: 5,  label: '쉬움' },
  medium: { maxNum: 20,   mulMax: 9,  label: '보통' },
  hard:   { maxNum: 100,  mulMax: 12, label: '어려움' },
  expert: { maxNum: 500,  mulMax: 15, label: '전문가' },
  master: { maxNum: 1000, mulMax: 20, label: '마스터' }
};
```

---

## 초등 1학년: 쉬움

```
maxNum: 10
mulMax: 5
```

예시:
- 3 + 5 = 8
- 9 - 4 = 5
- 2 × 3 = 6
- 10 ÷ 2 = 5

10 이하 숫자만. 받아올림 최소화.

---

## 초등 2학년: 보통

```
maxNum: 20
mulMax: 9
```

예시:
- 13 + 7 = 20
- 18 - 9 = 9
- 7 × 8 = 56
- 45 ÷ 9 = 5

두 자리 수, 구구단 전체.

---

## 초등 3~4학년: 어려움

```
maxNum: 100
mulMax: 12
```

예시:
- 47 + 38 = 85
- 73 - 29 = 44
- 11 × 12 = 132

세 자리까지, 12단까지.

---

## 고학년 이상: 전문가/마스터

```
expert: { maxNum: 500,  mulMax: 15 }
master: { maxNum: 1000, mulMax: 20 }
```

암산 훈련용.

---

## 난이도 선택 UI

```jsx
<div className="grid grid-cols-5 gap-1">
  {Object.entries(DIFFICULTY_SETTINGS).map(([key, val]) => (
    <button 
      key={key} 
      onClick={() => setDifficulty(key)}
      className={`py-2 rounded-lg text-xs font-medium btn-3d ${
        difficulty === key 
          ? 'bg-orange-400 text-white' 
          : 'bg-gray-100 text-gray-600'
      }`}
    >
      {val.label}
    </button>
  ))}
</div>
```

5개 버튼 가로 배열.

---

## 연산 선택

```jsx
<div className="grid grid-cols-4 gap-1 mb-1">
  {[
    { key: 'add', label: '➕' }, 
    { key: 'sub', label: '➖' }, 
    { key: 'mul', label: '✖️' }, 
    { key: 'div', label: '➗' }
  ].map(({ key, label }) => (
    <button 
      key={key} 
      onClick={() => setOperation(key)}
      className={`py-2 rounded-lg text-lg font-medium btn-3d ${
        operation === key 
          ? 'bg-rose-400 text-white' 
          : 'bg-gray-100 text-gray-600'
      }`}
    >
      {label}
    </button>
  ))}
</div>

<div className="grid grid-cols-3 gap-1">
  {[
    { key: 'addsub', label: '➕➖' }, 
    { key: 'muldiv', label: '✖️➗' }, 
    { key: 'all', label: '전부' }
  ].map(({ key, label }) => (
    <button ...>
      {label}
    </button>
  ))}
</div>
```

- 단일 연산: +, -, ×, ÷
- 조합: 덧뺄, 곱나눗, 전부

---

## 문제 수 선택

```jsx
<div className="grid grid-cols-4 gap-2">
  {[10, 20, 30, 50].map((num) => (
    <button 
      key={num} 
      onClick={() => setTotalQuestions(num)}
      className={`py-2.5 rounded-lg text-sm font-medium btn-3d ${
        totalQuestions === num 
          ? 'bg-emerald-400 text-white' 
          : 'bg-gray-100 text-gray-600'
      }`}
    >
      {num}문제
    </button>
  ))}
</div>
```

짧게 10문제, 길게 50문제.

---

## 성장 경로

1. **쉬움 + 덧뺄 + 10문제**: 처음 시작
2. **보통 + 덧뺄 + 20문제**: 익숙해지면
3. **보통 + 전부 + 30문제**: 구구단 추가
4. **어려움 + 전부 + 50문제**: 도전

아이 속도에 맞춰 조절.

---

다음 글에서 아들 반응 & 개선점.

[#15 - 아들 반응 & 개선점](/posts/math-champion/math-champion-15/)
