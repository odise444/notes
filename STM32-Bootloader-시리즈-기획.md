# STM32 커스텀 부트로더 개발기

CAN/UART로 현장에서 펌웨어 업데이트하는 부트로더 직접 만들기.

---

## 프로젝트

- **MCU**: STM32F103VE (512KB Flash)
- **통신**: CAN 500kbps
- **도구**: STM32CubeIDE, python-can

---

## 스크린샷 목록

저장 위치: `static/imgs/`

- [ ] bl-memory-map.png - 메모리 맵 그림
- [ ] bl-flash-layout.png - Flash 영역 분리
- [ ] bl-cubemx.png - CubeMX 설정
- [ ] bl-linker.png - 링커 스크립트
- [ ] bl-debug.png - 디버깅 화면
- [ ] bl-python-upload.png - Python 업로더 실행

---

## 글 목록 (20개)

### Part 1: 기초 (4개)

- [ ] #1 왜 커스텀 부트로더인가
- [ ] #2 STM32 메모리 맵
- [ ] #3 부트로더 vs 앱 영역 분리
- [ ] #4 링커 스크립트 설정

### Part 2: 앱 점프 (4개)

- [ ] #5 앱 유효성 검사
- [ ] #6 Vector Table 재배치
- [ ] #7 앱으로 점프하기
- [ ] #8 부트 진입 조건

### Part 3: Flash 프로그래밍 (4개)

- [ ] #9 Flash Unlock/Lock
- [ ] #10 Page Erase
- [ ] #11 Half-Word Program
- [ ] #12 에러 처리

### Part 4: CAN IAP (4개)

- [ ] #13 프로토콜 설계
- [ ] #14 상태 머신 구현
- [ ] #15 데이터 수신 및 쓰기
- [ ] #16 CRC 검증

### Part 5: 업로더 & 마무리 (4개)

- [ ] #17 Python 업로더
- [ ] #18 테스트 및 검증
- [ ] #19 보안 고려사항
- [ ] #20 트러블슈팅 모음

---

## Ghidra 시리즈와 차이점

| 항목 | Ghidra 시리즈 | 이 시리즈 |
|------|--------------|----------|
| 관점 | 역분석 (리버싱) | 구현 (처음부터) |
| 시작점 | 바이너리 | 빈 프로젝트 |
| 목표 | 프로토콜 알아내기 | 직접 만들기 |
| 도구 | Ghidra, PCAN | CubeIDE, HAL |

---

## 진행 상황

```
스크린샷         0/6
글 작성          0/20
```

---

## 메모

- BMS, Ghidra, FloorPlanner 완료 (95개)
- 이 시리즈 끝나면 총 115개
