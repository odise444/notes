---
title: "수학 챔피언 개발기 #10 - 리더보드"
date: 2025-12-14
tags:
  - React
  - 리더보드
  - 순위
categories:
  - 개발기
series:
  - 수학 챔피언 개발기
summary: 가족끼리 점수 경쟁.
---

형제끼리, 부모랑 경쟁하면 더 재미있다.

리더보드로 순위 보여주자.

---

## 리더보드 데이터

```jsx
const leaderboardEntry = {
  name: '지수',
  score: 150,
  difficulty: 'easy',
  operation: 'addsub',
  date: '2024-12-20T10:30:00',
  id: 1703054200000
};
```

---

## 점수 저장

게임 끝날 때:

```jsx
const newEntry = {
  name: playerName,
  score: finalScore,
  difficulty,
  operation,
  date: new Date().toISOString(),
  id: Date.now()
};

const updated = [...leaderboard, newEntry]
  .sort((a, b) => b.score - a.score)  // 높은 점수 순
  .slice(0, 20);  // 상위 20개만

saveLeaderboard(updated);
```

---

## 리더보드 화면

```jsx
{screen === 'leaderboard' && (
  <div className="bg-white/90 backdrop-blur rounded-2xl shadow-xl p-4 space-y-4">
    <div className="text-center">
      <div className="text-4xl mb-1">🏆</div>
      <h2 className="text-2xl text-orange-600">리더보드</h2>
    </div>
    
    {leaderboard.length === 0 ? (
      <div className="text-center py-10 text-gray-500">
        아직 기록이 없어요!<br/>
        첫 번째 챔피언이 되어보세요! 🌟
      </div>
    ) : (
      <div className="space-y-2 max-h-80 overflow-y-auto">
        {leaderboard.map((entry, idx) => (
          <LeaderboardRow key={entry.id} entry={entry} rank={idx} />
        ))}
      </div>
    )}
  </div>
)}
```

---

## 순위 행

```jsx
<div className={`flex items-center gap-3 p-3 rounded-lg ${
  idx === 0 
    ? 'bg-gradient-to-r from-yellow-100 to-amber-100 border-2 border-yellow-300' 
  : idx === 1 
    ? 'bg-gradient-to-r from-gray-100 to-slate-100 border border-gray-300' 
  : idx === 2 
    ? 'bg-gradient-to-r from-orange-50 to-amber-50 border border-orange-200' 
  : 'bg-gray-50'
}`}>
  {/* 순위 */}
  <div className="text-2xl w-8 text-center">
    {idx === 0 ? '🥇' : idx === 1 ? '🥈' : idx === 2 ? '🥉' : (
      <span className="text-gray-400 text-lg">{idx + 1}</span>
    )}
  </div>
  
  {/* 이름 & 난이도 */}
  <div className="flex-1 min-w-0">
    <div className="font-bold truncate">{entry.name}</div>
    <div className="text-xs text-gray-400">
      {DIFFICULTY_SETTINGS[entry.difficulty]?.label}
    </div>
  </div>
  
  {/* 점수 */}
  <div className="text-xl font-bold text-orange-500">
    {entry.score}
  </div>
</div>
```

- 1등: 금색 배경, 금메달
- 2등: 은색 배경, 은메달
- 3등: 동색 배경, 동메달
- 나머지: 숫자만

---

## 초기화 버튼

```jsx
{leaderboard.length > 0 && (
  <button onClick={resetLeaderboard} 
    className="w-full py-1.5 text-gray-300 text-xs">
    리더보드 초기화
  </button>
)}

const resetLeaderboard = () => {
  if (confirm('정말 리더보드를 초기화할까요?')) {
    setLeaderboard([]);
    storage.remove('math-leaderboard');
  }
};
```

---

## 가족 경쟁

같은 폰에서 이름만 바꿔가며 플레이.

```
🥇 아빠  180
🥈 지수  150
🥉 엄마  140
4  지호  120
```

"아빠 이기자!" 동기 부여.

---

다음 글에서 통계 화면.

[#11 - 통계 화면](/posts/math-champion/math-champion-11/)
