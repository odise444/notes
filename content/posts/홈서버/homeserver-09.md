---
title: "오드로이드 N2+ 홈서버 구축기 #9 - eMMC에서 SSD로 업그레이드"
date: 2026-02-25
tags: ["Odroid", "SSD", "NVMe", "스토리지"]
categories: ["홈서버"]
series: ["오드로이드 N2+ 홈서버 구축기"]
summary: "64GB eMMC 용량 한계, SSD로 갈아타기"
---

## eMMC 64GB의 한계

처음엔 64GB면 충분할 줄 알았다. 자동매매 봇이랑 PicoClaw 돌리는 데 얼마나 쓰겠어?

근데 몇 달 운영하다 보니 슬슬 빠듯해졌다:

```bash
df -h
```

```
Filesystem      Size  Used Avail Use%
/dev/mmcblk0p2   58G   47G   11G  81%
```

로그 파일이 생각보다 많이 쌓였다. Docker 이미지도 몇 개 받다 보니 용량이 금방 찬다. 로그 로테이션 열심히 해도 한계가 있다.

그리고 속도도 아쉽다. eMMC가 SD카드보단 빠르지만, SSD에 비하면 확연히 느리다.

## N2+에서 SSD 쓰는 방법

N2+는 USB 3.0 포트가 있다. 여기에 SSD 연결하면 된다.

선택지:

1. **USB to SATA 케이블 + 2.5인치 SSD**: 저렴하고 간단
2. **USB to NVMe 케이스 + NVMe SSD**: 더 빠름
3. **USB SSD (외장형)**: 제일 간편

나는 USB to NVMe 케이스 + NVMe SSD 조합으로 갔다. 속도가 제일 빠르니까.

## 구매한 것들

| 품목 | 제품 | 가격 |
|------|------|------|
| SSD | 삼성 980 500GB | 55,000원 |
| 케이스 | USB 3.1 to NVMe 케이스 | 15,000원 |

총 7만원 정도. 500GB면 당분간 용량 걱정 없다.

## SSD 준비

### 파티션 생성

SSD를 N2+에 연결하고:

```bash
lsblk
```

```
NAME         MAJ:MIN RM   SIZE RO TYPE MOUNTPOINT
sda            8:0    0 465.8G  0 disk
mmcblk0      179:0    0  58.2G  0 disk
├─mmcblk0p1  179:1    0   128M  0 part /boot
└─mmcblk0p2  179:2    0    58G  0 part /
```

`sda`가 SSD다. 파티션 만들자:

```bash
sudo fdisk /dev/sda
```

- `g`: GPT 파티션 테이블 생성
- `n`: 새 파티션 (기본값으로 전체 용량)
- `w`: 저장

### 파일시스템 생성

```bash
sudo mkfs.ext4 /dev/sda1
```

### 마운트 테스트

```bash
sudo mkdir /mnt/ssd
sudo mount /dev/sda1 /mnt/ssd
df -h /mnt/ssd
```

잘 되면 다음 단계로.

## 시스템 마이그레이션

eMMC에서 SSD로 전체 시스템을 옮긴다.

### rsync로 복사

```bash
sudo rsync -axHAWXS --numeric-ids --info=progress2 / /mnt/ssd/ --exclude={"/dev/*","/proc/*","/sys/*","/tmp/*","/run/*","/mnt/*","/media/*","/lost+found"}
```

시간 좀 걸린다. 데이터 양에 따라 10~30분.

### fstab 수정

SSD의 fstab 수정해야 부팅 시 제대로 마운트된다:

```bash
sudo blkid /dev/sda1
```

UUID 복사해두고:

```bash
sudo vim /mnt/ssd/etc/fstab
```

```
UUID=복사한-UUID  /  ext4  defaults,noatime  0  1
```

기존 eMMC 관련 줄은 주석 처리.

### boot 설정

N2+는 Petitboot 부트로더 쓴다. USB에서 부팅하도록 설정해야 한다.

```bash
sudo vim /mnt/ssd/boot/boot.ini
```

`root=` 부분을 SSD UUID로 변경:

```
root=UUID=복사한-UUID
```

## 부팅 순서 변경

### Petitboot 설정

1. N2+에 HDMI, 키보드 연결
2. 재부팅
3. Petitboot 메뉴에서 USB 장치 선택
4. "Set as default" 설정

또는 eMMC를 아예 빼버리면 자동으로 USB에서 부팅한다.

### 확인

재부팅 후:

```bash
lsblk
```

```
NAME   MAJ:MIN RM   SIZE RO TYPE MOUNTPOINT
sda      8:0    0 465.8G  0 disk
└─sda1   8:1    0 465.8G  0 part /
```

루트가 SSD로 바뀌었으면 성공.

## 속도 비교

### eMMC (이전)

```bash
sudo hdparm -Tt /dev/mmcblk0
```

```
Timing buffered disk reads: 220 MB/sec
```

### SSD (이후)

```bash
sudo hdparm -Tt /dev/sda
```

```
Timing buffered disk reads: 380 MB/sec
```

체감도 확실히 빨라졌다. Docker 컨테이너 시작 시간도 단축됐고, 로그 쓰기도 부드럽다.

## fstrim 설정

SSD 수명을 위해 TRIM 활성화:

```bash
sudo systemctl enable fstrim.timer
sudo systemctl start fstrim.timer
```

주기적으로 TRIM 실행해서 SSD 성능 유지.

## eMMC는 어떻게?

두 가지 선택지:

1. **백업용으로 보관**: 나중에 SSD 문제 생기면 eMMC로 부팅 가능
2. **다른 용도로 활용**: 다른 SBC에 꽂아서 쓰기

나는 그냥 보관해뒀다. 언젠간 쓸 일 있겠지.

## 현재 구성

| 항목 | 이전 | 이후 |
|------|------|------|
| 저장장치 | eMMC 64GB | NVMe SSD 500GB |
| 속도 | 220 MB/s | 380 MB/s |
| 용량 | 58GB 가용 | 465GB 가용 |
| 가격 | 25,000원 | 70,000원 |

7만원 더 들었지만 용량 8배, 속도 1.7배. 충분히 가치 있다.

## 마무리

eMMC 64GB로 시작하는 건 나쁘지 않다. 일단 돌려보고, 용량이 부족하면 그때 SSD로 업그레이드하면 된다.

처음부터 SSD로 가도 되지만, N2+의 장점 중 하나가 저렴한 시작 비용이니까. 필요할 때 확장하는 게 합리적이다.

다음엔 뭘 더 추가해볼까. 미디어 서버? 개인 Git 서버? 홈서버의 확장성은 끝이 없다.
