---
title: "오드로이드 N2+ 홈서버 구축기 #5 - 자동매매 시스템 구조"
date: 2026-02-21
tags: ["Odroid", "자동매매", "업비트", "Python"]
categories: ["홈서버"]
series: ["오드로이드 N2+ 홈서버 구축기"]
summary: "업비트 API 연동하고 프로젝트 구조 잡기"
---

## 전체 아키텍처

자동매매 시스템 구조는 단순하게 잡았다:

```
┌─────────────────────────────────────────┐
│              Auto Trader                 │
├─────────────────────────────────────────┤
│  main.py          - 진입점              │
│  trader.py        - 매매 로직           │
│  api/upbit.py     - 업비트 API 래퍼     │
│  strategy/        - 전략 모듈           │
│  utils/           - 유틸리티            │
│  config/          - 설정 파일           │
└─────────────────────────────────────────┘
          │
          ▼
┌─────────────────┐
│   업비트 API     │
└─────────────────┘
```

복잡하게 만들 필요 없다. 돌아가면 장땡이다.

## 업비트 API

업비트는 REST API 제공한다. 인증 방식이 좀 독특한데, JWT 토큰 쓴다.

### API 키 발급

1. 업비트 로그인
2. 마이페이지 > Open API 관리
3. API 키 발급

주의할 점:
- **출금 권한은 절대 주지 마라**. 해킹당하면 끝.
- IP 제한 걸어두면 더 안전

### pyupbit 라이브러리

직접 API 호출해도 되는데, pyupbit 쓰면 편하다:

```bash
pip install pyupbit
```

기본 사용법:

```python
import pyupbit

# 인증
upbit = pyupbit.Upbit(access_key, secret_key)

# 잔고 조회
balances = upbit.get_balances()

# 현재가 조회
price = pyupbit.get_current_price("KRW-BTC")

# 매수 (시장가)
upbit.buy_market_order("KRW-BTC", 10000)  # 1만원어치

# 매도 (시장가)
upbit.sell_market_order("KRW-BTC", 0.001)  # 0.001 BTC
```

간단하다.

## 디렉토리 구조

```
auto-trader/
├── main.py
├── trader.py
├── requirements.txt
├── Dockerfile
├── docker-compose.yml
├── .env.example
├── api/
│   ├── __init__.py
│   └── upbit.py
├── strategy/
│   ├── __init__.py
│   └── base.py
├── utils/
│   ├── __init__.py
│   ├── logger.py
│   └── notifier.py
├── config/
│   └── settings.py
└── logs/
```

## 설정 파일 관리

API 키 같은 민감한 정보는 코드에 넣으면 안 된다. `.env` 파일로 분리:

```bash
# .env
UPBIT_ACCESS_KEY=your_access_key
UPBIT_SECRET_KEY=your_secret_key
TELEGRAM_TOKEN=your_bot_token
TELEGRAM_CHAT_ID=your_chat_id
```

`.env`는 `.gitignore`에 추가해서 실수로 커밋 안 되게.

파이썬에서 불러올 때:

```python
import os
from dotenv import load_dotenv

load_dotenv()

ACCESS_KEY = os.getenv('UPBIT_ACCESS_KEY')
SECRET_KEY = os.getenv('UPBIT_SECRET_KEY')
```

python-dotenv 설치 필요:

```bash
pip install python-dotenv
```

## requirements.txt

```
pyupbit==0.2.33
python-dotenv==1.0.0
requests==2.31.0
schedule==1.2.0
python-telegram-bot==20.7
```

버전 명시해두는 게 좋다. 나중에 재설치할 때 동일한 환경 보장.

## 기본 구조 코드

### main.py

```python
import time
import schedule
from trader import Trader
from utils.logger import setup_logger

logger = setup_logger()

def main():
    trader = Trader()
    
    # 1분마다 체크
    schedule.every(1).minutes.do(trader.run)
    
    logger.info("자동매매 시작")
    
    while True:
        schedule.run_pending()
        time.sleep(1)

if __name__ == "__main__":
    main()
```

### trader.py

```python
from api.upbit import UpbitAPI
from strategy.base import BaseStrategy
from utils.logger import get_logger

logger = get_logger()

class Trader:
    def __init__(self):
        self.api = UpbitAPI()
        self.strategy = BaseStrategy()
    
    def run(self):
        try:
            # 시세 조회
            price = self.api.get_current_price("KRW-BTC")
            
            # 전략 판단
            signal = self.strategy.decide(price)
            
            # 매매 실행
            if signal == "BUY":
                self.api.buy("KRW-BTC", amount=10000)
            elif signal == "SELL":
                self.api.sell("KRW-BTC")
                
        except Exception as e:
            logger.error(f"에러 발생: {e}")
```

### api/upbit.py

```python
import pyupbit
from config.settings import ACCESS_KEY, SECRET_KEY

class UpbitAPI:
    def __init__(self):
        self.client = pyupbit.Upbit(ACCESS_KEY, SECRET_KEY)
    
    def get_current_price(self, ticker):
        return pyupbit.get_current_price(ticker)
    
    def get_balance(self, currency="KRW"):
        return self.client.get_balance(currency)
    
    def buy(self, ticker, amount):
        return self.client.buy_market_order(ticker, amount)
    
    def sell(self, ticker, volume=None):
        if volume is None:
            # 전량 매도
            currency = ticker.split("-")[1]
            volume = self.client.get_balance(currency)
        return self.client.sell_market_order(ticker, volume)
```

실제 전략 로직은 `strategy/` 폴더에 따로 구현한다. 여기선 구조만.

## 다음 편에서

매매 로직 구현한다. 전략 구조 잡고, 에러 처리하는 법.
