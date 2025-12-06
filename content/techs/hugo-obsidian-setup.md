---
title: "Hugo + Obsidian + GitHub Pages 블로그 설정 가이드"
date: 2025-12-06
draft: false
tags: ["Hugo", "Obsidian", "GitHub", "블로그"]
---

## 개요

Obsidian에서 Markdown 작성 → Git push → GitHub Actions 자동 빌드 → GitHub Pages 배포

## 전체 구성

| 역할 | 도구 |
|------|------|
| 에디터 | Obsidian |
| 동기화 | Git + GitHub Sync 플러그인 |
| 빌드 | Hugo |
| 호스팅 | GitHub Pages |
| URL | https://odise444.github.io/notes/ |

---

## 새 PC 설정 방법 (회사 등)

### 1. 필수 도구 설치

```powershell
# Scoop 설치
Set-ExecutionPolicy -ExecutionPolicy RemoteSigned -Scope CurrentUser
Invoke-RestMethod -Uri https://get.scoop.sh | Invoke-Expression

# Git, Hugo 설치
scoop install git hugo-extended

# Git 설정
git config --global user.name "이름"
git config --global user.email "이메일"
```

### 2. 저장소 클론

```powershell
cd D:\WorkSpace
git clone --recursive https://github.com/odise444/notes.git
```

### 3. Obsidian 설정

1. Obsidian 다운로드/설치: https://obsidian.md
2. 실행 → **보관함 폴더 열기** → `D:\WorkSpace\notes` 선택
3. 설정 → 커뮤니티 플러그인 → **GitHub Sync** 설치
4. GitHub Sync 설정:
   - Remote URL: `https://github.com/odise444/notes.git`
   - Auto sync on startup: ON
   - Auto sync at interval: 5 (분)

### 4. GitHub 인증

처음 Sync 시 GitHub 로그인 팝업 → 인증

---

## 글 작성 방법

1. Obsidian에서 `content/posts/` 또는 `content/techs/` 폴더에 md 파일 생성
2. 상단에 Front Matter 추가:

```yaml
---
title: "글 제목"
date: 2025-12-06
draft: false
tags: ["태그1", "태그2"]
---
```

3. 본문 작성
4. Sync 아이콘 클릭 (또는 자동 동기화 대기)
5. GitHub Actions 빌드 후 사이트 반영

---

## 폴더 구조

```
notes/
├── content/
│   ├── posts/      # 일반 글
│   ├── techs/      # 기술 노트
│   └── search.md   # 검색 페이지
├── themes/PaperMod # 테마
├── hugo.toml       # Hugo 설정
└── .github/workflows/hugo.yml  # 배포 워크플로우
```

---

## 주요 파일

### hugo.toml (사이트 설정)

```toml
baseURL = 'https://odise444.github.io/notes/'
languageCode = 'ko-kr'
title = 'My Notes'
theme = 'PaperMod'

[[menu.main]]
  name = "Posts"
  url = "/posts/"
  weight = 2
```

### .github/workflows/hugo.yml (배포)

- main 브랜치 push 시 자동 빌드/배포
- Hugo 버전: 0.147.0

---

## 로컬 테스트

```powershell
cd D:\WorkSpace\notes
hugo server -D
# http://localhost:1313/notes/ 에서 확인
```

---

## 문제 해결

### Push 거부 (토큰 노출)
`.gitignore`에 추가:
```
.obsidian/plugins/githobs/data.json
```

### 빌드 실패 (Hugo 버전)
`.github/workflows/hugo.yml`에서 `HUGO_VERSION` 확인 (0.146.0 이상 필요)

### Upstream 설정 안됨
```powershell
git push --set-upstream origin main
```
