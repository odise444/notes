---
title: "수학 챔피언 개발기 #7 - 점수 시스템"
date: 2025-12-14
tags:
  - React
  - 게임화
  - 점수
categories:
  - 개발기
series:
  - 수학 챔피언 개발기
summary: 연속 정답 보너스로 동기 부여.
---

점수 없으면 재미없다.

게임처럼 보상 체계가 있어야 함.

---

## 기본 점수

정답 하나에 10점:

```jsx
const [score, setScore] = useState(0);

if (isCorrect) {
  setScore(prev => prev + 10);
}
```

---

## 연속 정답 보너스

3문제 연속 맞추면 보너스:

```jsx
const [streak, setStreak] = useState(0);

if (isCorrect) {
  const streakBonus = Math.floor(streak / 3) * 5;
  const points = 10 + streakBonus;
  setScore(prev => prev + points);
  setStreak(prev => prev + 1);
} else {
  setStreak(0);  // 틀리면 연속 초기화
}
```

| 연속 | 보너스 | 총점 |
|------|--------|------|
| 0~2 | 0 | 10 |
| 3~5 | 5 | 15 |
| 6~8 | 10 | 20 |
| 9~11 | 15 | 25 |

연속하면 할수록 점수 뻥튀기.

---

## 불꽃 이펙트

연속 3개 넘으면 불꽃 표시:

```jsx
{streak >= 3 && (
  <span className="text-lg streak-fire animate-pulse">
    🔥{streak}
  </span>
)}
```

```css
.streak-fire {
  text-shadow: 0 0 8px #ff6b35, 0 0 16px #ff6b35;
}
```

아이가 좋아함. "불 붙었다!"

---

## 피드백 메시지

정답 맞추면 랜덤 칭찬:

```jsx
const ENCOURAGEMENTS = [
  '잘했어!', '멋져!', '최고야!', '대단해!', 
  '훌륭해!', '굉장해!', '완벽해!'
];

const EMOJIS = [
  '🌟', '⭐', '🎉', '🎊', '💫', 
  '✨', '🏆', '🥇', '🎯', '💪'
];

if (isCorrect) {
  setFeedback({
    correct: true,
    message: ENCOURAGEMENTS[Math.floor(Math.random() * ENCOURAGEMENTS.length)],
    points,
    emoji: EMOJIS[Math.floor(Math.random() * EMOJIS.length)]
  });
}
```

---

## 피드백 UI

```jsx
{feedback && (
  <div className={`text-center py-4 rounded-xl animate-pop ${
    feedback.correct 
      ? 'bg-gradient-to-r from-emerald-100 to-green-100' 
      : 'bg-gradient-to-r from-rose-100 to-red-100 animate-shake'
  }`}>
    <div className="text-4xl mb-1">{feedback.emoji}</div>
    <div className={`text-2xl ${
      feedback.correct ? 'text-emerald-600' : 'text-rose-600'
    }`}>
      {feedback.message}
    </div>
    {feedback.points && (
      <div className="text-lg text-emerald-500">
        +{feedback.points}점
      </div>
    )}
  </div>
)}
```

- 정답: 초록 배경, 팝 애니메이션
- 오답: 빨간 배경, 흔들림 애니메이션

---

## 상단 점수 표시

```jsx
<div className="flex justify-between items-center text-sm">
  <span className="text-gray-500">
    {questionCount + 1} / {totalQuestions}
  </span>
  <div className="flex items-center gap-3">
    {streak >= 3 && (
      <span className="text-lg streak-fire animate-pulse">
        🔥{streak}
      </span>
    )}
    <span className="text-lg text-orange-500 font-bold">
      ⭐{score}
    </span>
  </div>
</div>
```

현재 진행 상황과 점수가 항상 보임.

---

다음 글에서 LocalStorage 저장.

[#8 - LocalStorage 저장](/posts/math-champion/math-champion-08/)
