# PDF2MD 개발기 시리즈 기획

## 개요
- **주제**: 브라우저에서 동작하는 PDF to Markdown 변환기 개발기
- **대상**: 웹 개발자, Vue.js 사용자
- **총 글 수**: 5개
- **핵심 키워드**: Vue 3, PDF.js, Tesseract.js, IndexedDB, WASM, 서버리스

## 프로젝트 특징
- 서버 없이 클라이언트에서 모든 처리
- PDF.js로 PDF 렌더링
- 클릭-투-클릭 방식 영역 선택
- Tesseract.js OCR로 Figure/Table 자동 인식
- IndexedDB로 프로젝트 저장
- JSZip으로 ZIP 내보내기

## 시리즈 구성

### #1 - 왜 만들었나
- 기존 PDF 변환 도구의 한계
- 데이터시트 문서화 작업의 고통
- 서버리스로 만들고 싶었던 이유
- 최종 목표와 기능 정의

### #2 - PDF.js로 브라우저에서 PDF 렌더링
- PDF.js 소개 및 설치
- Vue 3에서 PDF.js 연동
- 캔버스에 PDF 페이지 렌더링
- 페이지 네비게이션, 줌 구현
- 워커 설정 이슈

### #3 - 클릭-투-클릭 영역 선택 구현
- 드래그 vs 클릭-투-클릭 방식 비교
- Canvas 오버레이로 선택 영역 표시
- 마우스 이벤트 처리
- 선택 영역 좌표 계산
- 크롭 이미지 생성

### #4 - Tesseract.js OCR로 Figure/Table 인식
- Tesseract.js 소개
- 이미지에서 텍스트 추출
- Figure/Table 패턴 인식 로직
- OCR 성능 최적화
- 한글 인식 설정

### #5 - IndexedDB 저장 + ZIP 내보내기
- Dexie.js로 IndexedDB 쉽게 사용
- 프로젝트 데이터 구조 설계
- 이미지 Blob 저장
- JSZip으로 Markdown + 이미지 묶기
- 다운로드 구현

## 글 저장 위치
D:\WorkSpace\notes\content\posts\PDF2MD\

## 날짜
2026-01-21
