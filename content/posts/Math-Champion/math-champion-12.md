---
title: "수학 챔피언 개발기 #12 - 애니메이션"
date: 2024-12-20
tags: ["React", "CSS", "애니메이션"]
categories: ["개발기"]
series: ["수학 챔피언 개발기"]
summary: "정답이면 빵빠레, 오답이면 흔들흔들."
---

애니메이션이 재미를 더한다.

---

## 정답 축하 - Confetti

별, 이모지가 떨어지는 효과:

```css
@keyframes fall {
  0% { 
    transform: translateY(-20px) rotate(0deg); 
    opacity: 1; 
  }
  100% { 
    transform: translateY(100vh) rotate(720deg); 
    opacity: 0; 
  }
}
.animate-fall { 
  animation: fall 2s ease-out forwards; 
}
```

```jsx
const Confetti = () => (
  <div className="fixed inset-0 pointer-events-none overflow-hidden z-50">
    {[...Array(15)].map((_, i) => (
      <div 
        key={i} 
        className="absolute animate-fall"
        style={{ 
          left: `${Math.random() * 100}%`, 
          top: '-20px',
          animationDelay: `${Math.random() * 0.5}s`, 
          fontSize: `${Math.random() * 16 + 16}px` 
        }}
      >
        {EMOJIS[Math.floor(Math.random() * EMOJIS.length)]}
      </div>
    ))}
  </div>
);
```

정답 맞추면 1초간 이모지 비 내림.

---

## 정답 피드백 - Pop

```css
@keyframes pop {
  0% { transform: scale(0); }
  50% { transform: scale(1.2); }
  100% { transform: scale(1); }
}
.animate-pop { 
  animation: pop 0.3s ease-out forwards; 
}
```

피드백 카드가 "팡!" 하고 나타남.

---

## 오답 피드백 - Shake

```css
@keyframes shake {
  0%, 100% { transform: translateX(0); }
  25% { transform: translateX(-5px); }
  75% { transform: translateX(5px); }
}
.animate-shake { 
  animation: shake 0.3s ease-out; 
}
```

틀리면 카드가 좌우로 흔들림.

---

## 둥실둥실 - Float

홈 화면 아이콘:

```css
@keyframes float {
  0%, 100% { transform: translateY(0px); }
  50% { transform: translateY(-8px); }
}
.animate-float { 
  animation: float 2s ease-in-out infinite; 
}
```

```jsx
<div className="text-5xl mb-2 animate-float">🧮</div>
```

---

## 버튼 글로우

시작 버튼 주목:

```css
@keyframes pulse-glow {
  0%, 100% { 
    box-shadow: 0 0 15px rgba(251, 191, 36, 0.4); 
  }
  50% { 
    box-shadow: 0 0 30px rgba(251, 191, 36, 0.8); 
  }
}
.glow { 
  animation: pulse-glow 2s ease-in-out infinite; 
}
```

```jsx
<button className="... glow">
  🚀 시작하기!
</button>
```

---

## 불꽃 효과

연속 정답 표시:

```css
.streak-fire {
  text-shadow: 0 0 8px #ff6b35, 0 0 16px #ff6b35;
}
```

```jsx
{streak >= 3 && (
  <span className="text-lg streak-fire animate-pulse">
    🔥{streak}
  </span>
)}
```

Tailwind `animate-pulse`와 조합.

---

## 프로그레스 바 전환

부드럽게:

```jsx
<div 
  className="h-full bg-gradient-to-r from-orange-400 to-rose-400 transition-all duration-300"
  style={{ width: `${(questionCount / totalQuestions) * 100}%` }} 
/>
```

`transition-all duration-300`으로 스무스하게.

---

## 3D 버튼

눌림 효과:

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

실제로 누르는 느낌.

---

아이들은 시각적 피드백을 좋아함. 과하지 않게 적당히.

다음 글에서 모바일 최적화.

[#13 - 모바일 최적화](/posts/math-champion/math-champion-13/)
