# Floor Planner 작업 이력 요약

## 프로젝트 개요
Vue 3 + Nuxt 3 + TypeScript 기반의 평면도 에디터 애플리케이션

## 주요 작업 내역

### 1. 레이어 정렬 기능 구현

#### 1.1 가구 레이어 순서 변경
- **파일**: `app/utils/layerOrder.ts`
- 가구에 `zIndex` 속성 추가하여 레이어 순서 관리
- `bringToFront()`, `sendToBack()`, `moveForward()`, `moveBackward()` 함수 구현
- `reorderToPosition()` 함수로 드래그 앤 드롭 순서 변경 지원

#### 1.2 레이어 패널 UI
- **파일**: `app/components/editor/LayerPanel.vue`
- 통합 레이어 목록 (방 + 이미지 + 가구) 표시
- zIndex 내림차순 정렬 (높은 값이 위에 표시)
- 드래그 앤 드롭으로 순서 변경 가능
- 타입별 아이콘 및 색상 구분 (가구: 파란색, 이미지: 보라색, 방: 회색)

#### 1.3 키보드 단축키
- `]`: 한 단계 앞으로
- `[`: 한 단계 뒤로
- `Ctrl+]`: 맨 앞으로
- `Ctrl+[`: 맨 뒤로
- `L`: 레이어 패널 토글

---

### 2. 평면도 이미지 기능

#### 2.1 이미지 업로드 및 관리
- **파일**: `app/utils/floorPlanImage.ts`
- PNG, JPEG, GIF, WebP 지원 (최대 10MB)
- 이미지 Data URL 변환 및 크기 파싱
- 기본 zIndex: -1 (가구보다 아래에 위치)

#### 2.2 이미지 컨트롤
- 투명도 조절 (슬라이더)
- 잠금/해제 기능
- 삭제 기능
- 드래그로 위치 이동

---

### 3. 방 영역 기능

#### 3.1 방 생성 및 편집
- **파일**: `app/components/editor/FloorPlanCanvas.vue`
- 방 생성 모달로 초기 크기(cm) 입력
- 드래그로 방 크기 리사이즈
- 더블클릭으로 편집 폼 열기

#### 3.2 스케일 자동 계산 기능
- **Room 인터페이스 확장**:
  ```typescript
  interface Room {
    id: string
    x: number
    y: number
    width: number      // 픽셀 크기
    height: number     // 픽셀 크기
    widthCm?: number   // 실제 크기 (cm)
    heightCm?: number  // 실제 크기 (cm)
    opacity: number
    zIndex: number
  }
  ```

- **스케일 계산 로직**:
  ```typescript
  const scale = computed(() => {
    if (room.value?.widthCm && room.value.widthCm > 0) {
      return room.value.width / room.value.widthCm;
    }
    return DEFAULT_SCALE; // 2
  });
  ```

- **워크플로우**:
  1. 평면도 이미지 업로드
  2. 방 생성 후 드래그로 평면도에 맞게 크기 조절
  3. 방 더블클릭하여 실제 cm 치수 입력
  4. 스케일이 자동으로 계산되어 가구 크기가 올바르게 표시됨

---

### 4. 레이어 순서 수정

#### 4.1 레이어 렌더링 순서
- **변경 전**: 방 → 이미지 → 가구
- **변경 후**: 이미지 → 방 → 가구

이미지가 맨 아래에 위치하여 방 영역이 이미지 위에 표시됨

---

### 5. 문(Door) 기능

#### 5.1 문 생성 및 편집
- **파일**: `app/utils/door.ts`
- 벽에 자동 스냅
- 열림 방향 (inside/outside)
- 경첩 위치 (left/right)
- 호(arc) 및 패널 렌더링

#### 5.2 문 편집 폼
- 너비 수정
- 열림 방향 변경
- 경첩 위치 변경
- 삭제 기능

---

### 6. 벽체(Wall) 기능

#### 6.1 벽체 생성 및 편집
- **파일**: `app/utils/wall.ts`
- 드래그로 벽체 생성
- 시작점/끝점 좌표 및 두께 관리
- 각도 스냅 (Shift 키) 지원
- 기존 벽체 끝점에 자동 스냅

#### 6.2 벽체 폴리곤
- 연결된 벽체들을 폴리곤으로 변환
- 닫힌 영역 자동 감지
- 폴리곤 뷰 토글 (P키)

#### 6.3 벽체 편집 폼
- 길이, 두께 수정
- 색상 변경 (내벽: 노란색, 외벽: 진한 회색)
- 삭제 기능

---

### 7. 객체 그룹화 기능

#### 7.1 그룹 생성 및 관리
- **파일**: `app/utils/group.ts`
- 가구, 벽체, 방을 하나의 그룹으로 묶기
- 그룹 선택 시 멤버 전체 하이라이트
- 그룹 드래그 시 멤버 전체 이동

#### 7.2 레이어 패널 그룹 표시
- 그룹 펼침/접기 (하위 노드 표시)
- 그룹 해제 기능
- 다중 선택 모드로 새 그룹 생성

---

### 8. Activity Bar UI

#### 8.1 VSCode 스타일 사이드바
- **파일**: `app/components/layout/ActivityBar.vue`, `ActivityBarItem.vue`
- 좌측에 세로 아이콘 메뉴 고정 배치
- 가구 라이브러리, 레이어, 히스토리, 설정 탭

#### 8.2 패널 전환
- 아이콘 클릭 시 우측 패널 내용 전환
- 선택된 아이콘 강조 표시 (왼쪽 흰색 바)
- 마우스 호버 시 툴팁 표시

#### 8.3 레이어 패널 통합
- 기존 플로팅 레이어 패널을 Activity Bar에 통합
- `embedded` 모드로 전체 높이 사용
- 닫기 버튼 숨김 (embedded 모드)

---

### 10. 버그 수정

#### 10.1 scale computed ref 사용 오류 수정
- **문제**: 템플릿에서 `scale.value` 사용 시 TypeScript 오류
- **해결**: Vue 템플릿에서 computed ref는 자동 언래핑되므로 `.value` 제거
- **수정 위치**: 84, 85, 105, 106, 152-155, 171, 172, 183-185, 195-198, 219, 231번 줄

#### 10.2 MIN_SIZE/MAX_SIZE 상수 수정
- **문제**: `const MIN_SIZE = 20 * scale`에서 scale이 computed ref라 오류
- **해결**: cm 단위 상수로 변경하고 함수 내에서 동적 계산
  ```typescript
  const MIN_SIZE_CM = 20;
  const MAX_SIZE_CM = 500;

  const boundBoxFunc = (oldBox, newBox) => {
    const minSizePx = MIN_SIZE_CM * scale.value;
    const maxSizePx = MAX_SIZE_CM * scale.value;
    // ...
  };
  ```

#### 10.3 onFurnitureTransformEnd 수정
- **문제**: `scale.valueX`, `scale.valueY` 잘못된 속성 접근
- **해결**: `scaleX`, `scaleY` (노드 변환 스케일 값) 사용

#### 10.4 그룹 선택 해제 버그 수정
- **문제**: 그룹 선택 후 다른 곳 클릭해도 선택 효과가 유지됨
- **해결**: 가구, 벽체, 방, 문, 이미지 클릭 시 `selectedGroup.value = null` 추가

---

### 11. 테스트

#### 11.1 E2E 테스트
- **파일**: `e2e/layerOrder.spec.ts`, `e2e/objectEdit.spec.ts` 등
- Playwright 기반 48개 테스트 케이스
- 모든 테스트 통과 확인

---

## 커밋 이력

1. `문 호(arc) 방향 및 범위 버그 수정`
2. `TDD 환경 구축 및 문 호(arc) 렌더링 버그 수정`
3. `가구 선택 시 정보 팝업 표시 기능 추가`
4. `문 드래그 시 벽 자동 스냅 기능 구현`
5. `문 패널 선 위치 수정 및 선택 해제 기능 추가`
6. `vue-konva insertBefore DOM 충돌 버그 수정`
7. `레이어 패널 드래그 앤 드롭 순서 변경 기능 추가`
8. `평면도 이미지 레이어 순서 변경 기능 추가`
9. `방 영역 레이어 패널 추가 및 순서 변경 기능`
10. `평면도에 맞춘 방 크기 입력 시 scale 자동 계산 기능`
11. `scale computed ref 사용 오류 수정`
12. `평면도 이미지와 방 레이어 순서 및 스케일 계산 수정`
13. `벽체 기능 및 객체 그룹화 기능 구현`
14. `그룹 기능 개선 및 선택 해제 버그 수정`
15. `VSCode 스타일 Activity Bar UI 추가`

---

## 파일 구조

```
app/
├── components/
│   ├── editor/
│   │   ├── FloorPlanCanvas.vue    # 메인 캔버스 컴포넌트
│   │   ├── LayerPanel.vue         # 레이어 패널 컴포넌트
│   │   ├── FurnitureEditForm.vue  # 가구 편집 폼
│   │   ├── DoorEditForm.vue       # 문 편집 폼
│   │   └── WallEditForm.vue       # 벽체 편집 폼
│   ├── layout/
│   │   ├── ActivityBar.vue        # 좌측 Activity Bar
│   │   └── ActivityBarItem.vue    # Activity Bar 아이콘 버튼
│   └── sidebar/
│       └── FurnitureLibrary.vue   # 가구 라이브러리 패널
├── layouts/
│   └── default.vue                # 기본 레이아웃 (헤더)
├── pages/
│   └── index.vue                  # 메인 페이지
├── types/
│   └── furniture.ts               # 가구 타입 정의
└── utils/
    ├── door.ts                    # 문 관련 유틸리티
    ├── floorPlanImage.ts          # 평면도 이미지 유틸리티
    ├── floorPlanStorage.ts        # 로컬 스토리지 저장/불러오기
    ├── group.ts                   # 객체 그룹화 유틸리티
    ├── layerOrder.ts              # 레이어 순서 관리
    ├── objectEdit.ts              # 오브젝트 편집 유틸리티
    └── wall.ts                    # 벽체 관련 유틸리티

e2e/
├── layerOrder.spec.ts             # 레이어 정렬 테스트
├── objectEdit.spec.ts             # 오브젝트 편집 테스트
├── floorPlanImage.spec.ts         # 평면도 이미지 테스트
├── customFurniture.spec.ts        # 사용자 정의 가구 테스트
├── measureTool.spec.ts            # 측정 도구 테스트
└── wall.spec.ts                   # 벽체 기능 테스트
```

---

## 기술 스택

- **프레임워크**: Nuxt 3, Vue 3
- **언어**: TypeScript
- **캔버스**: Konva.js + vue-konva
- **스타일**: TailwindCSS
- **상태관리**: Vue Composition API (ref, computed)
- **테스트**: Playwright (E2E)
- **저장**: LocalStorage
