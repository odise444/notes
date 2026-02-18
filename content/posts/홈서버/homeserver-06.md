---
title: "오드로이드 N2+ 홈서버 구축기 #6 - 매매 로직 구현"
date: 2026-02-22
tags: ["Odroid", "자동매매", "Python", "트레이딩"]
categories: ["홈서버"]
series: ["오드로이드 N2+ 홈서버 구축기"]
summary: "전략 구조 잡고 에러 처리까지"
---

## 전략 구조

매매 전략은 교체 가능하게 만든다. 기본 클래스 정의하고 상속받아서 구현.

### strategy/base.py

```python
from abc import ABC, abstractmethod

class BaseStrategy(ABC):
    """전략 기본 클래스"""
    
    @abstractmethod
    def decide(self, data) -> str:
        """
        매매 신호 반환
        Returns: "BUY", "SELL", "HOLD"
        """
        pass
    
    @abstractmethod
    def get_name(self) -> str:
        """전략 이름"""
        pass
```

### 전략 예시

실제 전략 코드를 공개하진 않겠다. 대신 구조만:

```python
from strategy.base import BaseStrategy

class MyStrategy(BaseStrategy):
    def __init__(self):
        self.position = None  # 현재 포지션
        self.buy_price = 0    # 매수가
    
    def get_name(self):
        return "MyStrategy"
    
    def decide(self, data):
        price = data['price']
        # ... 여기에 판단 로직 ...
        
        if 매수_조건:
            return "BUY"
        elif 매도_조건:
            return "SELL"
        else:
            return "HOLD"
```

전략이 단순하든 복잡하든 인터페이스는 같다. `decide()` 호출하면 "BUY", "SELL", "HOLD" 중 하나 반환.

## 시세 데이터 수집

pyupbit으로 여러 종류 데이터 가져올 수 있다:

```python
import pyupbit

# 현재가
price = pyupbit.get_current_price("KRW-BTC")

# 호가창
orderbook = pyupbit.get_orderbook("KRW-BTC")

# OHLCV (캔들)
df = pyupbit.get_ohlcv("KRW-BTC", interval="minute1", count=200)
```

캔들 데이터 쓰면 이동평균, RSI 같은 기술적 지표 계산 가능:

```python
import pandas as pd

df = pyupbit.get_ohlcv("KRW-BTC", interval="minute1", count=200)

# 이동평균
df['ma5'] = df['close'].rolling(5).mean()
df['ma20'] = df['close'].rolling(20).mean()

# 최근 값
current_price = df['close'].iloc[-1]
ma5 = df['ma5'].iloc[-1]
ma20 = df['ma20'].iloc[-1]
```

## 주문 실행

### 시장가 주문

제일 간단하다. 바로 체결됨:

```python
# 매수 (금액 기준)
upbit.buy_market_order("KRW-BTC", 10000)  # 1만원어치

# 매도 (수량 기준)
upbit.sell_market_order("KRW-BTC", 0.001)  # 0.001 BTC
```

### 지정가 주문

원하는 가격에 걸어둔다:

```python
# 지정가 매수
upbit.buy_limit_order("KRW-BTC", 50000000, 0.001)  # 5천만원에 0.001 BTC

# 지정가 매도
upbit.sell_limit_order("KRW-BTC", 60000000, 0.001)  # 6천만원에 0.001 BTC
```

자동매매에선 시장가가 편하다. 체결 안 될 걱정 없으니까.

## 에러 처리

API 호출은 실패할 수 있다. 네트워크 문제, 서버 점검, 잔고 부족 등.

```python
import time
from utils.logger import get_logger

logger = get_logger()

class Trader:
    def execute_order(self, order_type, ticker, amount):
        max_retries = 3
        
        for attempt in range(max_retries):
            try:
                if order_type == "BUY":
                    result = self.api.buy(ticker, amount)
                else:
                    result = self.api.sell(ticker)
                
                logger.info(f"주문 성공: {result}")
                return result
                
            except Exception as e:
                logger.warning(f"주문 실패 (시도 {attempt + 1}): {e}")
                
                if attempt < max_retries - 1:
                    time.sleep(5)  # 5초 후 재시도
                else:
                    logger.error(f"주문 최종 실패: {e}")
                    raise
```

재시도 로직 넣어두면 일시적인 오류는 넘어간다.

## 자주 발생하는 에러

### 잔고 부족

```python
def buy(self, ticker, amount):
    balance = self.api.get_balance("KRW")
    
    if balance < amount:
        logger.warning(f"잔고 부족: {balance} < {amount}")
        return None
    
    return self.api.buy(ticker, amount)
```

### 최소 주문 금액

업비트는 최소 주문 금액이 있다 (보통 5000원):

```python
MIN_ORDER_AMOUNT = 5000

def buy(self, ticker, amount):
    if amount < MIN_ORDER_AMOUNT:
        logger.warning(f"최소 주문 금액 미달: {amount}")
        return None
    
    return self.api.buy(ticker, amount)
```

### API 호출 제한

업비트 API는 초당 호출 제한이 있다:
- 주문: 초당 8회
- 조회: 초당 10회

너무 빠르게 호출하면 차단당한다:

```python
import time

class RateLimiter:
    def __init__(self, calls_per_second=5):
        self.min_interval = 1.0 / calls_per_second
        self.last_call = 0
    
    def wait(self):
        now = time.time()
        elapsed = now - self.last_call
        
        if elapsed < self.min_interval:
            time.sleep(self.min_interval - elapsed)
        
        self.last_call = time.time()
```

## 백테스트 vs 실거래

처음부터 실거래 돌리면 안 된다. 무조건 백테스트 먼저.

```python
class Trader:
    def __init__(self, live=False):
        self.live = live
    
    def execute_order(self, order_type, ticker, amount):
        if self.live:
            # 실제 주문
            return self.api.buy(ticker, amount)
        else:
            # 가상 주문 (로그만)
            logger.info(f"[백테스트] {order_type} {ticker} {amount}")
            return {"status": "simulated"}
```

`live=False`로 한동안 돌려보면서 전략 검증하고, 괜찮으면 그때 `live=True`.

## 다음 편에서

마지막이다. 텔레그램 알림 연동하고, 서버 자동 시작 설정, 한 달 운영 후기.
