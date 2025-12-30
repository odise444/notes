---
title: "수학 챔피언 개발기 #3 - 기본 구조 잡기"
date: 2024-12-20
tags: ["React", "상태관리", "useState"]
categories: ["개발기"]
series: ["수학 챔피언 개발기"]
summary: "화면 전환과 상태 관리."
---

화면이 여러 개 필요하다.

- 홈
- 게임
- 결과
- 리더보드
- 통계
- 오답 노트

---

## 화면 전환

복잡한 라우터 필요 없음. useState로 충분.

```jsx
const [screen, setScreen] = useState('home');

return (
  <div>
    {screen === 'home' && <HomeScreen />}
    {screen === 'game' && <GameScreen />}
    {screen === 'result' && <ResultScreen />}
    {screen === 'leaderboard' && <LeaderboardScreen />}
    {screen === 'stats' && <StatsScreen />}
    {screen === 'wrong' && <WrongAnswersScreen />}
  </div>
);
```

---

## 상태 목록

게임에 필요한 상태들:

```jsx
// 설정
const [playerName, setPlayerName] = useState('');
const [difficulty, setDifficulty] = useState('easy');
const [operation, setOperation] = useState('addsub');
const [totalQuestions, setTotalQuestions] = useState(10);
const [timerSetting, setTimerSetting] = useState(10);

// 게임 진행
const [problem, setProblem] = useState(null);
const [userAnswer, setUserAnswer] = useState('');
const [score, setScore] = useState(0);
const [streak, setStreak] = useState(0);
const [questionCount, setQuestionCount] = useState(0);
const [feedback, setFeedback] = useState(null);
const [timeLeft, setTimeLeft] = useState(10);

// 데이터
const [leaderboard, setLeaderboard] = useState([]);
const [stats, setStats] = useState({});
const [wrongAnswers, setWrongAnswers] = useState([]);
```

---

## 컴포넌트 분리? 안 함

파일 하나에 다 넣을 거라 컴포넌트 분리 안 함.

대신 화면별로 JSX 블록 구분:

```jsx
function MathGame() {
  // ... 모든 상태와 로직 ...

  return (
    <div>
      {/* HOME */}
      {screen === 'home' && (
        <div className="...">
          {/* 홈 화면 내용 */}
        </div>
      )}

      {/* GAME */}
      {screen === 'game' && (
        <div className="...">
          {/* 게임 화면 내용 */}
        </div>
      )}

      {/* ... 다른 화면들 ... */}
    </div>
  );
}
```

---

## 레이아웃

모바일 중심 디자인:

```jsx
<div className="min-h-screen bg-gradient-to-br from-amber-100 via-orange-100 to-rose-100 p-3">
  <div className="max-w-md mx-auto">
    {/* 카드 형태 */}
    <div className="bg-white/90 backdrop-blur rounded-2xl shadow-xl p-4">
      {/* 내용 */}
    </div>
  </div>
</div>
```

- 따뜻한 그라디언트 배경
- 최대 너비 제한 (max-w-md)
- 카드에 그림자와 블러

---

## 버튼 스타일

3D 느낌 버튼:

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

누르면 눌리는 느낌. 아이들 좋아함.

---

다음 글에서 문제 생성 로직.

[#4 - 문제 생성 로직](/posts/math-champion/math-champion-04/)
