---
title: "수학 챔피언 개발기 #6 - 타이머 구현"
date: 2024-12-20
tags: ["React", "타이머", "useEffect", "useRef"]
categories: ["개발기"]
series: ["수학 챔피언 개발기"]
summary: "시간 압박으로 긴장감 주기."
---

타이머 있으면 긴장감 생긴다.

근데 너무 짧으면 스트레스. 적당히.

---

## 타이머 설정

```jsx
const [timerSetting, setTimerSetting] = useState(10);
const [timeLeft, setTimeLeft] = useState(10);
const timerRef = useRef(null);
```

- `timerSetting`: 사용자가 선택한 시간 (끄기/3초/5초/10초/15초)
- `timeLeft`: 남은 시간
- `timerRef`: setInterval 참조

---

## 타이머 시작

문제 나오면 타이머 시작:

```jsx
useEffect(() => {
  if (screen === 'game' && feedback === null && problem) {
    setTimeLeft(timerSetting);
    
    if (timerSetting === 0) return; // 타이머 끄기
    
    timerRef.current = setInterval(() => {
      setTimeLeft(prev => {
        if (prev <= 1) {
          clearInterval(timerRef.current);
          return 0;
        }
        return prev - 1;
      });
    }, 1000);
    
    return () => clearInterval(timerRef.current);
  }
}, [screen, problem, feedback, timerSetting]);
```

---

## 시간 초과 처리

```jsx
useEffect(() => {
  if (timeLeft === 0 && feedback === null && screen === 'game' && timerSetting > 0) {
    handleTimeout();
  }
}, [timeLeft]);

const handleTimeout = () => {
  if (feedback !== null || !problem) return;
  
  // 오답 노트에 추가
  addWrongAnswer(
    problem.num1, problem.num2, problem.op, 
    null, problem.answer, '시간초과'
  );
  
  setStreak(0);
  setFeedback({
    correct: false,
    message: `시간 초과! 정답: ${problem.answer}`,
    emoji: '⏰'
  });
  
  proceedToNext(false);
};
```

---

## 정답 제출 시 타이머 멈춤

```jsx
const checkAnswer = () => {
  if (!userAnswer || feedback !== null) return;
  
  clearInterval(timerRef.current);  // 타이머 멈춤
  
  // ... 정답 체크 로직
};
```

---

## 타이머 UI

```jsx
{timerSetting === 0 ? (
  <div className="text-2xl text-gray-300">⏱️ 끄기</div>
) : (
  <>
    <div className={`text-3xl font-bold ${
      timeLeft <= 1 ? 'text-red-500 animate-pulse' : 
      timeLeft <= 2 ? 'text-orange-500' : 
      'text-gray-400'
    }`}>
      {feedback === null ? `⏱️ ${timeLeft}` : '⏸️'}
    </div>
    
    {/* 프로그레스 바 */}
    {feedback === null && (
      <div className="w-32 h-2 bg-gray-200 rounded-full overflow-hidden">
        <div 
          className={`h-full transition-all duration-1000 ease-linear ${
            timeLeft <= 1 ? 'bg-red-500' : 
            timeLeft <= 2 ? 'bg-orange-400' : 
            'bg-emerald-400'
          }`}
          style={{ width: `${(timeLeft / timerSetting) * 100}%` }} 
        />
      </div>
    )}
  </>
)}
```

- 시간 적으면 빨간색 + 깜빡임
- 프로그레스 바로 시각화

---

## 타이머 선택 UI

홈 화면에서:

```jsx
<div className="grid grid-cols-5 gap-1">
  {[
    {v: 0, l: '끄기'}, 
    {v: 3, l: '3초'}, 
    {v: 5, l: '5초'}, 
    {v: 10, l: '10초'}, 
    {v: 15, l: '15초'}
  ].map(({v, l}) => (
    <button 
      key={v} 
      onClick={() => setTimerSetting(v)}
      className={`py-2 rounded-lg text-xs font-medium btn-3d ${
        timerSetting === v 
          ? 'bg-cyan-400 text-white' 
          : 'bg-gray-100 text-gray-600'
      }`}
    >
      {l}
    </button>
  ))}
</div>
```

처음엔 10초로. 익숙해지면 줄이기.

---

다음 글에서 점수 시스템.

[#7 - 점수 시스템](/posts/math-champion/math-champion-07/)
