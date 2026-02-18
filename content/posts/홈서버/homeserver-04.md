---
title: "오드로이드 N2+ 홈서버 구축기 #4 - Docker로 환경 분리"
date: 2026-02-20
tags: ["Odroid", "Docker", "ARM64", "컨테이너"]
categories: ["홈서버"]
series: ["오드로이드 N2+ 홈서버 구축기"]
summary: "ARM64 환경에서 Docker 설치하고 컨테이너 다루기"
---

## 왜 Docker인가

자동매매 봇 하나 돌리는 데 Docker까지 필요할까? 몇 가지 이유가 있다.

1. **환경 격리**: 파이썬 버전, 라이브러리 버전 충돌 걱정 없음
2. **재현성**: 다른 서버로 옮겨도 똑같이 동작
3. **관리 편의**: 시작, 중지, 로그 확인이 깔끔함
4. **확장성**: 나중에 다른 서비스 추가하기 쉬움

처음엔 귀찮아 보여도 한 번 세팅해두면 편하다.

## ARM64용 Docker 설치

N2+는 ARM64 아키텍처다. x86이랑 다르니까 주의.

공식 설치 스크립트 쓰면 알아서 맞는 버전 깔린다:

```bash
curl -fsSL https://get.docker.com | sh
```

설치 끝나면 현재 사용자를 docker 그룹에 추가:

```bash
sudo usermod -aG docker $USER
```

로그아웃했다가 다시 로그인해야 적용된다.

확인:

```bash
docker --version
docker run hello-world
```

hello-world 컨테이너가 잘 돌면 성공.

## docker-compose 설치

여러 컨테이너 관리하려면 compose가 편하다.

```bash
sudo apt install docker-compose-plugin
```

또는 최신 버전:

```bash
sudo curl -L "https://github.com/docker/compose/releases/latest/download/docker-compose-$(uname -s)-$(uname -m)" -o /usr/local/bin/docker-compose
sudo chmod +x /usr/local/bin/docker-compose
```

확인:

```bash
docker compose version
```

## 기본 명령어

자주 쓰는 것들:

```bash
# 컨테이너 목록
docker ps
docker ps -a  # 중지된 것도 포함

# 이미지 목록
docker images

# 컨테이너 실행
docker run -d --name myapp myimage

# 컨테이너 중지/시작
docker stop myapp
docker start myapp

# 로그 보기
docker logs myapp
docker logs -f myapp  # 실시간

# 컨테이너 안으로 들어가기
docker exec -it myapp /bin/bash

# 삭제
docker rm myapp        # 컨테이너
docker rmi myimage     # 이미지
```

## Dockerfile 예시

자동매매 봇용 Dockerfile:

```dockerfile
FROM python:3.11-slim

WORKDIR /app

COPY requirements.txt .
RUN pip install --no-cache-dir -r requirements.txt

COPY . .

CMD ["python", "main.py"]
```

ARM64에서도 `python:3.11-slim` 이미지 잘 동작한다. 공식 이미지들은 대부분 멀티 아키텍처 지원.

## docker-compose.yml 예시

```yaml
version: '3.8'

services:
  trader:
    build: .
    container_name: auto-trader
    restart: unless-stopped
    volumes:
      - ./config:/app/config
      - ./logs:/app/logs
    env_file:
      - .env
```

실행:

```bash
docker compose up -d
```

중지:

```bash
docker compose down
```

## 볼륨 관리

컨테이너는 죽으면 데이터도 사라진다. 유지해야 할 건 볼륨으로 빼둔다:

- `./config`: 설정 파일
- `./logs`: 로그 파일
- `./data`: DB나 저장 데이터

위 compose 파일처럼 volumes에 명시해두면 컨테이너 재시작해도 데이터 유지된다.

## ARM64 주의사항

가끔 x86 전용 이미지가 있다. 그런 건 안 돌아간다.

```
exec format error
```

이 에러 나면 ARM64 지원 안 하는 이미지다. Docker Hub에서 이미지 찾을 때 `arm64` 태그 있는지 확인하자.

대부분의 공식 이미지는 멀티 아키텍처라 걱정 없다. 문제 되는 건 개인이 만든 이미지들.

## 리소스 제한

N2+가 아무리 좋아도 RAM 4GB다. 컨테이너가 메모리 다 잡아먹으면 시스템 죽는다.

```yaml
services:
  trader:
    # ...
    deploy:
      resources:
        limits:
          memory: 512M
```

메모리 제한 걸어두면 안전하다.

## 다음 편에서

이제 진짜 자동매매 시스템 구조 잡는다. 업비트 API 연동하고 프로젝트 뼈대 만든다.
