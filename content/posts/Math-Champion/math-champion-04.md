---
title: "수학 챔피언 개발기 #4 - 문제 생성 로직"
date: 2024-12-20
tags: ["React", "알고리즘", "수학"]
categories: ["개발기"]
series: ["수학 챔피언 개발기"]
summary: "음수 안 나오게, 나눗셈은 딱 떨어지게."
---

문제 생성이 핵심이다.

초등 1학년이 풀 수 있는 문제만 나와야 함.

---

## 난이도 설정

```js
const DIFFICULTY_SETTINGS = {
  easy: { maxNum: 10, mulMax: 5, label: '쉬움' },
  medium: { maxNum: 20, mulMax: 9, label: '보통' },
  hard: { maxNum: 100, mulMax: 12, label: '어려움' },
  expert: { maxNum: 500, mulMax: 15, label: '전문가' },
  master: { maxNum: 1000, mulMax: 20, label: '마스터' }
};
```

- `maxNum`: 덧셈/뺄셈 최대 숫자
- `mulMax`: 곱셈/나눗셈 최대 숫자

---

## 덧셈

```js
if (op === '+') {
  num1 = Math.floor(Math.random() * maxNum) + 1;
  num2 = Math.floor(Math.random() * (maxNum - num1)) + 1;
  answer = num1 + num2;
}
```

결과가 maxNum 넘지 않도록 num2 범위 조절.

---

## 뺄셈

```js
if (op === '-') {
  num1 = Math.floor(Math.random() * maxNum) + 1;
  num2 = Math.floor(Math.random() * num1) + 1;
  answer = num1 - num2;
}
```

**핵심**: num2가 num1보다 크면 안 됨.

음수 나오면 아이 혼란.

```
✅ 8 - 3 = 5
❌ 3 - 8 = -5  ← 이러면 안 됨
```

---

## 곱셈

```js
if (op === '×') {
  num1 = Math.floor(Math.random() * mulMax) + 1;
  num2 = Math.floor(Math.random() * mulMax) + 1;
  answer = num1 * num2;
}
```

구구단 범위 내에서 생성.

---

## 나눗셈

```js
if (op === '÷') {
  num2 = Math.floor(Math.random() * mulMax) + 1;
  answer = Math.floor(Math.random() * mulMax) + 1;
  num1 = num2 * answer;
}
```

**핵심**: 역으로 생성.

먼저 나누는 수와 답을 정하고, 곱해서 num1 만듦.

```
num2 = 3
answer = 4
num1 = 3 × 4 = 12

문제: 12 ÷ 3 = 4  ← 딱 떨어짐
```

소수점 안 나옴.

---

## 연산 선택

```js
let ops = [];
if (operation === 'add') ops = ['+'];
else if (operation === 'sub') ops = ['-'];
else if (operation === 'mul') ops = ['×'];
else if (operation === 'div') ops = ['÷'];
else if (operation === 'addsub') ops = ['+', '-'];
else if (operation === 'muldiv') ops = ['×', '÷'];
else ops = ['+', '-', '×', '÷']; // all

const op = ops[Math.floor(Math.random() * ops.length)];
```

사용자가 선택한 연산만 나옴.

---

## 전체 함수

```js
const generateProblem = useCallback(() => {
  const settings = DIFFICULTY_SETTINGS[difficulty];
  const maxNum = settings.maxNum;
  const mulMax = settings.mulMax;
  
  // 연산 선택
  let ops = [];
  if (operation === 'add') ops = ['+'];
  // ... (생략)
  
  const op = ops[Math.floor(Math.random() * ops.length)];
  
  let num1, num2, answer;
  
  if (op === '+') {
    // 덧셈 로직
  } else if (op === '-') {
    // 뺄셈 로직
  } else if (op === '×') {
    // 곱셈 로직
  } else {
    // 나눗셈 로직
  }
  
  return { num1, num2, op, answer };
}, [difficulty, operation]);
```

useCallback으로 메모이제이션. difficulty나 operation 바뀔 때만 재생성.

---

다음 글에서 숫자 키패드 UI.

[#5 - 숫자 키패드 UI](/posts/math-champion/math-champion-05/)
