# Floor Planner 개발기

이사 준비하다가 만든 웹 평면도 도구. 가구 배치 시뮬레이션용.

---

## 프로젝트

- **기술**: Nuxt 3, vue-konva, Pinia, TailwindCSS
- **테스트**: Vitest 170개 + Playwright E2E 39개
- **배포**: https://mcu2cloud.kr/floor-planner/

---

## 스크린샷 목록

저장 위치: `static/imgs/`

### 필수 (9개)

- [x] fp-main.png - 메인 화면 전체 (방 + 가구 배치된 상태)
- [x] fp-empty.png - 빈 캔버스 (그리드만)
- [x] fp-furniture-panel.png - 좌측 가구 팔레트
- [x] fp-selected.png - 가구 선택 상태 (Transformer 핸들)
- [x] fp-edit-form.png - 더블클릭 편집 폼
- [x] fp-door.png - 문 추가된 상태 (호 표시)
- [x] fp-measure.png - 측정 도구 사용 중
- [x] fp-image-upload.png - 평면도 이미지 업로드
- [x] fp-toolbar.png - 상단 툴바 / 내보내기

---

## 글 목록 (20개)

### Part 1: 시작

- [x] #1 왜 만들게 됐나
- [x] #2 캔버스 라이브러리 선택

### Part 2: 캔버스 기초

- [x] #3 좌표계 삽질
- [x] #4 무한 그리드
- [x] #5 줌/팬

### Part 3: 가구 시스템

- [x] #6 가구 시스템
- [x] #7 다양한 모양
- [x] #8 Konva Transformer
- [x] #9 회전된 객체

### Part 4: 문/도구

- [x] #10 문 시스템
- [x] #11 측정 도구
- [x] #12 키보드 단축키
- [x] #13 Undo/Redo

### Part 5: 이미지/저장

- [x] #14 이미지 업로드
- [x] #15 저장/불러오기
- [x] #16 PNG 내보내기

### Part 6: 테스트/최적화

- [x] #17 Vitest로 Konva 테스트
- [x] #18 Playwright E2E
- [x] #19 성능 최적화
- [x] #20 Vue Reactivity + Konva 충돌

---

## 진행 상황

```
프로젝트 완성     ✅
배포             ✅
테스트           ✅ (209개)

스크린샷         9/9 ✅
글 작성          20/20 ✅
```

---

## 완료!

- BMS 36개 ✅
- Ghidra 39개 ✅
- Floor Planner 20개 ✅
- **총 95개**
