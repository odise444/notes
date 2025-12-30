---
title: "수학 챔피언 개발기 #8 - LocalStorage 저장"
date: 2024-12-20
tags: ["React", "LocalStorage", "데이터"]
categories: ["개발기"]
series: ["수학 챔피언 개발기"]
summary: "서버 없이 브라우저에 데이터 저장."
---

서버 만들기 귀찮다.

LocalStorage로 충분.

---

## Storage 래퍼

try-catch로 에러 처리:

```jsx
const storage = {
  get: (key) => {
    try {
      const item = localStorage.getItem(key);
      return item ? JSON.parse(item) : null;
    } catch (e) {
      return null;
    }
  },
  set: (key, value) => {
    try {
      localStorage.setItem(key, JSON.stringify(value));
    } catch (e) {
      console.error('Storage error:', e);
    }
  },
  remove: (key) => {
    try {
      localStorage.removeItem(key);
    } catch (e) {
      console.error('Storage error:', e);
    }
  }
};
```

---

## 저장하는 데이터

```jsx
// 플레이어 이름
storage.set('math-player-name', playerName);

// 리더보드
storage.set('math-leaderboard', leaderboard);

// 통계
storage.set('math-stats', stats);

// 오답 노트
storage.set('math-wrong-answers', wrongAnswers);
```

---

## 초기 로드

앱 시작할 때 불러오기:

```jsx
const [isLoading, setIsLoading] = useState(true);

useEffect(() => {
  setPlayerName(storage.get('math-player-name') || '');
  setLeaderboard(storage.get('math-leaderboard') || []);
  setStats(storage.get('math-stats') || {
    totalGames: 0,
    totalQuestions: 0,
    correctAnswers: 0,
    bestScore: 0,
    bestStreak: 0,
    byDifficulty: {}
  });
  setWrongAnswers(storage.get('math-wrong-answers') || []);
  setIsLoading(false);
}, []);
```

---

## 로딩 화면

데이터 불러오는 동안:

```jsx
if (isLoading) {
  return (
    <div className="min-h-screen flex items-center justify-center bg-gradient-to-br from-amber-100 via-orange-100 to-rose-100">
      <div className="text-2xl animate-bounce">🔢 로딩중...</div>
    </div>
  );
}
```

---

## 이름 자동 저장

이름 바뀔 때마다 저장:

```jsx
useEffect(() => {
  if (playerName) {
    storage.set('math-player-name', playerName);
  }
}, [playerName]);
```

다음에 들어오면 이름 입력 안 해도 됨.

---

## 게임 끝날 때 저장

```jsx
// 통계 업데이트
const newStats = {
  totalGames: stats.totalGames + 1,
  totalQuestions: stats.totalQuestions + totalQuestions,
  correctAnswers: stats.correctAnswers + finalCorrect,
  bestScore: Math.max(stats.bestScore, finalScore),
  bestStreak: Math.max(stats.bestStreak, finalStreak),
  byDifficulty: {
    ...stats.byDifficulty,
    [difficulty]: {
      games: (stats.byDifficulty[difficulty]?.games || 0) + 1,
      correct: (stats.byDifficulty[difficulty]?.correct || 0) + finalCorrect,
      total: (stats.byDifficulty[difficulty]?.total || 0) + totalQuestions
    }
  }
};
saveStats(newStats);

// 리더보드 추가
const newEntry = {
  name: playerName,
  score: finalScore,
  difficulty,
  operation,
  date: new Date().toISOString(),
  id: Date.now()
};
const updated = [...leaderboard, newEntry]
  .sort((a, b) => b.score - a.score)
  .slice(0, 20);
saveLeaderboard(updated);
```

---

## 초기화 기능

데이터 지우는 버튼도 필요:

```jsx
const resetLeaderboard = () => {
  if (confirm('정말 리더보드를 초기화할까요?')) {
    setLeaderboard([]);
    storage.remove('math-leaderboard');
  }
};

const resetStats = () => {
  if (confirm('정말 통계를 초기화할까요?')) {
    const emptyStats = { /* ... */ };
    setStats(emptyStats);
    storage.remove('math-stats');
  }
};
```

확인 창 띄우고 진행.

---

다음 글에서 오답 노트.

[#9 - 오답 노트](/posts/math-champion/math-champion-09/)
