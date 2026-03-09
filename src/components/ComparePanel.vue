<script setup lang="ts">
import { computed, onBeforeUnmount, onMounted, ref, watch } from 'vue'

type WeightKey = 'structure' | 'color' | 'texture'

interface WeightItem {
  key: WeightKey
  label: string
  value: number
  tone?: 'pink' | 'mint' | 'blue'
}

interface SearchItem {
  path: string
  is_representative: boolean
  label: string
  score: number
  radius: number
  angle: number
}

interface OrbitPoint {
  x: number
  y: number
  raw: SearchItem
}

interface RepresentativeNode {
  x: number
  y: number
  raw: SearchItem
  imageUrl: string
}

interface Props {
  title?: string
  promptPlaceholder?: string
  weights?: WeightItem[]
  referenceImage?: string
  galleryImages?: string[]
  searchApiBase?: string
  searchFile?: File | null
}

const props = withDefaults(defineProps<Props>(), {
  title: 'Compare',
  promptPlaceholder: 'Describe the style, palette, and mood',
  weights: () => [
    { key: 'structure', label: 'Structure', value: 1, tone: 'mint' },
    { key: 'color', label: 'Color', value: 0.3, tone: 'blue' },
    { key: 'texture', label: 'Texture', value: 0.2, tone: 'pink' }
  ],
  referenceImage: '',
  galleryImages: () => [],
  searchApiBase: 'http://127.0.0.1:8001',
  searchFile: null
})

const weightRows = ref<WeightItem[]>(
  props.weights.map((weight) => ({ ...weight, value: clamp(weight.value, 0, 1) }))
)
const searchResults = ref<SearchItem[]>([])
const isSearching = ref(false)
const searchError = ref('')
const hoverItem = ref<SearchItem | null>(null)
const pinnedItem = ref<SearchItem | null>(null)
const galleryUploads = ref<string[]>([])
const orbitRef = ref<HTMLDivElement | null>(null)
const orbitScale = ref(1)
const orbitOffsetX = ref(0)
const orbitOffsetY = ref(0)
const isPanning = ref(false)
const shadowRadius = ref(4)

let panStartX = 0
let panStartY = 0
let panOriginX = 0
let panOriginY = 0

let debounceTimer: ReturnType<typeof setTimeout> | null = null
let activeController: AbortController | null = null
let orbitResizeObserver: ResizeObserver | null = null

function updateShadowRadius() {
  if (!orbitRef.value) return
  const size = Math.min(orbitRef.value.clientWidth, orbitRef.value.clientHeight)
  if (size <= 0) return

  // 代表圈直径是 44px，这里将其换算为 viewBox(0-100) 的半径单位。
  const r = (22 * 100) / size
  shadowRadius.value = clamp(r, 1.5, 10)
}

function spreadRepresentativeNodes(nodes: RepresentativeNode[]) {
  if (nodes.length < 2) return nodes

  // 只处理严重拥挤，不追求等距分布。
  const collisionDist = 6.2
  const left = 6
  const right = 94
  const top = 6
  const bottom = 94
  const iterations = 10
  const maxShift = 4.5

  const original = nodes.map((node) => ({ x: node.x, y: node.y }))
  const pos = nodes.map((node) => ({ x: node.x, y: node.y }))

  for (let iter = 0; iter < iterations; iter += 1) {
    for (let i = 0; i < pos.length; i += 1) {
      const pi = pos[i]!
      for (let j = i + 1; j < pos.length; j += 1) {
        const pj = pos[j]!
        const dx = pj.x - pi.x
        const dy = pj.y - pi.y
        const d2 = dx * dx + dy * dy

        if (d2 >= collisionDist * collisionDist) continue

        let dist = Math.sqrt(d2)
        if (dist < 1e-6) {
          const jitter = ((i + 1) * (j + 3)) % 7
          dist = 0.001
          pj.x += (jitter - 3) * 0.12
          pj.y += (3 - jitter) * 0.12
        }

        // 推开量只覆盖“重叠部分”，并故意保守，避免排成均匀圆环。
        const overlap = collisionDist - dist
        const softness = 0.35 + 0.35 * (iter / Math.max(1, iterations - 1))
        const pushX = (dx / dist) * overlap * softness * 0.5
        const pushY = (dy / dist) * overlap * softness * 0.5

        pi.x -= pushX
        pi.y -= pushY
        pj.x += pushX
        pj.y += pushY
      }
    }

    // 轻微回弹，保持整体布局语义不被过度破坏。
    for (let i = 0; i < pos.length; i += 1) {
      const pi = pos[i]!
      const oi = original[i]!
      // 强回弹：尽量贴近后端原始位置，只做局部冲突修复。
      pi.x += (oi.x - pi.x) * 0.24
      pi.y += (oi.y - pi.y) * 0.24

      const offX = pi.x - oi.x
      const offY = pi.y - oi.y
      const offLen = Math.sqrt(offX * offX + offY * offY)
      if (offLen > maxShift) {
        const k = maxShift / offLen
        pi.x = oi.x + offX * k
        pi.y = oi.y + offY * k
      }

      pi.x = clamp(pi.x, left, right)
      pi.y = clamp(pi.y, top, bottom)
    }
  }

  return nodes.map((node, idx) => ({
    ...node,
    x: pos[idx]!.x,
    y: pos[idx]!.y
  }))
}

const representativeNodes = computed<RepresentativeNode[]>(() =>
  spreadRepresentativeNodes(
    searchResults.value
      .filter((item) => item.is_representative)
      .map((item) => {
        const point = toOrbitPoint(item)
        return {
          ...point,
          imageUrl: resolveImageUrl(item.path)
        }
      })
  )
)

const MAX_SHADOW_POINTS = 300

const shadowPoints = computed(() => {
  const members = searchResults.value
    .filter((item) => !item.is_representative)
    .map((item) => toOrbitPoint(item))

  if (!members.length) {
    return []
  }

  const reps = representativeNodes.value
  if (!reps.length) {
    return members.slice(0, MAX_SHADOW_POINTS)
  }

  const withDistance = members.map((point) => {
    let nearestDistance = Number.POSITIVE_INFINITY
    for (const rep of reps) {
      const dx = point.x - rep.x
      const dy = point.y - rep.y
      const dist = dx * dx + dy * dy
      if (dist < nearestDistance) {
        nearestDistance = dist
      }
    }
    return {
      point,
      nearestDistance
    }
  })

  withDistance.sort((a, b) => a.nearestDistance - b.nearestDistance)
  return withDistance.slice(0, MAX_SHADOW_POINTS).map((item) => item.point)
})

const activePreviewItem = computed(() => pinnedItem.value ?? hoverItem.value)

const orbitCenterImageUrl = computed(() => props.referenceImage || '')

const displayImageUrl = computed(() => {
  if (activePreviewItem.value) {
    return resolveImageUrl(activePreviewItem.value.path)
  }
  if (props.referenceImage) return props.referenceImage
  const first = searchResults.value[0]
  if (!first) return ''
  return resolveImageUrl(first.path)
})

const galleryResultImages = computed(() => galleryUploads.value)

const arcStroke = computed(() => ({
  structure: String(getWeightValue('structure')),
  color: String(getWeightValue('color')),
  texture: String(getWeightValue('texture'))
}))

const orbitSceneStyle = computed(() => ({
  transform: `translate(${orbitOffsetX.value}px, ${orbitOffsetY.value}px) scale(${orbitScale.value})`
}))

watch(
  () => props.searchFile,
  (file) => {
    if (!file) {
      searchResults.value = []
      searchError.value = ''
      hoverItem.value = null
      pinnedItem.value = null
      return
    }
    queueSearch()
  },
  { immediate: true }
)

onBeforeUnmount(() => {
  if (debounceTimer) {
    clearTimeout(debounceTimer)
  }
  if (activeController) {
    activeController.abort()
  }
  if (orbitResizeObserver) {
    orbitResizeObserver.disconnect()
    orbitResizeObserver = null
  }
  galleryUploads.value.forEach((url) => URL.revokeObjectURL(url))
})

onMounted(() => {
  updateShadowRadius()
  if (orbitRef.value) {
    orbitResizeObserver = new ResizeObserver(() => {
      updateShadowRadius()
    })
    orbitResizeObserver.observe(orbitRef.value)
  }
})

function queueSearch() {
  if (debounceTimer) {
    clearTimeout(debounceTimer)
  }
  debounceTimer = setTimeout(() => {
    runSearch()
  }, 300)
}

function onWeightInput(weight: WeightItem) {
  weight.value = clamp(weight.value, 0, 1)
  if (!props.searchFile) return
  queueSearch()
}

function onRepresentativeEnter(item: SearchItem) {
  if (pinnedItem.value) return
  hoverItem.value = item
}

function onRepresentativeLeave() {
  if (pinnedItem.value) return
  hoverItem.value = null
}

function onRepresentativeClick(item: SearchItem) {
  pinnedItem.value = item
}

function clearPinnedPreview() {
  pinnedItem.value = null
  hoverItem.value = null
}

function addPinnedToGallery() {
  if (!pinnedItem.value) return
  galleryUploads.value.unshift(resolveImageUrl(pinnedItem.value.path))
}

function onOrbitWheel(event: WheelEvent) {
  if (!orbitRef.value) return

  const factor = event.deltaY < 0 ? 1.1 : 0.9
  orbitScale.value = clamp(orbitScale.value * factor, 0.6, 3)
}

function onOrbitMouseDown(event: MouseEvent) {
  if (event.button !== 0) return
  isPanning.value = true
  panStartX = event.clientX
  panStartY = event.clientY
  panOriginX = orbitOffsetX.value
  panOriginY = orbitOffsetY.value
}

function onOrbitMouseMove(event: MouseEvent) {
  if (!isPanning.value) return
  orbitOffsetX.value = panOriginX + (event.clientX - panStartX)
  orbitOffsetY.value = panOriginY + (event.clientY - panStartY)
}

function onOrbitMouseUp() {
  isPanning.value = false
}

async function runSearch() {
  if (!props.searchFile) return

  if (activeController) {
    activeController.abort()
  }

  const controller = new AbortController()
  activeController = controller
  isSearching.value = true
  searchError.value = ''
  hoverItem.value = null

  const formData = new FormData()
  formData.append('file', props.searchFile)
  formData.append('structure_w', String(getWeightValue('structure')))
  formData.append('color_w', String(getWeightValue('color')))
  formData.append('texture_w', String(getWeightValue('texture')))
  formData.append('nearby_k', '50')
  formData.append('max_total', '2000')

  try {
    console.log('Sending search request with weights:', {
      structure: getWeightValue('structure'),
      color: getWeightValue('color'),
      texture: getWeightValue('texture')
    })
    const response = await fetch(`${props.searchApiBase}/photo_search`, {
      method: 'POST',
      body: formData,
      signal: controller.signal
    })

    if (!response.ok) {
      throw new Error(await response.text())
    }

    const data = (await response.json()) as SearchItem[]
    console.log('Search results:', data)
    searchResults.value = data

    if (pinnedItem.value) {
      const stillExists = data.some((item) => item.path === pinnedItem.value?.path)
      if (!stillExists) {
        pinnedItem.value = null
      }
    }
  } catch (error) {
    if (controller.signal.aborted) return
    searchResults.value = []
    searchError.value = `检索失败：${error instanceof Error ? error.message : '未知错误'}`
  } finally {
    if (activeController === controller) {
      activeController = null
    }
    isSearching.value = false
  }
}

function getWeightValue(key: WeightKey) {
  return weightRows.value.find((weight) => weight.key === key)?.value ?? 0
}

function toOrbitPoint(item: SearchItem): OrbitPoint {
  // 指数式放大：越外层扩张越明显，呈现等比扩大的视觉感受。
  const rawRadius = Math.max(0, item.radius)
  const radius = Math.expm1(rawRadius * 1.5) * 36
  const radian = (item.angle * Math.PI) / 180
  return {
    x: 50 + Math.cos(radian) * radius,
    y: 50 + Math.sin(radian) * radius,
    raw: item
  }
}

function resolveImageUrl(path: string) {
  if (/^https?:\/\//i.test(path)) {
    return path
  }

  const base = props.searchApiBase.replace(/\/$/, '')
  let normalized = path.replace(/\\/g, '/')

  // 兼容绝对磁盘路径，优先截取 /data/... 段用于后端静态映射。
  const dataIdx = normalized.indexOf('/data/')
  if (dataIdx >= 0) {
    normalized = normalized.slice(dataIdx)
  } else if (normalized.startsWith('./data/')) {
    normalized = normalized.slice(1)
  } else if (normalized.startsWith('data/')) {
    normalized = `/${normalized}`
  } else if (normalized.startsWith('./')) {
    normalized = normalized.slice(1)
  } else if (!normalized.startsWith('/')) {
    normalized = `/${normalized}`
  }

  // 编码中文与空格等字符，避免部分图片 404。
  const encodedPath = normalized
    .split('/')
    .map((segment, idx) => (idx === 0 ? segment : encodeURIComponent(segment)))
    .join('/')

  return `${base}${encodedPath}`
}

function clamp(value: number, min: number, max: number) {
  return Math.max(min, Math.min(max, value))
}
</script>

<template>
  <section class="panel panel-compare">
    <div class="compare-grid">
      <div class="compare-column compare-column-controls">
        <div class="prompt-card">
          <div class="prompt-header">
            <span class="prompt-search-icon" aria-hidden="true">🔍</span>
            <span class="prompt-title">Prompt</span>
          </div>
          <div class="prompt-input">
            <slot name="prompt">
              <textarea
                :placeholder="props.promptPlaceholder"
                aria-label="prompt"
                rows="4"
              ></textarea>
            </slot>
          </div>
          <div class="weights">
            <div class="weights-title">Field weights (0-1)</div>
            <slot name="fieldweights">
              <div class="weights-list">
                <div
                  v-for="weight in weightRows"
                  :key="weight.key"
                  class="weight-row"
                >
                  <span>{{ weight.label }}</span>
                  <input
                    v-model.number="weight.value"
                    class="weight-slider"
                    type="range"
                    min="0"
                    max="1"
                    step="0.01"
                    @input="onWeightInput(weight)"
                  />
                  <span class="weight-value">{{ weight.value.toFixed(2) }}</span>
                </div>
              </div>
            </slot>
          </div>
        </div>
      </div>

      <div class="compare-column">
        <slot name="compare">
          <div
            ref="orbitRef"
            class="orbit"
            @mouseleave="onRepresentativeLeave"
            @wheel.prevent="onOrbitWheel"
            @mousedown="onOrbitMouseDown"
            @mousemove="onOrbitMouseMove"
            @mouseup="onOrbitMouseUp"
            @mouseleave.self="onOrbitMouseUp"
          >
            <div v-if="!props.searchFile" class="orbit-state">
              左侧编辑完成后点击“暂存”，自动调用后端生成轨道图
            </div>
            <div v-else-if="isSearching" class="orbit-state">正在计算轨道图...</div>
            <div v-else-if="searchError" class="orbit-state orbit-state-error">{{ searchError }}</div>
            <div v-else-if="!searchResults.length" class="orbit-state">暂无检索结果</div>
            <div class="orbit-scene" :style="orbitSceneStyle">
              <svg
                v-if="searchResults.length"
                class="orbit-chart"
                viewBox="0 0 100 100"
                role="img"
                aria-label="search orbit"
              >
                <circle class="orbit-ring" cx="50" cy="50" r="15" />
                <circle class="orbit-ring" cx="50" cy="50" r="30" />
                <circle class="orbit-ring" cx="50" cy="50" r="44" />

                <circle
                  v-for="(point, index) in shadowPoints"
                  :key="`${point.raw.path}-${index}`"
                  class="orbit-shadow"
                  :cx="point.x"
                  :cy="point.y"
                  :r="shadowRadius"
                />
              </svg>

              <div v-if="orbitCenterImageUrl" class="orbit-center-image">
                <img :src="orbitCenterImageUrl" alt="center" />
              </div>

              <button
                v-for="(node, index) in representativeNodes"
                :key="`${node.raw.path}-rep-${index}`"
                type="button"
                class="rep-node"
                :style="{ left: `${node.x}%`, top: `${node.y}%` }"
                @mousedown.stop
                @mouseenter="onRepresentativeEnter(node.raw)"
                @mouseleave="onRepresentativeLeave"
                @click="onRepresentativeClick(node.raw)"
              >
                <svg class="circle-arcs circle-arcs-node" viewBox="0 0 100 100" aria-hidden="true">
                  <g transform="rotate(-130 50 50)">
                    <circle
                      class="weight-arc-track"
                      cx="50"
                      cy="50"
                      r="47"
                    />
                    <circle
                      class="weight-arc arc-structure"
                      cx="50"
                      cy="50"
                      r="47"
                      pathLength="1"
                      :stroke-dasharray="`${arcStroke.structure} 1`"
                    />
                  </g>
                  <g transform="rotate(15 50 50)">
                    <circle
                      class="weight-arc-track"
                      cx="50"
                      cy="50"
                      r="49"
                    />
                    <circle
                      class="weight-arc arc-color"
                      cx="50"
                      cy="50"
                      r="49"
                      pathLength="1"
                      :stroke-dasharray="`${arcStroke.color} 1`"
                    />
                  </g>
                  <g transform="rotate(75 50 50)">
                    <circle
                      class="weight-arc-track"
                      cx="50"
                      cy="50"
                      r="51"
                    />
                    <circle
                      class="weight-arc arc-texture"
                      cx="50"
                      cy="50"
                      r="51"
                      pathLength="1"
                      :stroke-dasharray="`${arcStroke.texture} 1`"
                    />
                  </g>
                </svg>
                <img
                  :src="node.imageUrl"
                  :alt="node.raw.label"
                  loading="lazy"
                  decoding="async"
                />
              </button>
            </div>

            <div v-if="pinnedItem" class="pin-tip">
              已固定: {{ pinnedItem.label }}
              <button type="button" class="pin-clear" @click="clearPinnedPreview">取消固定</button>
            </div>
          </div>
        </slot>
      </div>

      <div class="compare-column compare-column-media">
        <div class="panel-block">
          <div class="panel-subtitle-row">
            <div class="panel-subtitle">Image</div>
            <button
              class="upload-icon-btn"
              type="button"
              title="加入 Gallery"
              :disabled="!pinnedItem"
              @click="addPinnedToGallery"
            >
              ⤴
            </button>
          </div>
          <slot name="image">
            <div class="hero-image">
              <img v-if="displayImageUrl" :src="displayImageUrl" alt="reference" />
              <div v-else class="hero-empty">等待左侧保存后的分割图</div>
            </div>
            <div v-if="activePreviewItem" class="image-meta">
              {{ activePreviewItem.label }} · 相似度 {{ activePreviewItem.score.toFixed(4) }}
            </div>
          </slot>
        </div>
        <div class="panel-block gallery-block">
          <div class="panel-subtitle">Gallery</div>
          <slot name="gallery">
            <div class="gallery-grid">
              <div v-if="!galleryResultImages.length" class="gallery-empty">暂无图片，点击 Image 右上角上传</div>
              <img
                v-for="(image, index) in galleryResultImages"
                :key="`${image}-${index}`"
                :src="image"
                :alt="`thumb ${index + 1}`"
              />
            </div>
          </slot>
        </div>
      </div>
    </div>
  </section>
</template>

<style scoped>
.panel {
  background: var(--panel-bg);
  border-radius: 18px;
  border: 1px solid var(--panel-border);
  padding: 18px;
  box-shadow: var(--shadow-soft);
}

.panel-compare {
  display: flex;
  flex-direction: column;
  gap: 16px;
  min-height: 0;
  overflow: hidden;
}

.compare-grid {
  display: grid;
  grid-template-columns: 0.56fr 1.56fr 0.72fr;
  gap: 16px;
  align-items: stretch;
  flex: 1;
  min-height: 0;
}

.compare-column {
  display: flex;
  flex-direction: column;
  gap: 16px;
  min-height: 0;
}

.compare-column-controls {
  width: 100%;
}

.prompt-card {
  display: flex;
  flex-direction: column;
  gap: 10px;
  min-height: 0;
  flex: 1;
}

.prompt-header {
  display: flex;
  align-items: center;
  gap: 8px;
  font-weight: 700;
  color: #5f6b62;
}

.prompt-search-icon {
  font-size: 14px;
  line-height: 1;
}

.prompt-title {
  font-size: 14px;
  letter-spacing: 0.04em;
  text-transform: uppercase;
}

.prompt-input {
  padding: 8px 10px;
  border-radius: 12px;
  border: 1px solid var(--panel-border);
  background: #f6f8f3;
}

.prompt-input textarea {
  width: 100%;
  border: none;
  background: transparent;
  outline: none;
  font-size: 14px;
  line-height: 1.4;
  resize: none;
  min-height: 5.6em;
  font-family: inherit;
}

.weights-title {
  font-weight: 600;
  color: #6a756c;
  margin-bottom: 6px;
}

.weights {
  min-height: 0;
  flex: 1;
  display: flex;
  flex-direction: column;
}

.weights-list {
  min-height: 0;
  flex: 1;
}

.weight-row {
  display: grid;
  grid-template-columns: 72px 1fr 40px;
  align-items: center;
  gap: 12px;
  font-size: 13px;
  margin-top: 8px;
}

.weight-slider {
  width: 100%;
  accent-color: var(--accent-mint);
}

.weight-value {
  color: #6d7b73;
  font-weight: 600;
  text-align: right;
}

.orbit {
  position: relative;
  flex: 1;
  border-radius: 18px;
  border: 1px dashed #d7ded4;
  background: radial-gradient(circle at center, #f4f7f2 0%, #fdfdfb 50%, #f4f7f2 100%);
  min-height: 0;
  overflow: hidden;
}

.orbit-chart {
  width: 100%;
  height: 100%;
  display: block;
}

.orbit-scene {
  position: absolute;
  inset: 0;
  transform-origin: 50% 50%;
  will-change: transform;
}

.orbit-ring {
  fill: none;
  stroke: #dfe6dd;
  stroke-width: 0.35;
}

.orbit-shadow {
  fill: rgba(141, 151, 146, 0.4);
}

.orbit-state {
  position: absolute;
  inset: 0;
  display: flex;
  align-items: center;
  justify-content: center;
  text-align: center;
  padding: 0 24px;
  color: #6c776f;
  font-size: 13px;
}

.orbit-state-error {
  color: #a45448;
}

.rep-node {
  position: absolute;
  width: 44px;
  height: 44px;
  transform: translate(-50%, -50%);
  border-radius: 50%;
  overflow: visible;
  border: none;
  box-shadow: none;
  cursor: pointer;
  padding: 0;
  background: transparent;
}

.orbit-center-image {
  position: absolute;
  left: 50%;
  top: 50%;
  width: 44px;
  height: 44px;
  transform: translate(-50%, -50%);
  border-radius: 50%;
  overflow: visible;
  border: none;
  box-shadow: none;
  background: transparent;
}

.orbit-center-image img {
  width: 100%;
  height: 100%;
  object-fit: cover;
  border-radius: 50%;
}

.rep-node img {
  width: 100%;
  height: 100%;
  object-fit: cover;
  border-radius: 50%;
  transform: translateZ(0);
  backface-visibility: hidden;
}

.circle-arcs {
  position: absolute;
  left: 50%;
  top: 50%;
  transform: translate(-50%, -50%);
  overflow: visible;
  pointer-events: none;
}

.circle-arcs-node {
  width: 48px;
  height: 48px;
}

.weight-arc {
  fill: none;
  stroke-linecap: round;
}

.weight-arc-track {
  fill: none;
  stroke: rgba(132, 141, 137, 0.28);
  stroke-width: 0.9;
}

.arc-structure {
  stroke: #d88a2e;
  stroke-width: 0.9;
}

.arc-color {
  stroke: #6a67dc;
  stroke-width: 0.9;
}

.arc-texture {
  stroke: #31d6be;
  stroke-width: 0.9;
}

.pin-tip {
  position: absolute;
  left: 12px;
  bottom: 12px;
  display: inline-flex;
  align-items: center;
  gap: 8px;
  background: rgba(255, 255, 255, 0.92);
  border: 1px solid var(--panel-border);
  border-radius: 999px;
  padding: 6px 10px;
  font-size: 12px;
  color: #4b5953;
}

.pin-clear {
  border: none;
  background: transparent;
  color: var(--accent-blue);
  font-size: 12px;
  cursor: pointer;
}

.panel-block {
  display: flex;
  flex-direction: column;
  gap: 12px;
  min-height: 0;
}

.compare-column-media {
  padding-bottom: 6px;
}

.gallery-block {
  flex: 1;
}

.panel-subtitle {
  font-weight: 700;
  font-size: 14px;
  text-transform: uppercase;
  letter-spacing: 0.06em;
  color: #6a756c;
}

.panel-subtitle-row {
  display: flex;
  align-items: center;
  justify-content: space-between;
}

.upload-icon-btn {
  width: 28px;
  height: 28px;
  border-radius: 8px;
  border: 1px solid var(--panel-border);
  background: #ffffff;
  color: #4f5c56;
  font-size: 14px;
  cursor: pointer;
}

.upload-icon-btn:disabled {
  opacity: 0.45;
  cursor: not-allowed;
}

.hero-image {
  border-radius: 14px;
  overflow: hidden;
  height: 148px;
  border: 1px solid var(--panel-border);
  background: #f7faf5;
}

.hero-image img {
  width: 100%;
  height: 100%;
  object-fit: cover;
}

.hero-empty {
  width: 100%;
  height: 100%;
  display: flex;
  align-items: center;
  justify-content: center;
  color: #6f7b74;
  font-size: 12px;
  text-align: center;
  padding: 10px;
}

.image-meta {
  font-size: 12px;
  color: #617068;
}

.gallery-grid {
  display: grid;
  grid-template-columns: repeat(2, 1fr);
  gap: 10px;
  min-height: 0;
  overflow-y: auto;
  padding-right: 4px;
  padding-bottom: 8px;
  align-content: start;
}

.gallery-empty {
  grid-column: 1 / -1;
  font-size: 12px;
  color: #6f7b74;
  text-align: center;
  padding: 16px 8px;
  border: 1px dashed var(--panel-border);
  border-radius: 10px;
}

.gallery-grid img {
  border-radius: 10px;
  height: 80px;
  object-fit: cover;
}

@media (max-width: 1200px) {
  .compare-grid {
    grid-template-columns: 1fr;
  }

  .orbit {
    min-height: 260px;
  }
}

@media (max-width: 720px) {
  .gallery-grid {
    grid-template-columns: repeat(3, 1fr);
  }
}
</style>
