# Floor Planner 개발기

이사 준비하다가 만든 웹 평면도 도구. 가구 배치 시뮬레이션용.

---

## 프로젝트

- **기술**: Nuxt 3, vue-konva, Pinia, TailwindCSS
- **테스트**: Vitest 170개 + Playwright E2E 39개
- **배포**: https://mcu2cloud.kr/floor-planner/

---

## 스크린샷 목록

저장 위치: `content/posts/Floor-Planner/`

### 필수 (10개)

- [x] fp-main.png - 메인 화면 전체 (방 + 가구 배치된 상태)![fp-main.png](FloorPlanner-시리즈-기획-3.png)
- [x] fp-empty.png - 빈 캔버스 (그리드만)![fp-empty.png](FloorPlanner-시리즈-기획-1.png)
- [x] fp-furniture-panel.png - 좌측 가구 팔레트![fp-furniture-panel.png](FloorPlanner-시리즈-기획-4.png)
- [x] fp-selected.png - 가구 선택 상태 (Transformer 핸들)![fp-selected.png](FloorPlanner-시리즈-기획-5.png)
- [x] fp-edit-form.png - 더블클릭 편집 폼![fp-edit-form.png](FloorPlanner-시리즈-기획-6.png)
- [ ] fp-door.png - 문 추가된 상태 (호 표시)![fp-door.png](FloorPlanner-시리즈-기획-7.png)
- [x] fp-measure.png - 측정 도구 사용 중![fp-measure.png](FloorPlanner-시리즈-기획-8.png)
- [ ] fp-image-upload.png - 평면도 이미지 업로드
- [x] fp-export.png - 내보내기 결과![fp-export.png](Pasted%20image%2020251222170615.png)
- [ ] fp-toolbar.png - 상단 툴바

### 추가 (필요시)

- [ ] fp-drag-drop.png - 가구 드래그 중
- [ ] fp-shapes.png - 다양한 모양 (사각형, 원형, L자형)
- [ ] fp-resize.png - 리사이즈 중
- [ ] fp-rotate.png - 회전 중
- [ ] fp-door-snap.png - 벽에 스냅된 문
- [ ] fp-door-direction.png - 열림 방향 비교
- [ ] fp-measure-multi.png - 여러 측정
- [ ] fp-zoom-out.png - 줌 아웃
- [ ] fp-zoom-in.png - 줌 인
- [ ] fp-image-opacity.png - 투명도 조절
- [ ] fp-image-locked.png - 이미지 잠금
- [ ] fp-undo-redo.png - Undo/Redo 버튼
- [ ] fp-shortcuts.png - 단축키 도움말

---

## 글 목록 (20개)

### Part 1: 시작

- [ ] #1 왜 만들게 됐나
- [ ] #2 캔버스 라이브러리 선택

### Part 2: 캔버스 기초

- [ ] #3 좌표계 삽질
- [ ] #4 무한 그리드
- [ ] #5 줌/팬

### Part 3: 가구 시스템

- [ ] #6 가구 시스템
- [ ] #7 다양한 모양
- [ ] #8 Konva Transformer
- [ ] #9 회전된 객체

### Part 4: 문/도구

- [ ] #10 문 시스템
- [ ] #11 측정 도구
- [ ] #12 키보드 단축키
- [ ] #13 Undo/Redo

### Part 5: 이미지/저장

- [ ] #14 이미지 업로드
- [ ] #15 저장/불러오기
- [ ] #16 PNG 내보내기

### Part 6: 테스트/최적화

- [ ] #17 Vitest로 Konva 테스트
- [ ] #18 Playwright E2E
- [ ] #19 성능 최적화
- [ ] #20 Vue Reactivity + Konva 충돌

---

## 진행 상황

```
프로젝트 완성     ✅
배포             ✅
테스트           ✅ (209개)

스크린샷         0/10 (필수)
글 작성          0/20
```

---

## 메모

- BMS 36개, Ghidra 39개 완료
- 프론트엔드 개발자 타겟
- 캔버스 + Vue 조합 경험 공유
