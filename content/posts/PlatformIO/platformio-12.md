---
title: "PlatformIO 완벽 가이드 #12 - 코드 포맷팅"
date: 2024-12-22
tags: ["PlatformIO", "포맷팅", "clang-format", "코드스타일"]
categories: ["개발환경"]
series: ["PlatformIO 완벽 가이드"]
summary: "코드 정리를 자동으로."
---

들여쓰기 일일이 맞추기 귀찮다.

자동으로 정리해주는 포맷터 설정하자.

---

## 포맷팅이란

Before:
```cpp
void setup(){
Serial.begin(115200);
  if(true){
    Serial.println("Hello");
}}
```

After:
```cpp
void setup() {
    Serial.begin(115200);
    if (true) {
        Serial.println("Hello");
    }
}
```

들여쓰기, 괄호 위치, 공백 자동 정리.

---

## 포맷터 선택

VSCode C/C++ 확장에 **clang-format** 포함됨.

별도 설치 필요 없음.

---

## 포맷 실행

### 전체 파일

`Shift + Alt + F`

### 선택 영역만

코드 선택 후 `Ctrl + K`, `Ctrl + F`

---

## 저장 시 자동 포맷

VSCode 설정:

`Ctrl + ,` (설정 열기)

검색: "format on save"

☑️ **Editor: Format On Save** 체크

또는 `settings.json`:

```json
{
    "editor.formatOnSave": true
}
```

이제 저장할 때마다 자동 정리.

---

## 포맷 스타일 설정

### 내장 스타일

```json
{
    "C_Cpp.clang_format_fallbackStyle": "LLVM"
}
```

옵션:
- `LLVM` - LLVM 스타일
- `Google` - Google 스타일
- `Chromium` - Chromium 스타일
- `Mozilla` - Mozilla 스타일
- `WebKit` - WebKit 스타일

---

## 커스텀 스타일 (.clang-format)

프로젝트 루트에 `.clang-format` 파일 생성:

```yaml
BasedOnStyle: Google
IndentWidth: 4
ColumnLimit: 100
BreakBeforeBraces: Attach
AllowShortFunctionsOnASingleLine: Empty
```

### 주요 옵션

| 옵션 | 설명 | 예시 값 |
|------|------|---------|
| `IndentWidth` | 들여쓰기 칸 수 | `4` |
| `TabWidth` | 탭 너비 | `4` |
| `UseTab` | 탭 사용 여부 | `Never` |
| `ColumnLimit` | 한 줄 최대 길이 | `100` |
| `BreakBeforeBraces` | 중괄호 위치 | `Attach`, `Allman` |

---

## 중괄호 스타일

### Attach (K&R)

```cpp
void setup() {
    // ...
}
```

### Allman

```cpp
void setup()
{
    // ...
}
```

설정:
```yaml
BreakBeforeBraces: Allman
```

---

## 포인터 정렬

```yaml
PointerAlignment: Left   # int* ptr
PointerAlignment: Right  # int *ptr
PointerAlignment: Middle # int * ptr
```

---

## 내 .clang-format 예시

```yaml
# 프로젝트 루트에 저장: .clang-format
BasedOnStyle: Google

# 들여쓰기
IndentWidth: 4
TabWidth: 4
UseTab: Never

# 줄 길이
ColumnLimit: 100

# 중괄호
BreakBeforeBraces: Attach
AllowShortIfStatementsOnASingleLine: false
AllowShortLoopsOnASingleLine: false

# 정렬
AlignConsecutiveAssignments: true
AlignConsecutiveDeclarations: true

# 포인터
PointerAlignment: Left
```

---

## 특정 코드 포맷 제외

포맷하고 싶지 않은 부분:

```cpp
// clang-format off
const int lookup_table[] = {
    1,  2,  3,  4,
    5,  6,  7,  8,
    9, 10, 11, 12
};
// clang-format on
```

`// clang-format off`와 `// clang-format on` 사이는 건드리지 않음.

---

## EditorConfig

`.editorconfig` 파일로 기본 설정:

```ini
root = true

[*]
indent_style = space
indent_size = 4
end_of_line = lf
charset = utf-8
trim_trailing_whitespace = true
insert_final_newline = true
```

팀 전체 통일할 때 유용.

---

## 다음 단계

코드 포맷팅 설정 완료!

다음 글에서 멀티보드 설정.

---

다음 글: [#13 - 여러 보드 동시 설정](/posts/platformio/platformio-13/)
