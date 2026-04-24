# mcu2cloud.kr 블로그 운영 가이드

## 기본 정보

| 항목 | 내용 |
|------|------|
| 사이트 | https://mcu2cloud.kr |
| 저장소 | https://github.com/odise444/notes |
| 로컬 경로 | C:\WorkSpaces\notes\ |
| 엔진 | Hugo + PaperMod 테마 |
| 배포 | GitHub Pages (main push 시 자동) |
| 댓글 | Giscus (repo: odise444/notes) |
| 광고 | Google AdSense (pub-8951852696539817) |

---

## 폴더 구조

```
C:\WorkSpaces\notes\
├── content/
│   ├── posts/           # 블로그 글
│   │   ├── BMS/
│   │   ├── Ghidra/
│   │   ├── Floor-Planner/
│   │   ├── STM32-Bootloader/
│   │   ├── Math-Champion/
│   │   ├── PlatformIO-Guide/
│   │   ├── PDF2MD/
│   │   ├── HomeServer/
│   │   ├── Motor-Control/
│   │   ├── stock-reports/
│   │   └── ...
│   └── techs/           # 기술 문서
├── static/              # 정적 파일 (웹앱, 이미지)
│   ├── floor-planner/
│   ├── math/
│   ├── pdf2md/
│   ├── ads.txt
│   └── imgs/
├── layouts/             # 커스텀 레이아웃
├── hugo.toml            # Hugo 설정
└── .github/workflows/   # GitHub Actions
```

---

## 글 작성 규칙

### Frontmatter 형식

```yaml
---
title: "시리즈명 #번호 - 제목"
date: YYYY-MM-DD
tags: ["태그1", "태그2"]
categories: ["카테고리"]
series: ["시리즈명"]
summary: "한 줄 요약"
---
```

### 파일명 규칙

| 시리즈 | 파일명 패턴 | 예시 |
|--------|-------------|------|
| BMS 개발기 | `ad7280a-bms-dev-{번호}.md` | ad7280a-bms-dev-01.md |
| Ghidra | `ghidra-{번호}.md` | ghidra-01.md |
| 홈서버 | `homeserver-{번호}.md` | homeserver-01.md |
| Stock-Reports | `stock-report-YYYY-MM-DD.md` | stock-report-2026-03-05.md |
| Motor-Control | `{slug}/index.md` (Page Bundle) | maxon-escon-.../index.md |

### 작성 톤

- 구어체, 직설적
- "~한다" 체
- 코드는 실제 동작하는 예제 위주
- 이모지 최소화

### 이미지 처리

**일반 포스트**: `![설명](images/파일명.png)` 형식으로 참조

**Page Bundle 구조** (이미지 많은 글):
```
content/posts/시리즈명/글제목/
├── index.md
└── images/
    ├── 01_xxx.png
    ├── 02_xxx.png
    └── ...
```

---

## 시리즈 현황

| 시리즈 | 개수 | 폴더 | 상태 |
|--------|------|------|------|
| BMS 개발기 | 36 | BMS/ | 완료 |
| Ghidra 역분석 | 39 | Ghidra/ | 완료 |
| Floor Planner | 20 | Floor-Planner/ | 완료 |
| STM32 부트로더 | 20 | STM32-Bootloader/ | 완료 |
| 수학 챔피언 | 15 | Math-Champion/ | 완료 |
| PlatformIO | 15 | PlatformIO-Guide/ | 완료 |
| PDF2MD | 5 | PDF2MD/ | 완료 |
| 홈서버 구축기 | 9 | HomeServer/ | 진행중 |
| Motor-Control | 1 | Motor-Control/ | 진행중 |
| Stock-Reports | 2 | stock-reports/ | 진행중 |

**총 글 수**: 약 162개

---

## 새 시리즈 시작 시

1. 기획 문서 먼저 작성: `{시리즈명}-시리즈-기획.md`
2. 폴더 생성: `content/posts/{시리즈폴더}/`
3. 첫 글 작성 후 frontmatter에 series 필드 추가

---

## 웹앱 배포

### 위치
`static/` 폴더에 배치

### 주의사항
- 서브경로 배포이므로 asset 경로에 앱폴더명 포함 필요
- 예: `/floor-planner/js/app.js`

### 현재 웹앱
- `/floor-planner/` - 평면도 도구
- `/math/` - 수학 챔피언 게임
- `/pdf2md/` - PDF to Markdown 변환

---

## 배포 프로세스

```bash
cd C:\WorkSpaces\notes
git add .
git commit -m "feat: 설명"
git push
```

GitHub Actions가 자동으로 Hugo 빌드 → GitHub Pages 배포

---

## 문제 해결

### 404 에러
- 폴더명/파일명에 한글, 공백 있으면 안 됨
- 폴더명 대소문자 확인 (URL은 소문자)

### Hugo 빌드 에러
- `"<" in attribute name` → 머지 충돌 마커 남아있음
- `layouts/partials/` 파일들 확인

### 대소문자 폴더명 변경 (Windows)
Windows는 대소문자만 다른 폴더명 직접 변경 불가. 2단계로:
```powershell
# Stock-Reports → stock-reports
Rename-Item "Stock-Reports" "stock-reports-temp"
Rename-Item "stock-reports-temp" "stock-reports"
```

### AdSense "찾을 수 없음"
`static/ads.txt` 있으면 정상. Google 재크롤링 대기 (24시간~며칠)

---

## 주요 설정 파일

### hugo.toml
```toml
baseURL = 'https://mcu2cloud.kr/'
languageCode = 'ko-kr'
title = 'Simply the Best!!!'
theme = 'PaperMod'

[params.giscus]
  repo = "odise444/notes"
  # ...
```

### static/ads.txt
```
google.com, pub-8951852696539817, DIRECT, f08c47fec0942fa0
```

---

## 연락처

- GitHub: https://github.com/odise444
- 블로그: https://mcu2cloud.kr
