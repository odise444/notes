---
title: "수학 챔피언 개발기 #9 - 오답 노트"
date: 2024-12-20
tags: ["React", "오답노트", "학습"]
categories: ["개발기"]
series: ["수학 챔피언 개발기"]
summary: "틀린 문제를 모아서 복습."
---

틀린 문제를 그냥 넘기면 안 된다.

오답 노트에 모아서 나중에 다시 보자.

---

## 오답 데이터 구조

```jsx
const wrongAnswer = {
  id: Date.now(),
  num1: 7,
  num2: 8,
  op: '+',
  userAnswer: 14,      // 내가 쓴 답
  correctAnswer: 15,   // 정답
  reason: '오답',       // 또는 '시간초과'
  difficulty: 'easy',
  date: '2024-12-20T10:30:00',
  reviewed: false      // 복습 여부
};
```

---

## 오답 추가

틀렸거나 시간 초과일 때:

```jsx
const addWrongAnswer = (num1, num2, op, userAns, correctAns, reason) => {
  const newEntry = {
    id: Date.now(),
    num1, num2, op,
    userAnswer: userAns,
    correctAnswer: correctAns,
    reason,
    difficulty,
    date: new Date().toISOString(),
    reviewed: false
  };
  
  // 최대 100개까지만 저장
  const updated = [newEntry, ...wrongAnswers].slice(0, 100);
  saveWrongAnswers(updated);
};
```

---

## 오답 호출

```jsx
// 오답일 때
if (!isCorrect) {
  addWrongAnswer(
    problem.num1, problem.num2, problem.op, 
    parseInt(userAnswer), problem.answer, 
    '오답'
  );
}

// 시간 초과일 때
const handleTimeout = () => {
  addWrongAnswer(
    problem.num1, problem.num2, problem.op, 
    null, problem.answer, 
    '시간초과'
  );
};
```

---

## 오답 노트 화면

```jsx
{screen === 'wrong' && (
  <div className="bg-white/90 backdrop-blur rounded-2xl shadow-xl p-4 space-y-4">
    <div className="text-center">
      <div className="text-4xl mb-1">📝</div>
      <h2 className="text-2xl text-rose-600">오답 노트</h2>
      <p className="text-sm text-gray-500">
        {wrongAnswers.length}개 
        ({wrongAnswers.filter(w => !w.reviewed).length}개 미복습)
      </p>
    </div>
    
    {wrongAnswers.length === 0 ? (
      <div className="text-center py-10 text-gray-500">
        틀린 문제가 없어요!<br/>대단해요! 🌟
      </div>
    ) : (
      <div className="space-y-2 max-h-80 overflow-y-auto">
        {wrongAnswers.map((item) => (
          <WrongAnswerCard key={item.id} item={item} />
        ))}
      </div>
    )}
  </div>
)}
```

---

## 오답 카드

```jsx
<div className={`p-3 rounded-xl border-2 ${
  item.reviewed 
    ? 'bg-gray-50 border-gray-200' 
    : 'bg-rose-50 border-rose-200'
}`}>
  <div className="flex items-start justify-between gap-2">
    <div className="flex-1">
      {/* 세로 계산식 */}
      <div className="bg-white rounded-lg p-2 inline-block font-mono text-right text-lg">
        <div>{String(item.num1).padStart(4, '\u00A0')}</div>
        <div>
          <span className="text-orange-500">{item.op}</span>
          {String(item.num2).padStart(3, '\u00A0')}
        </div>
        <div className="border-t-2 border-gray-300 pt-1">
          <span className="text-red-500 line-through mr-2">
            {item.userAnswer !== null ? item.userAnswer : '?'}
          </span>
          <span className="text-emerald-600 font-bold">
            {item.correctAnswer}
          </span>
        </div>
      </div>
      <div className="text-xs text-gray-400 mt-1">
        {item.reason} · {DIFFICULTY_SETTINGS[item.difficulty]?.label}
        {item.reviewed && ' · ✓ 복습완료'}
      </div>
    </div>
    
    {/* 버튼들 */}
    <div className="flex flex-col gap-1">
      {!item.reviewed && (
        <button onClick={() => markAsReviewed(item.id)} 
          className="px-2 py-1 bg-emerald-100 text-emerald-600 text-xs rounded-lg">
          ✓ 복습
        </button>
      )}
      <button onClick={() => removeFromWrongAnswers(item.id)} 
        className="px-2 py-1 bg-gray-100 text-gray-500 text-xs rounded-lg">
        삭제
      </button>
    </div>
  </div>
</div>
```

틀린 답은 취소선, 정답은 초록색으로.

---

## 복습 표시

```jsx
const markAsReviewed = (id) => {
  const updated = wrongAnswers.map(w => 
    w.id === id ? { ...w, reviewed: true } : w
  );
  saveWrongAnswers(updated);
};
```

---

## 미복습 알림

홈 화면에서 뱃지로 표시:

```jsx
<button onClick={() => setScreen('wrong')} className="... relative">
  📝 오답 노트
  {wrongAnswers.filter(w => !w.reviewed).length > 0 && (
    <span className="absolute -top-2 -right-2 bg-red-500 text-white text-xs w-6 h-6 rounded-full flex items-center justify-center">
      {wrongAnswers.filter(w => !w.reviewed).length}
    </span>
  )}
</button>
```

"복습 안 한 게 3개 있어!" 느낌.

---

다음 글에서 리더보드.

[#10 - 리더보드](/posts/math-champion/math-champion-10/)
