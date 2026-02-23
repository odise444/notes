---
title: "오드로이드 N2+ 홈서버 구축기 #8 - PicoClaw AI 어시스턴트"
date: 2026-02-24
tags: ["Odroid", "PicoClaw", "AI", "텔레그램", "Go"]
categories: ["홈서버"]
series: ["오드로이드 N2+ 홈서버 구축기"]
summary: "10MB로 돌아가는 초경량 AI 어시스턴트 설치하기"
---

## 자동매매만 돌리기엔 아깝다

N2+ 사양이 생각보다 넉넉하다. 자동매매 봇은 CPU 1~2%, 메모리 200MB 정도밖에 안 쓴다. 남는 자원으로 뭘 더 돌릴 수 없을까?

AI 어시스턴트가 떠올랐다. ChatGPT 같은 거. 근데 LLM 직접 돌리려면 GPU가 필요하고, 최소 8GB RAM은 있어야 한다. N2+로는 무리.

그러다 PicoClaw를 발견했다.

## PicoClaw가 뭔가

Sipeed에서 만든 초경량 AI 어시스턴트다. 특징이 미쳤다:

- **10MB 미만 RAM** 사용
- **1초** 만에 부팅
- **$10 하드웨어**에서도 동작
- Go 언어로 작성
- Telegram, Discord 연동

LLM을 직접 돌리는 게 아니라, OpenRouter나 Anthropic 같은 외부 API를 호출한다. 그래서 가벼운 거다. 클라이언트 역할만 하니까.

GitHub에서 일주일 만에 12K 스타 받았다. 2026년 2월에 나온 따끈따끈한 프로젝트.

## 설치

ARM64 바이너리 직접 받으면 된다:

```bash
# 최신 릴리즈 다운로드
wget https://github.com/sipeed/picoclaw/releases/latest/download/picoclaw-linux-arm64
chmod +x picoclaw-linux-arm64
sudo mv picoclaw-linux-arm64 /usr/local/bin/picoclaw
```

또는 소스에서 빌드:

```bash
git clone https://github.com/sipeed/picoclaw.git
cd picoclaw
make deps
make build
sudo make install
```

Go 1.21 이상 필요하다.

확인:

```bash
picoclaw version
```

## 설정

설정 파일 만든다:

```bash
mkdir -p ~/.picoclaw
cp config/config.example.json ~/.picoclaw/config.json
vim ~/.picoclaw/config.json
```

### LLM 프로바이더 설정

OpenRouter 쓰면 여러 모델 선택 가능하다. 무료 모델도 있음:

```json
{
  "agents": {
    "defaults": {
      "workspace": "~/.picoclaw/workspace",
      "model": "anthropic/claude-3-haiku",
      "max_tokens": 4096,
      "temperature": 0.7
    }
  },
  "providers": {
    "openrouter": {
      "api_key": "your_openrouter_api_key",
      "api_base": "https://openrouter.ai/api/v1"
    }
  }
}
```

OpenRouter 가입하고 API 키 발급받으면 된다. 무료 크레딧도 준다.

### 웹 검색 설정 (선택)

Brave Search API 연동하면 웹 검색도 가능:

```json
{
  "tools": {
    "web": {
      "brave": {
        "enabled": true,
        "api_key": "your_brave_api_key",
        "max_results": 5
      },
      "duckduckgo": {
        "enabled": true,
        "max_results": 5
      }
    }
  }
}
```

DuckDuckGo는 API 키 없이도 된다.

## 테스트

CLI로 바로 물어볼 수 있다:

```bash
picoclaw agent -m "오늘 날씨 어때?"
```

인터랙티브 모드:

```bash
picoclaw agent
```

잘 되면 다음 단계로.

## 텔레그램 연동

이게 핵심이다. 어디서든 폰으로 AI한테 물어볼 수 있다.

### 봇 토큰 발급

1. 텔레그램에서 @BotFather 검색
2. `/newbot` 명령
3. 이름, 유저네임 입력
4. 토큰 복사

### 설정 파일 수정

```json
{
  "gateway": {
    "telegram": {
      "enabled": true,
      "token": "your_telegram_bot_token",
      "allowed_users": ["your_telegram_username"]
    }
  }
}
```

`allowed_users`에 본인 텔레그램 유저네임 넣어야 한다. 안 그러면 아무나 쓸 수 있음.

### 게이트웨이 실행

```bash
picoclaw gateway
```

텔레그램 봇한테 메시지 보내면 AI가 답한다.

## Docker로 돌리기

서비스로 돌리려면 Docker가 편하다:

```yaml
# docker-compose.yml
version: '3.8'

services:
  picoclaw:
    image: sipeed/picoclaw:latest
    container_name: picoclaw
    restart: unless-stopped
    volumes:
      - ./config:/root/.picoclaw
      - ./workspace:/root/.picoclaw/workspace
    command: gateway
```

```bash
docker compose up -d
```

자동매매 봇이랑 같이 돌려도 문제없다. 메모리 10MB밖에 안 먹으니까.

## systemd 서비스 등록

Docker 안 쓰고 직접 돌리려면:

```bash
sudo vim /etc/systemd/system/picoclaw.service
```

```ini
[Unit]
Description=PicoClaw AI Assistant
After=network.target

[Service]
Type=simple
User=trader
ExecStart=/usr/local/bin/picoclaw gateway
Restart=always
RestartSec=10

[Install]
WantedBy=multi-user.target
```

```bash
sudo systemctl daemon-reload
sudo systemctl enable picoclaw
sudo systemctl start picoclaw
```

## 활용 예시

텔레그램으로 이런 것들 물어볼 수 있다:

- "비트코인 현재가 알려줘" (웹 검색)
- "파이썬으로 피보나치 함수 짜줘"
- "이 에러 메시지 해석해줘: [에러 내용]"
- "1시간 후에 리마인드해줘" (스케줄링)

자동매매 모니터링이랑 연계하면 더 유용하다:

```python
# 자동매매 봇에서 PicoClaw한테 분석 요청
import requests

def ask_picoclaw(question):
    # PicoClaw API 호출 (로컬)
    response = requests.post(
        "http://localhost:8080/api/chat",
        json={"message": question}
    )
    return response.json()["reply"]

# 시장 상황 분석 요청
analysis = ask_picoclaw("비트코인이 지금 5% 급등했어. 원인이 뭘까?")
```

## 리소스 사용량

htop으로 확인해봤다:

| 서비스 | CPU | RAM |
|--------|-----|-----|
| 자동매매 봇 | 1~2% | ~200MB |
| PicoClaw | <1% | ~15MB |
| Docker | 1% | ~50MB |
| 시스템 | 2% | ~300MB |

**총 600MB 정도**. 4GB 중 15% 수준. 여유 넘친다.

## 주의사항

### API 비용

PicoClaw는 무료지만, LLM API는 유료다. OpenRouter 통해서 저렴한 모델 쓰거나, 무료 크레딧 범위 내에서 쓰면 된다.

Claude Haiku 기준 100만 토큰당 $0.25. 개인 용도론 거의 무료 수준.

### 보안

`allowed_users` 꼭 설정하자. 안 그러면 텔레그램 봇 주소 알아낸 사람 누구나 쓸 수 있다. API 비용 폭탄 맞을 수 있음.

### 네트워크

PicoClaw는 외부 API 호출해야 해서 인터넷 필수. Tailscale 통해서 접속해도 PicoClaw가 외부 API 부르는 건 별개 문제 없다.

## 마무리

N2+ 하나로 자동매매 + AI 어시스턴트까지. 12만원짜리 서버 치고 가성비 미쳤다.

PicoClaw 덕분에 폰으로 언제든 AI한테 물어볼 수 있게 됐다. 코딩 질문, 번역, 요약, 리마인더 등등. 생각보다 자주 쓰게 된다.

홈서버의 장점이 이런 거다. 뭔가 더 돌려보고 싶으면 그냥 추가하면 된다. 클라우드처럼 비용 걱정 안 해도 되고.
