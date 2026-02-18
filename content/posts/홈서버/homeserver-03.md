---
title: "오드로이드 N2+ 홈서버 구축기 #3 - Tailscale로 어디서든 접속"
date: 2026-02-19
tags: ["Odroid", "Tailscale", "VPN", "네트워크"]
categories: ["홈서버"]
series: ["오드로이드 N2+ 홈서버 구축기"]
summary: "포트포워딩 없이 외부에서 홈서버 접속하기"
---

## 외부 접속이 필요하다

집에서만 서버 접속하면 의미가 없다. 회사에서도, 밖에서도 들어가서 상태 확인하고 싶다.

전통적인 방법은 포트포워딩이다. 공유기 설정 들어가서 외부 포트를 내부 서버로 연결. 근데 문제가 있다:

- 공유기 설정 건드려야 함
- 보안 취약 (포트 스캔에 노출)
- DDNS 설정도 필요
- ISP가 포트 막아놓은 경우도 있음

귀찮고 위험하다.

## Tailscale이 뭔가

Tailscale은 WireGuard 기반 VPN이다. 근데 일반 VPN이랑 다르다.

- 중앙 서버 없이 P2P로 연결
- 포트포워딩 필요 없음 (NAT traversal 자동)
- 설치하면 바로 연결됨
- 무료 플랜으로 개인 사용 충분

내 기기들끼리 가상 사설 네트워크를 만들어준다고 보면 된다. N2+, 내 PC, 스마트폰 다 연결해두면 어디서든 서로 접속 가능.

## 설치

N2+에서:

```bash
curl -fsSL https://tailscale.com/install.sh | sh
```

끝. 진짜 이게 끝이다.

## 계정 연결

설치 후 인증해야 한다:

```bash
sudo tailscale up
```

URL이 하나 뜬다. 브라우저에서 열어서 구글이나 깃헙 계정으로 로그인하면 된다.

인증 완료되면 Tailscale IP가 할당된다:

```bash
tailscale ip -4
```

`100.x.x.x` 형태의 IP가 나온다. 이게 Tailscale 네트워크 내에서 쓰는 IP다.

## 다른 기기에도 설치

### 내 PC (Windows)

https://tailscale.com/download 에서 다운받아서 설치. 같은 계정으로 로그인.

### 스마트폰

앱스토어에서 Tailscale 앱 설치. 역시 같은 계정으로 로그인.

## 접속 테스트

이제 어디서든 Tailscale IP로 접속하면 된다.

PC에서:
```bash
ssh -p 22022 trader@100.x.x.x
```

집이 아니어도 된다. 카페에서도, 회사에서도, LTE로도 된다.

## 상시 실행 설정

N2+가 부팅될 때 Tailscale이 자동 시작되어야 한다:

```bash
sudo systemctl enable tailscaled
```

기본으로 되어 있을 텐데, 확인 차원에서.

## MagicDNS

Tailscale 관리 페이지 (https://login.tailscale.com/admin) 가면 MagicDNS 설정이 있다. 켜두면 IP 대신 이름으로 접속할 수 있다.

```bash
ssh -p 22022 trader@n2plus
```

기기 이름 그대로 쓰면 된다. IP 외울 필요 없어서 편하다.

## 보안 고려사항

Tailscale 자체는 안전하다. WireGuard 암호화 쓰고, 키 관리도 잘 되어 있다.

근데 몇 가지 신경 쓸 건 있다:

1. **Tailscale 계정 보안**: 구글/깃헙 계정 털리면 끝이다. 2FA 필수.

2. **기기 관리**: 안 쓰는 기기는 Tailscale 관리 페이지에서 제거.

3. **ACL 설정**: 필요하면 어떤 기기가 어떤 기기에 접속할 수 있는지 제한 가능.

개인 용도로 쓸 거면 기본 설정으로도 충분하다.

## 포트포워딩 대비 장점

| 항목 | 포트포워딩 | Tailscale |
|------|-----------|-----------|
| 설정 난이도 | 공유기마다 다름 | 쉬움 |
| 보안 | 포트 노출 | 암호화된 터널 |
| DDNS | 필요 | 불필요 |
| ISP 제한 | 영향 받음 | 우회 가능 |
| 속도 | 빠름 | 약간 느림 (P2P 연결 시 거의 동일) |

속도가 약간 느릴 수 있는데, SSH로 터미널 작업하는 정도론 전혀 체감 안 된다.

## 다음 편에서

서버 환경 분리를 위해 Docker 설치한다. ARM64에서 Docker 돌리는 법.
