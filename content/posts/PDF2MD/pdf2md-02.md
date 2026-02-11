---
title: "PDF2MD 개발기 #2 - PDF.js로 브라우저에서 PDF 렌더링"
date: 2026-01-21
tags: ["Vue", "PDF.js", "Canvas", "WASM"]
categories: ["개발기"]
series: ["PDF2MD 개발기"]
summary: "PDF.js 연동하다가 워커 때문에 삽질했다."
---

## PDF.js란

Mozilla에서 만든 PDF 렌더링 라이브러리다. 브라우저의 Canvas API를 써서 PDF를 그린다.

Firefox의 기본 PDF 뷰어가 이걸로 만들어졌다. 꽤 안정적이고 기능도 많다.

## 설치

```bash
npm install pdfjs-dist
```

버전 주의. 4.x부터 ES 모듈만 지원한다. 구버전 프로젝트면 3.x 쓰는 게 편할 수 있다.

## 기본 사용법

```javascript
import * as pdfjsLib from 'pdfjs-dist'

// 워커 설정 (중요!)
pdfjsLib.GlobalWorkerOptions.workerSrc = 
  `//cdnjs.cloudflare.com/ajax/libs/pdf.js/${pdfjsLib.version}/pdf.worker.min.js`

// PDF 로드
const loadingTask = pdfjsLib.getDocument(pdfUrl)
const pdf = await loadingTask.promise

// 페이지 가져오기
const page = await pdf.getPage(1)

// 캔버스에 렌더링
const scale = 1.5
const viewport = page.getViewport({ scale })

const canvas = document.getElementById('pdf-canvas')
const context = canvas.getContext('2d')
canvas.height = viewport.height
canvas.width = viewport.width

await page.render({
  canvasContext: context,
  viewport: viewport
}).promise
```

간단해 보이지만 함정이 있다.

## 워커 설정 삽질

PDF.js는 무거운 파싱 작업을 Web Worker에서 처리한다. 워커 설정을 안 하면:

```
Warning: Setting up fake worker.
```

가짜 워커로 돌아가긴 하는데, 메인 스레드에서 처리하니까 UI가 멈춘다. 큰 PDF 열면 브라우저가 얼어버림.

### 방법 1: CDN

```javascript
pdfjsLib.GlobalWorkerOptions.workerSrc = 
  `//cdnjs.cloudflare.com/ajax/libs/pdf.js/${pdfjsLib.version}/pdf.worker.min.js`
```

간단하지만 오프라인에서 안 됨.

### 방법 2: 로컬 복사

```javascript
import workerSrc from 'pdfjs-dist/build/pdf.worker.min.mjs?url'
pdfjsLib.GlobalWorkerOptions.workerSrc = workerSrc
```

Vite에서 `?url` 붙이면 파일 경로를 가져온다. 빌드할 때 assets 폴더에 복사됨.

### 방법 3: 번들에 포함 (비추)

```javascript
import PDFWorker from 'pdfjs-dist/build/pdf.worker.min.mjs?worker'
pdfjsLib.GlobalWorkerOptions.workerPort = new PDFWorker()
```

워커를 번들에 포함시키는 방법. 번들 크기가 커지고 캐싱 효율이 떨어진다.

나는 **방법 2**를 썼다. 배포 환경에서 경로만 잘 맞추면 문제없다.

## Vue 컴포넌트로 감싸기

```vue
<template>
  <div class="pdf-viewer">
    <canvas ref="canvasRef"></canvas>
    <div class="controls">
      <button @click="prevPage" :disabled="currentPage <= 1">이전</button>
      <span>{{ currentPage }} / {{ totalPages }}</span>
      <button @click="nextPage" :disabled="currentPage >= totalPages">다음</button>
    </div>
  </div>
</template>

<script setup>
import { ref, onMounted, watch } from 'vue'
import * as pdfjsLib from 'pdfjs-dist'
import workerSrc from 'pdfjs-dist/build/pdf.worker.min.mjs?url'

pdfjsLib.GlobalWorkerOptions.workerSrc = workerSrc

const props = defineProps({
  file: File
})

const canvasRef = ref(null)
const pdfDoc = ref(null)
const currentPage = ref(1)
const totalPages = ref(0)
const scale = ref(1.5)

// PDF 로드
async function loadPdf(file) {
  const arrayBuffer = await file.arrayBuffer()
  const loadingTask = pdfjsLib.getDocument({ data: arrayBuffer })
  pdfDoc.value = await loadingTask.promise
  totalPages.value = pdfDoc.value.numPages
  await renderPage(1)
}

// 페이지 렌더링
async function renderPage(pageNum) {
  if (!pdfDoc.value) return
  
  const page = await pdfDoc.value.getPage(pageNum)
  const viewport = page.getViewport({ scale: scale.value })
  
  const canvas = canvasRef.value
  const context = canvas.getContext('2d')
  canvas.height = viewport.height
  canvas.width = viewport.width
  
  await page.render({
    canvasContext: context,
    viewport: viewport
  }).promise
  
  currentPage.value = pageNum
}

// 페이지 이동
function prevPage() {
  if (currentPage.value > 1) {
    renderPage(currentPage.value - 1)
  }
}

function nextPage() {
  if (currentPage.value < totalPages.value) {
    renderPage(currentPage.value + 1)
  }
}

// 파일 변경 감지
watch(() => props.file, (newFile) => {
  if (newFile) loadPdf(newFile)
})

onMounted(() => {
  if (props.file) loadPdf(props.file)
})
</script>
```

## 줌 기능

scale 값만 바꾸고 다시 렌더링하면 된다:

```javascript
function zoomIn() {
  scale.value = Math.min(scale.value + 0.25, 3)
  renderPage(currentPage.value)
}

function zoomOut() {
  scale.value = Math.max(scale.value - 0.25, 0.5)
  renderPage(currentPage.value)
}
```

## 메모리 누수 주의

페이지 객체는 사용 후 정리해줘야 한다:

```javascript
let currentPageObj = null

async function renderPage(pageNum) {
  // 이전 페이지 정리
  if (currentPageObj) {
    currentPageObj.cleanup()
  }
  
  currentPageObj = await pdfDoc.value.getPage(pageNum)
  // ... 렌더링
}

// 컴포넌트 언마운트 시
onUnmounted(() => {
  if (currentPageObj) currentPageObj.cleanup()
  if (pdfDoc.value) pdfDoc.value.destroy()
})
```

안 하면 큰 PDF 여러 번 열었다 닫았다 하면 메모리가 계속 쌓인다.

## 배포 시 경로 문제

Vite로 빌드하면 assets에 해시가 붙은 파일명이 생긴다:

```
/assets/pdf.worker.min-yatZIOMy.mjs
```

Hugo 같은 정적 사이트에서 서브 경로로 배포하면 경로가 안 맞을 수 있다:

```javascript
// 잘못된 경로
"/assets/pdf.worker.min-yatZIOMy.mjs"

// 서브 경로 배포 시 필요한 경로
"/pdf2md/assets/pdf.worker.min-yatZIOMy.mjs"
```

`vite.config.js`에서 `base` 옵션 설정하거나, 빌드 후 경로를 수정해야 한다.
![](pdf2md-02-1.png)
## 다음 글에서

PDF 위에 영역을 선택하는 기능을 구현한다. 드래그 대신 클릭-투-클릭 방식을 선택한 이유와 구현 방법을 다룬다.
