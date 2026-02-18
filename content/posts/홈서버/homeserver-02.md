---
title: "오드로이드 N2+ 홈서버 구축기 #2 - Ubuntu 설치와 기본 세팅"
date: 2026-02-18
tags: ["Odroid", "Ubuntu", "Linux", "eMMC"]
categories: ["홈서버"]
series: ["오드로이드 N2+ 홈서버 구축기"]
summary: "eMMC에 Ubuntu 굽고 SSH 세팅까지"
---

## 이미지 다운로드

오드로이드 공식 위키에서 Ubuntu 이미지 받는다. N2+ 전용으로 나와 있다.

https://wiki.odroid.com/odroid-n2/os_images/ubuntu

Server 버전으로 받았다. GUI 필요 없으니까. 용량도 작고 가볍다.

## eMMC에 굽기

eMMC는 SD카드처럼 그냥 꽂을 수가 없다. USB 리더기가 필요하다. 오드로이드에서 파는 eMMC-USB 리더기 샀다.

굽는 건 balenaEtcher 쓰면 된다. 윈도우에서:

1. eMMC를 USB 리더기에 꽂고 PC에 연결
2. balenaEtcher 실행
3. 다운받은 이미지 선택
4. eMMC 드라이브 선택
5. Flash

5분이면 끝난다.

## 첫 부팅

eMMC를 N2+ 보드 밑면에 끼운다. 딸깍 소리 나게 밀어넣으면 된다. 전원 연결하면 부팅 시작.

N2+는 Petitboot이라는 부트로더가 있다. HDMI 연결해서 보면 부팅 메뉴 뜬다. eMMC에서 부팅하도록 선택하면 된다. 한 번 설정해두면 다음부턴 자동으로 eMMC에서 부팅한다.

처음 로그인 정보:
- ID: `root`
- PW: `odroid`

## 기본 세팅

### 비밀번호 변경

```bash
passwd
```

당연히 바꿔야 한다. 기본 비밀번호 그대로 두면 보안 구멍.

### 사용자 추가

root로 계속 쓰긴 싫으니까 일반 유저 만든다.

```bash
adduser trader
usermod -aG sudo trader
```

이제부터 trader 계정으로 작업한다.

### 패키지 업데이트

```bash
sudo apt update
sudo apt upgrade -y
```

시간 좀 걸린다. 커피 한 잔.

### 필수 패키지 설치

```bash
sudo apt install -y vim git curl wget htop
```

자주 쓰는 것들 미리 깔아둔다.

### 타임존 설정

```bash
sudo timediff set-timezone Asia/Seoul
```

자동매매는 시간이 중요하다. 한국 시간으로 맞춰둔다.

## SSH 설정

Ubuntu Server엔 SSH가 기본으로 깔려 있다. 근데 몇 가지 설정 바꿔주는 게 좋다.

```bash
sudo vim /etc/ssh/sshd_config
```

바꿀 것들:

```
Port 22022                    # 기본 포트 변경
PermitRootLogin no            # root 로그인 금지
PasswordAuthentication yes    # 일단 비밀번호 허용 (나중에 키 인증으로 변경)
```

저장하고 SSH 재시작:

```bash
sudo systemctl restart sshd
```

포트 바꿨으니까 접속할 때 `-p 22022` 붙여야 한다.

## 고정 IP 설정

공유기에서 DHCP로 IP 받으면 재부팅할 때마다 바뀔 수 있다. 서버는 IP가 고정되어야 편하다.

### 방법 1: 공유기에서 설정

공유기 관리 페이지 들어가서 DHCP 예약 설정. N2+의 MAC 주소에 특정 IP 할당해두면 된다. 이게 제일 간단하다.

### 방법 2: Ubuntu에서 설정

netplan 설정 파일 수정:

```bash
sudo vim /etc/netplan/01-netcfg.yaml
```

```yaml
network:
  version: 2
  ethernets:
    eth0:
      addresses:
        - 192.168.0.100/24
      gateway4: 192.168.0.1
      nameservers:
        addresses:
          - 8.8.8.8
          - 8.8.4.4
```

적용:

```bash
sudo netplan apply
```

나는 공유기에서 설정하는 방법 썼다. 더 깔끔하니까.

## 접속 테스트

다른 PC에서:

```bash
ssh -p 22022 trader@192.168.0.100
```

잘 들어가지면 성공.

## 다음 편에서

집 밖에서도 접속해야 한다. Tailscale 설치해서 어디서든 접속 가능하게 만든다.
