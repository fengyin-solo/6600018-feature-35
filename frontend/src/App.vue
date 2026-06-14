<template>
  <div class="flex h-screen">
    <!-- Left: Document list -->
    <div class="w-64 bg-gray-900 p-4 flex flex-col gap-3 border-r border-gray-800">
      <h1 class="text-lg font-bold text-amber-400">古籍 OCR 标注平台</h1>

      <div>
        <label class="block bg-amber-500 text-black text-center py-2 rounded cursor-pointer hover:bg-amber-400 text-sm font-medium">
          上传古籍图片
          <input type="file" accept="image/*" @change="onUpload" class="hidden" />
        </label>
      </div>

      <button @click="store.loadMockDocument()" class="bg-gray-800 py-2 rounded text-sm hover:bg-gray-700">
        加载示例文档
      </button>

      <!-- Search -->
      <div>
        <input v-model="store.searchQuery" @input="store.searchInDocuments(store.searchQuery)"
          placeholder="全文检索..." class="w-full bg-gray-800 rounded px-3 py-2 text-sm" />
        <div v-if="store.searchResults.length" class="mt-1 space-y-1">
          <div v-for="r in store.searchResults" :key="r.id" class="bg-gray-800 rounded p-1 text-xs">
            {{ r.text }} <span class="text-gray-500">{{ (r.confidence * 100).toFixed(0) }}%</span>
          </div>
        </div>
      </div>

      <!-- Document list -->
      <div class="flex-1 overflow-y-auto space-y-1">
        <div v-for="d in store.documents" :key="d.id" @click="store.currentDoc = d"
          class="bg-gray-800 rounded p-2 cursor-pointer text-sm"
          :class="store.currentDoc?.id === d.id ? 'ring-1 ring-amber-500' : ''">
          {{ d.name }}
          <div class="text-xs text-gray-500">{{ d.results.length }} 行识别</div>
        </div>
      </div>

      <!-- Export -->
      <button @click="doExport" class="bg-green-700 py-2 rounded text-sm hover:bg-green-600">
        导出 TEI/XML
      </button>
    </div>

    <!-- Center: Image + OCR overlay -->
    <div class="flex-1 relative bg-gray-950 overflow-hidden">
      <ImageCanvas v-if="store.currentDoc" />
      <div v-else class="flex items-center justify-center h-full text-gray-600">
        请上传古籍图片或加载示例文档
      </div>
    </div>

    <!-- Right: OCR results & annotations -->
    <div class="w-80 bg-gray-900 p-4 flex flex-col gap-3 border-l border-gray-800 overflow-y-auto">
      <div class="flex items-center justify-between">
        <h3 class="text-amber-300 font-bold text-sm">OCR 识别结果</h3>
        <div class="flex gap-1" v-if="store.currentDoc">
          <button @click="store.expandAll()" class="text-xs text-gray-400 hover:text-amber-300 px-1">全部展开</button>
          <button @click="store.collapseAll()" class="text-xs text-gray-400 hover:text-amber-300 px-1">全部折叠</button>
        </div>
      </div>
      <div v-if="store.currentDoc" class="space-y-2">
        <div v-for="[pIndex, items] in store.paragraphGroups" :key="pIndex"
          class="border border-gray-700 rounded overflow-hidden">
          <div class="flex items-center justify-between bg-gray-800 px-3 py-2 cursor-pointer select-none hover:bg-gray-700 transition-colors"
            @click="store.toggleParagraph(pIndex)">
            <div class="flex items-center gap-2">
              <svg class="w-4 h-4 text-gray-400 transition-transform duration-200"
                :class="store.collapsedParagraphs.has(pIndex) ? '' : 'rotate-90'"
                fill="none" stroke="currentColor" viewBox="0 0 24 24">
                <path stroke-linecap="round" stroke-linejoin="round" stroke-width="2" d="M9 5l7 7-7 7"/>
              </svg>
              <span class="text-amber-200 text-xs font-medium">段落 {{ pIndex + 1 }}</span>
              <span class="text-gray-500 text-xs">({{ items.length }} 行)</span>
              <span v-if="store.getParagraphStats(items).lowConfCount > 0"
                class="text-xs bg-yellow-900 text-yellow-400 px-1.5 py-0.5 rounded">
                {{ store.getParagraphStats(items).lowConfCount }} 待校对
              </span>
              <span v-if="store.getParagraphStats(items).correctedCount > 0"
                class="text-xs bg-green-900 text-green-400 px-1.5 py-0.5 rounded">
                {{ store.getParagraphStats(items).correctedCount }} 已校
              </span>
            </div>
            <div class="flex items-center gap-2">
              <button class="text-xs text-blue-400 hover:text-blue-300 hover:bg-blue-900/30 px-2 py-0.5 rounded transition-colors"
                @click.stop="focusCorrection(findFirstLowConfOrFirst(items))">校对</button>
              <button class="text-xs text-green-400 hover:text-green-300 hover:bg-green-900/30 px-2 py-0.5 rounded transition-colors"
                @click.stop="addParagraphAnnotation(pIndex)">标注</button>
            </div>
          </div>
          <div v-show="!store.collapsedParagraphs.has(pIndex)" class="p-2 space-y-2 bg-gray-900">
            <div v-for="r in items" :key="r.id"
              class="bg-gray-800 rounded p-2 text-sm" :id="'result-' + r.id">
              <div class="flex justify-between">
                <span class="text-white font-medium">{{ r.text }}</span>
                <span class="text-xs px-2 py-0.5 rounded"
                  :class="r.confidence > 0.9 ? 'bg-green-900 text-green-400' : 'bg-yellow-900 text-yellow-400'">
                  {{ (r.confidence * 100).toFixed(0) }}%
                </span>
              </div>
              <div class="text-xs text-gray-400 mt-1">
                简体: {{ store.convertVariant(r.text) }}
              </div>
              <input v-model="r.corrected" placeholder="人工校正..."
                class="w-full bg-gray-700 rounded px-2 py-1 text-xs mt-1 focus:ring-1 focus:ring-amber-500 focus:outline-none" />
            </div>
          </div>
          <div v-show="store.collapsedParagraphs.has(pIndex)" class="px-3 py-2 bg-gray-800/50 border-t border-gray-700">
            <p class="text-gray-400 text-xs truncate">{{ items.map(r => r.text).join(' ') }}</p>
          </div>
        </div>
      </div>

      <h3 class="text-amber-300 font-bold text-sm mt-4">标注列表</h3>
      <div v-if="store.currentDoc" class="space-y-1">
        <div v-for="a in store.currentDoc.annotations" :key="a.id"
          class="bg-gray-800 rounded p-2 text-xs flex justify-between">
          <span>[{{ a.type }}] {{ a.label }}: {{ a.content }}</span>
          <button @click="store.removeAnnotation(a.id)" class="text-red-400 hover:underline">删除</button>
        </div>
        <div v-if="!store.currentDoc.annotations.length" class="text-gray-600 text-xs">
          在图片上拖拽框选区域添加标注
        </div>
      </div>
    </div>
  </div>
</template>

<script setup lang="ts">
import { useOcrStore } from './store/ocr'
import ImageCanvas from './components/ImageCanvas.vue'

const store = useOcrStore()

function findFirstLowConfOrFirst(items: any[]) {
  if (!items || !items.length) return undefined
  const lowConf = items.find(r => r.confidence < 0.9)
  return lowConf ? lowConf.id : items[0].id
}

function onUpload(e: Event) {
  const file = (e.target as HTMLInputElement).files?.[0]
  if (file) store.uploadAndOCR(file)
}

function focusCorrection(resultId?: string) {
  if (resultId === undefined) return
  const pIndex = store.currentDoc?.results.find(r => r.id === resultId)?.paragraph
  if (pIndex !== undefined && store.collapsedParagraphs.has(pIndex)) {
    store.toggleParagraph(pIndex)
  }
  setTimeout(() => {
    const el = document.getElementById('result-' + resultId)
    if (el) {
      el.scrollIntoView({ behavior: 'smooth', block: 'center' })
      const input = el.querySelector('input')
      if (input) input.focus()
    }
  }, 50)
}

function addParagraphAnnotation(pIndex: number) {
  const items = store.paragraphGroups.get(pIndex)
  if (!items || !items.length) return
  const label = prompt('标注标签（如：章节/段落/异体字）') || 'region'
  const content = prompt('标注内容') || items.map(r => r.text).join('')
  const firstBbox = items[0].bbox
  const annotationBbox: [number, number, number, number] = [
    firstBbox[0], firstBbox[1],
    firstBbox[2],
    items.reduce((sum, r) => sum + r.bbox[3], 0)
  ]
  store.addAnnotation('region', annotationBbox, label, content)
}

function doExport() {
  const tei = store.exportTEI()
  if (!tei) return
  const blob = new Blob([tei], { type: 'application/xml' })
  const url = URL.createObjectURL(blob)
  const a = document.createElement('a')
  a.href = url
  a.download = `${store.currentDoc?.name || 'export'}.xml`
  a.click()
  URL.revokeObjectURL(url)
}
</script>
