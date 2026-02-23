---
title: "오드로이드 N2+ 홈서버 구축기 #7 - 모니터링과 운영"
date: 2026-02-23
tags: ["Odroid", "자동매매", "텔레그램", "systemd"]
categories: ["홈서버"]
series: ["오드로이드 N2+ 홈서버 구축기"]
summary: "텔레그램 알림, 자동 시작, 운영 후기"
---

## 텔레그램 알림

24시간 돌아가는 봇이니까 상태 알림이 필요하다. 텔레그램이 제일 편하다.

### 봇 만들기

1. 텔레그램에서 @BotFather 검색
2. `/newbot` 명령
3. 봇 이름, 유저네임 입력
4. 토큰 받기

### Chat ID 확인

봇한테 아무 메시지나 보낸 후:

```
https://api.telegram.org/bot{토큰}/getUpdates
```

브라우저에서 열면 JSON 응답에 chat id 나온다.

### 알림 코드

```python
import requests

class TelegramNotifier:
    def __init__(self, token, chat_id):
        self.token = token
        self.chat_id = chat_id
        self.base_url = f"https://api.telegram.org/bot{token}"
    
    def send(self, message):
        url = f"{self.base_url}/sendMessage"
        data = {
            "chat_id": self.chat_id,
            "text": message,
            "parse_mode": "HTML"
        }
        
        try:
            response = requests.post(url, data=data)
            return response.json()
        except Exception as e:
            print(f"텔레그램 전송 실패: {e}")
            return None
```

### 언제 알림 보낼까

```python
# 매매 체결
notifier.send(f"🟢 매수 체결\n코인: BTC\n금액: 10,000원")
notifier.send(f"🔴 매도 체결\n코인: BTC\n수익: +5.2%")

# 에러 발생
notifier.send(f"⚠️ 에러 발생\n{error_message}")

# 일일 리포트
notifier.send(f"📊 일일 리포트\n수익률: +2.3%\n거래횟수: 5회")
```

과하게 보내면 알림 피로 생긴다. 중요한 것만.

## 로그 관리

### 로거 설정

```python
import logging
from logging.handlers import RotatingFileHandler

def setup_logger():
    logger = logging.getLogger('trader')
    logger.setLevel(logging.INFO)
    
    # 파일 핸들러 (10MB마다 로테이션, 최대 5개)
    file_handler = RotatingFileHandler(
        'logs/trader.log',
        maxBytes=10*1024*1024,
        backupCount=5
    )
    file_handler.setFormatter(
        logging.Formatter('%(asctime)s - %(levelname)s - %(message)s')
    )
    
    # 콘솔 핸들러
    console_handler = logging.StreamHandler()
    console_handler.setFormatter(
        logging.Formatter('%(asctime)s - %(message)s')
    )
    
    logger.addHandler(file_handler)
    logger.addHandler(console_handler)
    
    return logger
```

RotatingFileHandler 쓰면 로그 파일이 무한정 커지는 걸 방지한다.

### 로그 확인

```bash
# 최근 로그
tail -f logs/trader.log

# Docker 사용 시
docker logs -f auto-trader
```

## 시스템 자동 시작 (systemd)

서버 재부팅해도 자동으로 봇이 시작되어야 한다.

### Docker Compose 사용 시

`docker-compose.yml`에 `restart: unless-stopped` 넣어뒀으면 Docker 서비스만 자동 시작되면 된다:

```bash
sudo systemctl enable docker
```

### 직접 실행 시

systemd 서비스 파일 만든다:

```bash
sudo vim /etc/systemd/system/auto-trader.service
```

```ini
[Unit]
Description=Auto Trader Bot
After=network.target

[Service]
Type=simple
User=trader
WorkingDirectory=/home/trader/auto-trader
ExecStart=/usr/bin/python3 main.py
Restart=always
RestartSec=10

[Install]
WantedBy=multi-user.target
```

활성화:

```bash
sudo systemctl daemon-reload
sudo systemctl enable auto-trader
sudo systemctl start auto-trader
```

상태 확인:

```bash
sudo systemctl status auto-trader
```

## 장애 대응

### 서버 다운

Tailscale 접속 안 되면 서버 문제다. 집에 가서 확인해야 함.

예방책:
- UPS 연결 (정전 대비)
- 공유기 자동 재시작 설정

### 봇 멈춤

Docker 컨테이너 상태 확인:

```bash
docker ps -a
```

Exited 상태면:

```bash
docker logs auto-trader  # 에러 확인
docker restart auto-trader
```

### API 문제

업비트 점검 중이면 어쩔 수 없다. 에러 로그 남기고 대기하도록 처리해뒀으니 점검 끝나면 알아서 재개.

## 한 달 운영 후기

### 좋았던 점

1. **안정성**: 한 달 동안 서버 다운 0회. N2+ 튼튼하다.
2. **전력**: 전기세 거의 안 나옴. 클라우드 쓸 때보다 훨씬 저렴.
3. **Tailscale**: 외부 접속 너무 편함. 세팅도 쉽고.
4. **Docker**: 환경 관리 깔끔. 재배포도 쉬움.

### 아쉬웠던 점

1. **eMMC 용량**: 64GB가 생각보다 빠듯. 로그 로테이션 잘 해야 함.
2. **모니터링**: 그라파나 같은 거 붙이면 좋을 것 같은데 아직 안 함.
3. **백업**: 설정 파일 백업 자동화 필요.

### 수익은?

말 안 한다. 전략 공개 안 하는 것처럼 수익도 비공개. 다만 서버 유지비보단 낫다고만.

## 마무리

오드로이드 N2+로 자동매매 서버 구축하는 전 과정을 정리했다.

- 하드웨어 선택 (N2+ / eMMC)
- OS 설치 (Ubuntu)
- 원격 접속 (Tailscale)
- 환경 분리 (Docker)
- 자동매매 구조
- 모니터링과 운영

12만원 정도로 24시간 돌아가는 서버 하나 장만한 셈이다. 자동매매 아니어도 다른 용도로도 쓸 수 있다. 홈 오토메이션, 미디어 서버, 개인 Git 서버 등등.

SBC로 홈서버 구축 고민 중이라면 N2+ 추천한다.
