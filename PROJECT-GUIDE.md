# 블로그 운영 프로젝트 지침

## 이 블로그가 뭐냐면

https://mcu2cloud.kr 에서 돌아가는 기술 블로그다. Hugo로 만들었고 GitHub Pages로 배포한다. 저장소는 https://github.com/odise444/notes 여기.

테마는 PaperMod 쓰고 있는데, 심플해서 좋다. 배포는 `main` 브랜치에 push하면 GitHub Actions가 알아서 해준다.

## 폴더 구조

작업 폴더는 `D:\WorkSpace\notes\`다.

글은 `content/posts/` 밑에 시리즈별로 폴더 만들어서 넣는다. BMS 개발기, Ghidra 리버싱, Floor Planner 이런 식으로.

웹앱들은 `static/` 폴더에 넣으면 빌드할 때 그대로 복사된다. 지금 floor-planner, math, pdf2md 세 개 올라가 있다.

## 지금까지 쓴 글

총 150개 썼다.

- **BMS 개발기** 36개 - 배터리 관리 시스템 만들면서 겪은 일들
- **Ghidra 역분석** 39개 - 펌웨어 리버싱 삽질기
- **Floor Planner** 20개 - 아파트 평면도 웹앱 만든 이야기
- **STM32 부트로더** 20개 - 부트로더 직접 만들어본 경험
- **수학 챔피언** 15개 - 수학 문제 풀이 앱 개발기
- **PlatformIO 가이드** 15개 - 임베디드 개발 환경 세팅
- **PDF2MD 개발기** 5개 - PDF를 마크다운으로 바꾸는 도구

다 `posts/` 폴더 밑에 각자 폴더로 정리되어 있다.

## 글 쓸 때 규칙

### 파일 맨 위에 이거 넣어야 함

```yaml
---
title: "시리즈명 #번호 - 제목"
date: 2026-01-21
tags: ["태그1", "태그2"]
categories: ["카테고리"]
series: ["시리즈명"]
summary: "한 줄 요약"
---
```

### 어떻게 쓰냐면

직설적으로 쓴다. 구어체로. "~합니다"보다 "~한다"가 낫다. 코드 예제는 실제로 돌아가는 것 위주로 넣고, 너무 길면 읽기 지치니까 적당히 끊는다.

이모지는 제목에만 가끔 쓴다. 본문에는 잘 안 씀.

### 파일명

`{시리즈약어}-{번호}.md` 형식이다. bms-01.md, ghidra-15.md 이런 식.

## 웹앱 올리는 법

`static/` 폴더에 앱 폴더 만들고 index.html이랑 assets 폴더 넣으면 된다.

근데 주의할 게 있다. 서브 경로로 배포하니까 asset 경로를 맞춰줘야 한다. `/assets/index.js`라고 쓰면 안 되고 `/pdf2md/assets/index.js`처럼 앱 폴더명 붙여줘야 한다. 안 그러면 404 뜬다.

지금 올라간 웹앱들:
- `/floor-planner/` - 아파트 평면도
- `/math/` - 수학 문제 풀이
- `/pdf2md/` - PDF→마크다운 변환기

## 배포

그냥 git push하면 끝이다. GitHub Actions가 hugo 빌드하고 Pages에 올려준다.

로컬에서 미리보기 하고 싶으면:

```bash
cd D:\WorkSpace\notes
hugo server -D
```

도메인은 mcu2cloud.kr 쓰고 있고, GitHub Pages 설정에서 Custom domain 연결해뒀다. HTTPS는 자동으로 된다.

## 새 시리즈 시작할 때

바로 글 쓰지 말고 기획 문서 먼저 만든다. `{시리즈명}-시리즈-기획.md`로 저장하고, 뭘 쓸 건지 목록 정리해두면 나중에 헷갈리지 않는다.

## 커밋 메시지

대충 이렇게 쓴다:

- 새 시리즈 추가: `feat: PDF2MD 개발기 시리즈 추가 (5개)`
- 뭔가 고침: `fix: PDF2MD 경로 수정`
- 웹앱 추가: `feat: PDF2MD 웹앱 추가`

## 문제 생기면

### 사이트 안 보임 (404)

GitHub 저장소 Settings → Pages 가서 확인한다. Source가 "GitHub Actions"으로 되어 있어야 하고, Custom domain에 mcu2cloud.kr 들어가 있어야 한다. private으로 바꿨다가 다시 public으로 하면 설정 날아가니까 다시 해줘야 함.

### 웹앱 CSS/JS 안 불러옴

경로 문제다. `static/pdf2md/index.html` 열어서 asset 경로에 `/pdf2md/` 붙어있는지 확인한다.

### Hugo 빌드 에러

```bash
hugo --verbose
```

해서 뭐가 잘못됐는지 본다.

---

이 문서 자체도 프로젝트 루트에 `PROJECT-GUIDE.md`로 저장해뒀다. 나중에 까먹으면 여기 보면 된다.
