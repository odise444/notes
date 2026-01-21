---
title: "PDF2MD 개발기 #4 - Tesseract.js OCR로 Figure/Table 인식"
date: 2026-01-21
tags: ["Tesseract", "OCR", "JavaScript", "WASM"]
categories: ["개발기"]
series: ["PDF2MD 개발기"]
summary: "이미지에서 텍스트 뽑아서 Figure인지 Table인지 자동 판별."
---

## Tesseract.js란

Google의 Tesseract OCR 엔진을 WASM으로 포팅한 것. 브라우저에서 서버 없이 OCR이 가능하다.

정확도가 Google Vision API만큼은 아니지만, 데이터시트 같은 깔끔한 문서에서는 충분히 쓸만하다.

## 설치

```bash
npm install tesseract.js
```

첫 실행 시 언어 데이터를 다운로드한다. 영어 기준 약 10MB.

## 기본 사용법

```javascript
import Tesseract from 'tesseract.js'

const result = await Tesseract.recognize(
  imageSource,  // 이미지 URL, File, Blob, Canvas 등
  'eng',        // 언어
  {
    logger: m => console.log(m)  // 진행 상황
  }
)

console.log(result.data.text)
```

## 서비스로 감싸기

매번 워커를 생성하면 느리다. 워커를 재사용하는 서비스를 만들자:

```javascript
// services/ocrService.js
import { createWorker } from 'tesseract.js'

let worker = null

export async function initOCR() {
  if (worker) return worker
  
  worker = await createWorker('eng', 1, {
    logger: m => {
      if (m.status === 'recognizing text') {
        console.log(`OCR 진행: ${Math.round(m.progress * 100)}%`)
      }
    }
  })
  
  return worker
}

export async function recognizeText(imageSource) {
  const w = await initOCR()
  const result = await w.recognize(imageSource)
  return result.data.text
}

export async function terminateOCR() {
  if (worker) {
    await worker.terminate()
    worker = null
  }
}
```

## Figure/Table 인식 로직

OCR로 텍스트를 뽑은 후, 정규식으로 패턴을 찾는다:

```javascript
export function parseRegionType(ocrText) {
  const text = ocrText.toLowerCase().trim()
  
  // Figure 패턴
  const figureMatch = text.match(/fig(?:ure)?\.?\s*(\d+)/i)
  if (figureMatch) {
    return {
      type: 'Figure',
      number: figureMatch[1],
      confidence: 'high'
    }
  }
  
  // Table 패턴
  const tableMatch = text.match(/table\.?\s*(\d+)/i)
  if (tableMatch) {
    return {
      type: 'Table',
      number: tableMatch[1],
      confidence: 'high'
    }
  }
  
  // 애매한 경우
  if (text.includes('fig')) {
    return { type: 'Figure', number: null, confidence: 'low' }
  }
  if (text.includes('table')) {
    return { type: 'Table', number: null, confidence: 'low' }
  }
  
  return { type: 'Unknown', number: null, confidence: 'none' }
}
```

## 캡션 추출

Figure 번호 뒤에 오는 텍스트가 보통 캡션이다:

```javascript
export function extractCaption(ocrText) {
  // "Figure 1. Block Diagram" 에서 "Block Diagram" 추출
  const captionMatch = ocrText.match(/(?:fig(?:ure)?|table)\.?\s*\d+[.:]\s*(.+)/i)
  
  if (captionMatch) {
    return captionMatch[1].trim()
  }
  
  return null
}
```

## 전체 인식 플로우

```javascript
export async function analyzeRegion(imageBlob) {
  // OCR 실행
  const text = await recognizeText(imageBlob)
  
  // 타입 파싱
  const typeInfo = parseRegionType(text)
  
  // 캡션 추출
  const caption = extractCaption(text)
  
  // 파일명 생성
  const filename = generateFilename(typeInfo)
  
  return {
    ocrText: text,
    type: typeInfo.type,
    number: typeInfo.number,
    caption: caption,
    filename: filename,
    confidence: typeInfo.confidence
  }
}

function generateFilename(typeInfo) {
  if (typeInfo.type === 'Unknown') {
    return `image_${Date.now()}`
  }
  
  const num = typeInfo.number || 'x'
  return `${typeInfo.type.toLowerCase()}_${num}`
}
```

## Vue 컴포넌트에서 사용

```vue
<script setup>
import { ref } from 'vue'
import { analyzeRegion } from '@/services/ocrService'

const regions = ref([])
const isAnalyzing = ref(false)

async function handleCapture(captureData) {
  isAnalyzing.value = true
  
  try {
    // OCR 분석
    const analysis = await analyzeRegion(captureData.blob)
    
    // 영역 데이터 저장
    regions.value.push({
      id: Date.now(),
      imageUrl: captureData.imageUrl,
      blob: captureData.blob,
      rect: captureData.rect,
      page: captureData.page,
      ...analysis
    })
  } finally {
    isAnalyzing.value = false
  }
}
</script>

<template>
  <div v-if="isAnalyzing" class="analyzing">
    분석 중...
  </div>
</template>
```

## OCR 성능 최적화

### 1. 이미지 전처리

OCR 전에 이미지를 흑백으로 변환하면 정확도가 올라간다:

```javascript
function preprocessImage(canvas) {
  const ctx = canvas.getContext('2d')
  const imageData = ctx.getImageData(0, 0, canvas.width, canvas.height)
  const data = imageData.data
  
  // 그레이스케일 변환
  for (let i = 0; i < data.length; i += 4) {
    const gray = data[i] * 0.299 + data[i + 1] * 0.587 + data[i + 2] * 0.114
    data[i] = data[i + 1] = data[i + 2] = gray
  }
  
  ctx.putImageData(imageData, 0, 0)
  return canvas
}
```

### 2. 영역 제한

전체 이미지 대신 텍스트가 있을 것 같은 영역만 OCR:

```javascript
// 상단 20%만 OCR (Figure/Table 레이블은 보통 위에 있음)
const partialHeight = Math.min(100, canvas.height * 0.2)
const partialCanvas = document.createElement('canvas')
partialCanvas.width = canvas.width
partialCanvas.height = partialHeight

const ctx = partialCanvas.getContext('2d')
ctx.drawImage(canvas, 0, 0, canvas.width, partialHeight, 0, 0, canvas.width, partialHeight)

const text = await recognizeText(partialCanvas)
```

### 3. 워커 풀

여러 영역을 동시에 분석하려면 워커 풀을 사용:

```javascript
import { createScheduler, createWorker } from 'tesseract.js'

const scheduler = createScheduler()

// 워커 2개 생성
for (let i = 0; i < 2; i++) {
  const worker = await createWorker('eng')
  scheduler.addWorker(worker)
}

// 병렬 처리
const results = await Promise.all(
  images.map(img => scheduler.addJob('recognize', img))
)
```

## 한글 인식

한글 데이터시트도 있다면:

```javascript
const worker = await createWorker('eng+kor')
```

다만 언어 데이터가 추가로 필요하고, 인식 속도가 느려진다.

## 다음 글에서

분석한 영역들을 IndexedDB에 저장하고, 최종적으로 Markdown + 이미지를 ZIP으로 내보내는 기능을 구현한다.
