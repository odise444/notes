# Floor Planner 프로젝트 진행 상황

![](floor-planner-progress-1.png)


## 기술 스택
- Nuxt 3 + Vue 3 + TypeScript
- vue-konva (2D 캔버스)
- TailwindCSS + Pinia

## 완료된 기능

### 기본 기능
- 방 생성 (크기 입력)
- 그리드 배경, 줌/팬
- 치수 표시

### 가구 시스템
- 가구 라이브러리 (10종)
- 드래그 앤 드롭 배치
- 다양한 모양: 사각형, 원형, 타원형, L자형
- L자형 가구 방향/비율 독립 조절 (가로/세로)
- 드래그 리사이즈 (Transformer)

### 오브젝트 편집
- 더블클릭 또는 Enter로 편집 폼 열기
- 이름, 크기, 색상, 회전, 모양 수정
- 유효성 검사

### 문 시스템
- 문 추가/편집
- 벽 스냅 기능
- 열림 방향, 경첩 위치 설정
- 호(arc) 렌더링

### 기타
- Undo/Redo
- 저장/불러오기 (LocalStorage)
- 이미지 내보내기 (PNG)

## 키보드 단축키
| 키 | 기능 |
|---|---|
| R | 회전 (90도) |
| Delete | 삭제 |
| Enter | 편집 |
| Escape | 닫기 |
| 화살표 | 이동 |
| D | 문 열림 방향 |
| H | 경첩 위치 |
| Ctrl+Z | 실행 취소 |
| Ctrl+Y | 다시 실행 |

## 테스트
- Vitest 단위 테스트: 119개
- Playwright E2E 테스트: 18개+
