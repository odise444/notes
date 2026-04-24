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
│   └── imgs/            # 포스트 이미지 (전부 여기)
│       ├── {슬러그}/    # 서브폴더 구조 (권장)
│       └── ...          # 레거시 flat 파일들
├── layouts/             # 커스텀 레이아웃
│   └── _default/_markup/
│       └── render-image.html   # 이미지 경로 자동 처리
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

모든 포스트는 **flat 구조** (단일 `.md` 파일). Hugo Page Bundle은 쓰지 않음 — 이유는 아래 "이미지 처리" 참고.

| 시리즈 | 파일명 패턴 | 예시 |
|--------|-------------|------|
| BMS 개발기 | `ad7280a-bms-dev-{번호}.md` | ad7280a-bms-dev-01.md |
| Ghidra | `ghidra-{번호}.md` | ghidra-01.md |
| 홈서버 | `homeserver-{번호}.md` | homeserver-01.md |
| Stock-Reports | `stock-report-YYYY-MM-DD.md` | stock-report-2026-03-05.md |
| Motor-Control | `motor-control-{번호}.md` | motor-control-01.md |

### 작성 톤

- 구어체, 직설적
- "~한다" 체
- 코드는 실제 동작하는 예제 위주
- 이모지 최소화

---

## 이미지 처리

### 핵심 동작

`layouts/_default/_markup/render-image.html`이 마크다운의 모든 이미지 참조를 자동으로 `static/imgs/` 아래로 리라우팅한다.

```go
{{- if not (hasPrefix $src "imgs/") -}}
  {{- $src = printf "imgs/%s" $src -}}
{{- end -}}
```

즉, 마크다운에서 `![](xxx.png)`만 쓰면 Hugo가 알아서 `static/imgs/xxx.png`를 찾아 준다.

### 권장 방식: 슬러그별 서브폴더

이미지가 여러 장이면 포스트 슬러그 이름의 서브폴더에 모은다.

```
static/imgs/motor-control-01/
├── 01.png
├── 02.png
└── 27.png
```

포스트 내부 참조:

```markdown
![모터 사양서](motor-control-01/01.png)
![Motor Data 설정](motor-control-01/03.png)
```

### 레거시 방식: flat 파일명

이미 있는 대부분의 포스트는 `static/imgs/` 루트에 `{슬러그}-{번호}.{확장자}` 로 flat하게 둔다. 이 방식도 계속 유효하므로 기존 포스트는 건드리지 않는다.

```markdown
![사상 최고치 경신](2026-04-23-market-record-high.jpg)
```

### Page Bundle은 쓰지 않음

`content/posts/시리즈/글제목/index.md + images/` 형태의 Hugo Page Bundle은 **작동하지 않는다**. 렌더 훅이 모든 이미지 경로를 `imgs/` 아래로 강제하기 때문에 Page Bundle 내부 리소스를 못 찾는다. 반드시 flat 포스트 + `static/imgs/` 조합을 쓴다.

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
| Stock-Reports | 26 | stock-reports/ | 진행중 |

---

## 새 시리즈 시작 시

1. 기획 문서 먼저 작성: `{시리즈명}-시리즈-기획.md`
2. 폴더 생성: `content/posts/{시리즈폴더}/`
3. 이미지 서브폴더 생성: `static/imgs/{첫-포스트-슬러그}/`
4. 첫 글 작성 후 frontmatter에 series 필드 추가

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

### 이미지가 안 나옴
- `static/imgs/` 아래에 실제 파일이 있는지 확인 (경로/파일명 오타)
- 마크다운에서 `images/`, `./`, `../` 같은 접두사를 썼으면 제거. `static/imgs/`를 기준 루트로 생각하고 쓴다
- Page Bundle 구조(`content/posts/.../글제목/images/`)는 작동 안 함. flat 구조로 옮길 것
- Hugo 로컬 서버 실행 중에 대량 파일 추가하면 감지 놓치는 경우 있음 → `Ctrl+C` 후 `hugo server` 재실행

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

### layouts/_default/_markup/render-image.html
마크다운 이미지 참조를 `static/imgs/` 아래로 자동 리라우팅. 새 포스트는 이 규칙에 맞춰서 참조 경로를 쓰면 됨.

### static/ads.txt
```
google.com, pub-8951852696539817, DIRECT, f08c47fec0942fa0
```

---

## 연락처

- GitHub: https://github.com/odise444
- 블로그: https://mcu2cloud.kr
