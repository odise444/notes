---
title: "수학 챔피언 개발기 #11 - 통계 화면"
date: 2025-12-20
tags:
  - React
  - 통계
  - 데이터시각화
categories:
  - 개발기
series:
  - 수학 챔피언 개발기
summary: 얼마나 풀었나, 정답률은 몇 %인가.
---

숫자로 보여주면 성취감이 커진다.

"100문제나 풀었어!"

---

## 통계 데이터

```jsx
const stats = {
  totalGames: 15,           // 총 게임 수
  totalQuestions: 150,      // 총 문제 수
  correctAnswers: 135,      // 맞춘 문제
  bestScore: 180,           // 최고 점수
  bestStreak: 12,           // 최고 연속 정답
  byDifficulty: {           // 난이도별
    easy: { games: 10, correct: 90, total: 100 },
    medium: { games: 5, correct: 45, total: 50 }
  }
};
```

---

## 통계 업데이트

게임 끝날 때:

```jsx
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
```

---

## 통계 카드

```jsx
<div className="grid grid-cols-2 gap-2">
  <div className="bg-gradient-to-br from-blue-50 to-indigo-100 p-3 rounded-xl text-center">
    <div className="text-3xl font-bold text-indigo-600">
      {stats.totalGames}
    </div>
    <div className="text-xs text-gray-500">총 게임</div>
  </div>
  
  <div className="bg-gradient-to-br from-green-50 to-emerald-100 p-3 rounded-xl text-center">
    <div className="text-3xl font-bold text-emerald-600">
      {stats.totalQuestions > 0 
        ? Math.round((stats.correctAnswers / stats.totalQuestions) * 100) 
        : 0}%
    </div>
    <div className="text-xs text-gray-500">정답률</div>
  </div>
  
  <div className="bg-gradient-to-br from-amber-50 to-orange-100 p-3 rounded-xl text-center">
    <div className="text-3xl font-bold text-orange-600">
      {stats.bestScore}
    </div>
    <div className="text-xs text-gray-500">최고 점수</div>
  </div>
  
  <div className="bg-gradient-to-br from-rose-50 to-pink-100 p-3 rounded-xl text-center">
    <div className="text-3xl font-bold text-rose-600">
      {stats.bestStreak}🔥
    </div>
    <div className="text-xs text-gray-500">최고 연속</div>
  </div>
</div>
```

---

## 상세 정보

```jsx
<div className="bg-gray-50 p-3 rounded-xl space-y-2">
  <div className="flex justify-between text-sm">
    <span className="text-gray-500">총 문제 수</span>
    <span className="font-bold">{stats.totalQuestions}문제</span>
  </div>
  <div className="flex justify-between text-sm">
    <span className="text-gray-500">맞춘 문제</span>
    <span className="font-bold text-emerald-600">{stats.correctAnswers}문제</span>
  </div>
  <div className="flex justify-between text-sm">
    <span className="text-gray-500">틀린 문제</span>
    <span className="font-bold text-rose-500">
      {stats.totalQuestions - stats.correctAnswers}문제
    </span>
  </div>
</div>
```

---

## 난이도별 정답률

```jsx
{Object.keys(stats.byDifficulty).length > 0 && (
  <div className="space-y-2">
    <div className="text-sm font-bold text-gray-600">난이도별 정답률</div>
    {Object.entries(stats.byDifficulty).map(([diff, data]) => (
      <div key={diff} className="bg-gray-50 p-2 rounded-lg">
        <div className="flex justify-between items-center text-sm">
          <span>{DIFFICULTY_SETTINGS[diff]?.label || diff}</span>
          <span className="font-bold">
            {data.total > 0 
              ? Math.round((data.correct / data.total) * 100) 
              : 0}%
          </span>
        </div>
        
        {/* 프로그레스 바 */}
        <div className="h-2 bg-gray-200 rounded-full mt-1 overflow-hidden">
          <div 
            className="h-full bg-gradient-to-r from-indigo-400 to-purple-500" 
            style={{ 
              width: `${data.total > 0 ? (data.correct / data.total) * 100 : 0}%` 
            }} 
          />
        </div>
        
        <div className="text-xs text-gray-400 mt-1">
          {data.games}게임 · {data.correct}/{data.total}문제
        </div>
      </div>
    ))}
  </div>
)}
```

어떤 난이도에서 많이 틀리는지 한눈에.

---

## 성장 확인

"어제보다 정답률 올랐어!"

숫자로 보여주니까 아이도 성장 느낌.

---

다음 글에서 애니메이션.

[#12 - 애니메이션](/posts/math-champion/math-champion-12/)
