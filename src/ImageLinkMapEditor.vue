<template>
  <section class="image-map-editor">
    <header class="toolbar surface-panel">
      <div class="toolbar-copy">
        <p class="eyebrow">Image map editor</p>
        <h1>Редактор ссылок на изображении</h1>
        <p>Загрузите квадратную картинку, выделите прямоугольники мышью, меняйте размер и порядок областей.</p>
      </div>

      <button class="danger" type="button" :disabled="isBusy" @click="clearAll">
        Полная очистка
      </button>
    </header>

    <div class="upload-card surface-panel">
      <label class="upload-label" :class="{ disabled: isBusy }">
        <span v-if="isProcessingUpload" class="inline-loader" aria-hidden="true" />
        <span>{{ isProcessingUpload ? 'Обрабатываем картинку' : 'Добавить новую картинку' }}</span>
        <input type="file" accept="image/*" :disabled="isBusy" @change="onImageUpload" />
      </label>

      <div class="status-row">
        <span v-if="!isHydrated" class="status muted">
          <span class="inline-loader" aria-hidden="true" />
          Загружаем сохранённые данные...
        </span>
        <span v-else-if="saveError" class="status error">{{ saveError }}</span>
        <span v-else class="status success">Сохранено локально</span>
      </div>
    </div>

    <div v-if="isHydrated && images.length === 0" class="empty-state surface-panel">
      Пока нет загруженных картинок.
    </div>

    <article v-for="image in images" :key="image.id" class="image-block surface-panel">
      <div class="image-block-header">
        <div>
          <h2>{{ image.name }}</h2>
          <p>{{ image.rects.length }} прямоугольников</p>
        </div>

        <button type="button" class="secondary" :disabled="isBusy" @click="removeImage(image.id)">
          Удалить картинку
        </button>
      </div>

      <div class="editor-layout">
        <div
            :ref="(el) => setCanvasRef(el, image.id)"
            class="canvas-wrap"
            @pointerdown="startDrawing($event, image.id)"
            @pointermove="handlePointerMove"
            @pointerup="finishInteraction"
            @pointercancel="finishInteraction"
            @lostpointercapture="finishInteraction"
        >
          <img :src="image.src" :alt="image.name" draggable="false" />

          <div
              v-for="rect in image.rects"
              :key="rect.id"
              class="rect"
              :class="{ active: selectedRectId === rect.id }"
              :style="rectStyle(rect, image)"
              @pointerdown.stop="selectRect(rect.id)"
          >
            <span class="rect-index">{{ rectNumber(image, rect.id) }}</span>

            <template v-if="selectedRectId === rect.id">
              <button
                  v-for="handle in resizeHandles"
                  :key="handle"
                  type="button"
                  class="resize-handle"
                  :class="`handle-${handle}`"
                  :aria-label="`Изменить размер: ${handle}`"
                  @pointerdown.stop.prevent="startResizing($event, image.id, rect.id, handle)"
              />
            </template>
          </div>

          <div
              v-if="draftRect && draftRect.imageId === image.id"
              class="rect draft"
              :style="rectStyle(draftRect)"
          />
        </div>

        <aside class="side-panel">
          <div class="panel-title">
            <h3>Точки / прямоугольники</h3>
            <p>Порядок здесь совпадает с номерами на изображении и строками в таблице.</p>
          </div>

          <div v-if="image.rects.length === 0" class="hint note-box">
            Зажмите левую кнопку мыши на картинке и протяните область как в Paint.
          </div>

          <div v-for="rect in image.rects" :key="rect.id" class="rect-form" :class="{ active: selectedRectId === rect.id }">
            <div class="rect-form-header">
              <button type="button" class="rect-title" @click="selectRect(rect.id)">
                Прямоугольник №{{ rectNumber(image, rect.id) }}
              </button>

              <div class="order-actions" aria-label="Изменить порядок прямоугольника">
                <button
                    type="button"
                    class="icon-button"
                    :disabled="isFirstRect(image, rect.id)"
                    title="Поднять выше"
                    @click="moveRect(image.id, rect.id, -1)"
                >
                  ↑
                </button>
                <button
                    type="button"
                    class="icon-button"
                    :disabled="isLastRect(image, rect.id)"
                    title="Опустить ниже"
                    @click="moveRect(image.id, rect.id, 1)"
                >
                  ↓
                </button>
              </div>
            </div>

            <label>
              Ссылка
              <input
                  v-model="rect.link"
                  type="text"
                  placeholder="/catalog/womens/.../"
                  @focus="selectRect(rect.id)"
                  @input="persistState"
              />
            </label>

            <div class="metrics">
              <span>Ширина: {{ formatPercent(rect.width) }}</span>
              <span>Высота: {{ formatPercent(rect.height) }}</span>
              <span>Сверху: {{ formatPercent(rect.top) }}</span>
              <span>Слева: {{ formatPercent(rect.left) }}</span>
            </div>

            <button type="button" class="danger small" @click="removeRect(image.id, rect.id)">
              Удалить
            </button>
          </div>
        </aside>
      </div>

      <section class="result-table">
        <div class="table-header">
          <div>
            <h3>Ссылки</h3>
            <p>Строки идут в том же порядке, что и фигуры.</p>
          </div>
          <button type="button" class="secondary" @click="copyTable(image.id)">
            Скопировать таблицу
          </button>
        </div>

        <div class="table-scroll">
          <table>
            <thead>
            <tr>
              <th>№</th>
              <th>Ссылка</th>
              <th>Ширина</th>
              <th>Высота</th>
              <th>Отступ сверху</th>
              <th>Отступ слева</th>
            </tr>
            </thead>
            <tbody>
            <tr v-for="rect in image.rects" :key="rect.id">
              <td>{{ rectNumber(image, rect.id) }}</td>
              <td>{{ rect.link }}</td>
              <td>{{ formatValue(rect.width) }}</td>
              <td>{{ formatValue(rect.height) }}</td>
              <td>{{ formatValue(rect.top) }}</td>
              <td>{{ formatValue(rect.left) }}</td>
            </tr>
            </tbody>
          </table>
        </div>
      </section>
    </article>
  </section>
</template>

<script setup lang="ts">
import { computed, onBeforeUnmount, onMounted, ref } from 'vue'

type Rect = {
  id: string
  imageId: string
  link: string
  left: number
  top: number
  width: number
  height: number
}

type ImageItem = {
  id: string
  name: string
  src: string
  rects: Rect[]
}

type StoredImageItem = Omit<ImageItem, 'src'>

type Point = {
  x: number
  y: number
}

type ResizeHandle = 'nw' | 'n' | 'ne' | 'e' | 'se' | 's' | 'sw' | 'w'

type Interaction =
    | {
  mode: 'draw'
  imageId: string
  pointerId: number
  start: Point
}
    | {
  mode: 'resize'
  imageId: string
  rectId: string
  pointerId: number
  handle: ResizeHandle
  start: Point
  initial: Rect
}

const STORAGE_KEY = 'image-link-map-editor:v3'
const LEGACY_STORAGE_KEYS = ['image-link-map-editor:v2', 'image-link-map-editor:v1']
const DB_NAME = 'image-link-map-editor'
const DB_VERSION = 1
const IMAGE_STORE_NAME = 'images'
const MIN_RECT_SIZE_PERCENT = 0.5

const resizeHandles: ResizeHandle[] = ['nw', 'n', 'ne', 'e', 'se', 's', 'sw', 'w']

const images = ref<ImageItem[]>([])
const selectedRectId = ref<string | null>(null)
const draftRect = ref<Rect | null>(null)
const interaction = ref<Interaction | null>(null)
const isHydrated = ref(false)
const isProcessingUpload = ref(false)
const saveError = ref('')

const canvasRefs = new Map<string, HTMLElement>()
const objectUrls = new Map<string, string>()
let imageDbPromise: Promise<IDBDatabase> | null = null

const isBusy = computed(() => !isHydrated.value || isProcessingUpload.value)
const hasActiveInteraction = computed(() => Boolean(interaction.value))

onMounted(() => {
  void restoreState()
  window.addEventListener('beforeunload', handleBeforeUnload)
})

onBeforeUnmount(() => {
  window.removeEventListener('beforeunload', handleBeforeUnload)
  revokeAllObjectUrls()
})

async function restoreState() {
  let restoredImages: ImageItem[] = []
  let shouldPersistMigratedState = false

  try {
    const saved = localStorage.getItem(STORAGE_KEY)

    if (saved) {
      restoredImages = await hydrateStoredImages(normalizeStoredImages(JSON.parse(saved)))
    } else {
      const legacySaved = LEGACY_STORAGE_KEYS.map((key) => localStorage.getItem(key)).find(Boolean)

      if (legacySaved) {
        restoredImages = normalizeLegacyImages(JSON.parse(legacySaved))

        await Promise.all(
            restoredImages.map((image) => saveImageValue(image.id, image.src))
        )

        shouldPersistMigratedState = true
      }
    }
  } catch {
    restoredImages = []
    saveError.value = 'Сохранённые данные повреждены или недоступны. Начните заново или очистите редактор.'
  } finally {
    images.value = restoredImages
    isHydrated.value = true

    if (shouldPersistMigratedState) {
      persistState()
      LEGACY_STORAGE_KEYS.forEach((key) => localStorage.removeItem(key))
    }
  }
}

function normalizeStoredImages(value: unknown): StoredImageItem[] {
  if (!Array.isArray(value)) return []

  return value
      .filter((item): item is Partial<StoredImageItem> => Boolean(item && typeof item === 'object'))
      .map((item) => {
        const id = typeof item.id === 'string' ? item.id : makeId()
        const name = typeof item.name === 'string' ? item.name : 'image'
        const rects = Array.isArray(item.rects) ? item.rects : []

        return {
          id,
          name,
          rects: normalizeRects(rects, id),
        }
      })
}

function normalizeLegacyImages(value: unknown): ImageItem[] {
  if (!Array.isArray(value)) return []

  return value
      .filter((item): item is Partial<ImageItem> => Boolean(item && typeof item === 'object'))
      .map((item) => {
        const id = typeof item.id === 'string' ? item.id : makeId()
        const name = typeof item.name === 'string' ? item.name : 'image'
        const src = typeof item.src === 'string' ? item.src : ''
        const rects = Array.isArray(item.rects) ? item.rects : []

        return {
          id,
          name,
          src,
          rects: normalizeRects(rects, id),
        }
      })
      .filter((item) => item.src)
}

function normalizeRects(value: unknown[], imageId: string): Rect[] {
  return value
      .filter((rect): rect is Partial<Rect> => Boolean(rect && typeof rect === 'object'))
      .map((rect) => ({
        id: typeof rect.id === 'string' ? rect.id : makeId(),
        imageId,
        link: typeof rect.link === 'string' ? rect.link : '',
        left: clampNumber(rect.left),
        top: clampNumber(rect.top),
        width: clampNumber(rect.width),
        height: clampNumber(rect.height),
      }))
}

async function hydrateStoredImages(storedImages: StoredImageItem[]) {
  const result: ImageItem[] = []

  for (const image of storedImages) {
    const src = await loadImageSource(image.id)

    if (!src) continue

    result.push({
      ...image,
      src,
    })
  }

  return result
}

function handleBeforeUnload(event: BeforeUnloadEvent) {
  persistState()

  if (!isProcessingUpload.value && !hasActiveInteraction.value) return

  event.preventDefault()
  event.returnValue = ''
}

async function onImageUpload(event: Event) {
  const input = event.target as HTMLInputElement
  const file = input.files?.[0]

  if (!file) return

  const imageId = makeId()
  const src = createTrackedObjectUrl(imageId, file)

  isProcessingUpload.value = true
  saveError.value = ''

  try {
    const isSquare = await validateSquareImage(src)

    if (!isSquare) {
      alert('Картинка не квадратная. Лучше загрузить изображение с одинаковой шириной и высотой.')
    }

    await saveImageValue(imageId, file)

    images.value.push({
      id: imageId,
      name: file.name,
      src,
      rects: [],
    })

    persistState()
    input.value = ''
  } catch {
    revokeObjectUrl(imageId)
    saveError.value = 'Не удалось сохранить изображение. Проверьте доступность IndexedDB и свободное место браузера.'
    alert('Не удалось прочитать или сохранить изображение.')
  } finally {
    isProcessingUpload.value = false
  }
}

function validateSquareImage(src: string) {
  return new Promise<boolean>((resolve, reject) => {
    const img = new Image()

    img.onload = () => {
      const diff = Math.abs(img.naturalWidth - img.naturalHeight)
      resolve(diff <= 2)
    }

    img.onerror = reject
    img.src = src
  })
}

function startDrawing(event: PointerEvent, imageId: string) {
  if (isBusy.value) return

  const point = getPercentPoint(event, imageId)

  if (!point) return

  event.preventDefault()
  capturePointer(event, imageId)

  interaction.value = {
    mode: 'draw',
    imageId,
    pointerId: event.pointerId,
    start: point,
  }

  selectedRectId.value = null
  draftRect.value = {
    id: 'draft',
    imageId,
    link: '',
    left: point.x,
    top: point.y,
    width: 0,
    height: 0,
  }
}

function startResizing(event: PointerEvent, imageId: string, rectId: string, handle: ResizeHandle) {
  if (isBusy.value) return

  const image = findImage(imageId)
  const rect = image?.rects.find((item) => item.id === rectId)
  const point = getPercentPoint(event, imageId)

  if (!image || !rect || !point) return

  event.preventDefault()
  capturePointer(event, imageId)

  selectedRectId.value = rectId
  draftRect.value = null
  interaction.value = {
    mode: 'resize',
    imageId,
    rectId,
    pointerId: event.pointerId,
    handle,
    start: point,
    initial: { ...rect },
  }
}

function handlePointerMove(event: PointerEvent) {
  const currentInteraction = interaction.value

  if (!currentInteraction || event.pointerId !== currentInteraction.pointerId) return

  const currentPoint = getPercentPoint(event, currentInteraction.imageId)

  if (!currentPoint) return

  if (currentInteraction.mode === 'draw') {
    updateDraftRect(currentInteraction, currentPoint)
    return
  }

  updateResizedRect(currentInteraction, currentPoint)
}

function updateDraftRect(currentInteraction: Extract<Interaction, { mode: 'draw' }>, currentPoint: Point) {
  const left = Math.min(currentInteraction.start.x, currentPoint.x)
  const top = Math.min(currentInteraction.start.y, currentPoint.y)
  const width = Math.abs(currentPoint.x - currentInteraction.start.x)
  const height = Math.abs(currentPoint.y - currentInteraction.start.y)

  draftRect.value = {
    id: 'draft',
    imageId: currentInteraction.imageId,
    link: '',
    left: clampNumber(left),
    top: clampNumber(top),
    width: clampNumber(width),
    height: clampNumber(height),
  }
}

function updateResizedRect(currentInteraction: Extract<Interaction, { mode: 'resize' }>, currentPoint: Point) {
  const image = findImage(currentInteraction.imageId)
  const rect = image?.rects.find((item) => item.id === currentInteraction.rectId)

  if (!rect) return

  const nextRect = resizedRect(currentInteraction.initial, currentInteraction.handle, currentInteraction.start, currentPoint)

  rect.left = nextRect.left
  rect.top = nextRect.top
  rect.width = nextRect.width
  rect.height = nextRect.height
}

function finishInteraction(event?: PointerEvent) {
  const currentInteraction = interaction.value

  if (!currentInteraction) return

  if (event && event.pointerId !== currentInteraction.pointerId) return

  if (currentInteraction.mode === 'draw') {
    commitDraftRect(currentInteraction.imageId)
  }

  if (currentInteraction.mode === 'resize') {
    persistState()
  }

  releasePointer(event, currentInteraction.imageId)
  resetInteraction()
}

function commitDraftRect(imageId: string) {
  if (!draftRect.value || draftRect.value.imageId !== imageId) return

  if (draftRect.value.width < MIN_RECT_SIZE_PERCENT || draftRect.value.height < MIN_RECT_SIZE_PERCENT) return

  const image = findImage(imageId)

  if (!image) return

  const rect: Rect = {
    ...draftRect.value,
    id: makeId(),
  }

  image.rects.push(rect)
  selectedRectId.value = rect.id
  persistState()
}

function resetInteraction() {
  interaction.value = null
  draftRect.value = null
}

function getPercentPoint(event: PointerEvent, imageId: string): Point | null {
  const target = canvasRefs.get(imageId)

  if (!target) return null

  const bounds = target.getBoundingClientRect()
  const x = ((event.clientX - bounds.left) / bounds.width) * 100
  const y = ((event.clientY - bounds.top) / bounds.height) * 100

  return {
    x: clampNumber(x),
    y: clampNumber(y),
  }
}

function resizedRect(initial: Rect, handle: ResizeHandle, start: Point, current: Point) {
  const dx = current.x - start.x
  const dy = current.y - start.y

  let left = initial.left
  let top = initial.top
  let right = initial.left + initial.width
  let bottom = initial.top + initial.height

  if (handle.includes('w')) left = initial.left + dx
  if (handle.includes('e')) right = initial.left + initial.width + dx
  if (handle.includes('n')) top = initial.top + dy
  if (handle.includes('s')) bottom = initial.top + initial.height + dy

  left = clampNumber(left)
  top = clampNumber(top)
  right = clampNumber(right)
  bottom = clampNumber(bottom)

  if (right - left < MIN_RECT_SIZE_PERCENT) {
    if (handle.includes('w')) {
      left = Math.max(0, right - MIN_RECT_SIZE_PERCENT)
    } else {
      right = Math.min(100, left + MIN_RECT_SIZE_PERCENT)
    }
  }

  if (bottom - top < MIN_RECT_SIZE_PERCENT) {
    if (handle.includes('n')) {
      top = Math.max(0, bottom - MIN_RECT_SIZE_PERCENT)
    } else {
      bottom = Math.min(100, top + MIN_RECT_SIZE_PERCENT)
    }
  }

  return {
    left: clampNumber(left),
    top: clampNumber(top),
    width: clampNumber(right - left),
    height: clampNumber(bottom - top),
  }
}

function setCanvasRef(el: Element | null, imageId: string) {
  if (el instanceof HTMLElement) {
    canvasRefs.set(imageId, el)
    return
  }

  canvasRefs.delete(imageId)
}

function capturePointer(event: PointerEvent, imageId: string) {
  const canvas = canvasRefs.get(imageId)

  try {
    canvas?.setPointerCapture(event.pointerId)
  } catch {
    // Pointer capture is optional.
  }
}

function releasePointer(event: PointerEvent | undefined, imageId: string) {
  if (!event) return

  const canvas = canvasRefs.get(imageId)

  try {
    if (canvas?.hasPointerCapture(event.pointerId)) {
      canvas.releasePointerCapture(event.pointerId)
    }
  } catch {
    // Ignore release errors from browsers that already released the pointer.
  }
}

function rectStyle(rect: Pick<Rect, 'left' | 'top' | 'width' | 'height'> & Partial<Pick<Rect, 'id'>>, image?: ImageItem) {
  const index = image && rect.id ? image.rects.findIndex((item) => item.id === rect.id) : 0

  return {
    left: `${rect.left}%`,
    top: `${rect.top}%`,
    width: `${rect.width}%`,
    height: `${rect.height}%`,
    zIndex: String(index >= 0 ? index + 2 : 2),
  }
}

function selectRect(rectId: string) {
  selectedRectId.value = rectId
}

function removeRect(imageId: string, rectId: string) {
  const image = findImage(imageId)

  if (!image) return

  image.rects = image.rects.filter((rect) => rect.id !== rectId)

  if (selectedRectId.value === rectId) {
    selectedRectId.value = null
  }

  persistState()
}

async function removeImage(imageId: string) {
  images.value = images.value.filter((image) => image.id !== imageId)
  revokeObjectUrl(imageId)

  try {
    await deleteImageValue(imageId)
  } catch {
    saveError.value = 'Картинка удалена из интерфейса, но запись IndexedDB могла остаться.'
  }

  if (!images.value.some((image) => image.rects.some((rect) => rect.id === selectedRectId.value))) {
    selectedRectId.value = null
  }

  persistState()
}

function moveRect(imageId: string, rectId: string, direction: -1 | 1) {
  const image = findImage(imageId)

  if (!image) return

  const currentIndex = image.rects.findIndex((rect) => rect.id === rectId)
  const nextIndex = currentIndex + direction

  if (currentIndex < 0 || nextIndex < 0 || nextIndex >= image.rects.length) return

  const [rect] = image.rects.splice(currentIndex, 1)
  image.rects.splice(nextIndex, 0, rect)
  selectedRectId.value = rectId
  persistState()
}

async function clearAll() {
  const confirmed = confirm('Удалить все картинки и все прямоугольники?')

  if (!confirmed) return

  images.value = []
  selectedRectId.value = null
  resetInteraction()
  revokeAllObjectUrls()
  saveError.value = ''
  localStorage.removeItem(STORAGE_KEY)
  LEGACY_STORAGE_KEYS.forEach((key) => localStorage.removeItem(key))

  try {
    await clearImageStore()
  } catch {
    saveError.value = 'Данные очищены из localStorage, но IndexedDB могла очиститься не полностью.'
  }
}

function rectNumber(image: ImageItem, rectId: string) {
  return image.rects.findIndex((rect) => rect.id === rectId) + 1
}

function isFirstRect(image: ImageItem, rectId: string) {
  return image.rects.findIndex((rect) => rect.id === rectId) === 0
}

function isLastRect(image: ImageItem, rectId: string) {
  return image.rects.findIndex((rect) => rect.id === rectId) === image.rects.length - 1
}

function formatValue(value: number) {
  return Math.round(value)
}

function formatPercent(value: number) {
  return `${formatValue(value)}%`
}

function clampNumber(value: unknown) {
  const number = Number(value)

  if (!Number.isFinite(number)) return 0

  return Math.min(100, Math.max(0, number))
}

function persistState() {
  if (!isHydrated.value) return

  const payload: StoredImageItem[] = images.value.map((image) => ({
    id: image.id,
    name: image.name,
    rects: image.rects.map((rect) => ({
      id: rect.id,
      imageId: image.id,
      link: rect.link,
      left: rect.left,
      top: rect.top,
      width: rect.width,
      height: rect.height,
    })),
  }))

  try {
    localStorage.setItem(STORAGE_KEY, JSON.stringify(payload))
    saveError.value = ''
  } catch {
    saveError.value = 'Не удалось сохранить координаты. localStorage недоступен или переполнен.'
  }
}

function tableText(image: ImageItem) {
  const header = ['№', 'Ссылка', 'Ширина', 'Высота', 'Отступ сверху', 'Отступ слева'].join('	')
  const rows = image.rects.map((rect) => {
    return [
      rectNumber(image, rect.id),
      rect.link,
      formatValue(rect.width),
      formatValue(rect.height),
      formatValue(rect.top),
      formatValue(rect.left),
    ].join('	')
  })

  return [header, ...rows].join('')
}

async function copyTable(imageId: string) {
  const image = findImage(imageId)

  if (!image) return

  await navigator.clipboard.writeText(tableText(image))
}

function findImage(imageId: string) {
  return images.value.find((item) => item.id === imageId)
}

function makeId() {
  return globalThis.crypto?.randomUUID?.() || `${Date.now()}-${Math.random().toString(16).slice(2)}`
}

function createTrackedObjectUrl(imageId: string, blob: Blob) {
  revokeObjectUrl(imageId)

  const url = URL.createObjectURL(blob)
  objectUrls.set(imageId, url)

  return url
}

function revokeObjectUrl(imageId: string) {
  const url = objectUrls.get(imageId)

  if (!url) return

  URL.revokeObjectURL(url)
  objectUrls.delete(imageId)
}

function revokeAllObjectUrls() {
  objectUrls.forEach((url) => URL.revokeObjectURL(url))
  objectUrls.clear()
}

async function openImageDb() {
  if (imageDbPromise) return imageDbPromise

  imageDbPromise = new Promise<IDBDatabase>((resolve, reject) => {
    const request = indexedDB.open(DB_NAME, DB_VERSION)

    request.onupgradeneeded = () => {
      const db = request.result

      if (!db.objectStoreNames.contains(IMAGE_STORE_NAME)) {
        db.createObjectStore(IMAGE_STORE_NAME)
      }
    }

    request.onsuccess = () => resolve(request.result)
    request.onerror = () => reject(request.error)
  })

  return imageDbPromise
}

async function saveImageValue(imageId: string, value: Blob | string) {
  const db = await openImageDb()
  const transaction = db.transaction(IMAGE_STORE_NAME, 'readwrite')

  transaction.objectStore(IMAGE_STORE_NAME).put(value, imageId)

  await transactionDone(transaction)
}

async function loadImageSource(imageId: string) {
  const db = await openImageDb()
  const transaction = db.transaction(IMAGE_STORE_NAME, 'readonly')
  const request = transaction.objectStore(IMAGE_STORE_NAME).get(imageId)
  const value = await requestResult<Blob | string | undefined>(request)

  await transactionDone(transaction)

  if (value instanceof Blob) return createTrackedObjectUrl(imageId, value)
  if (typeof value === 'string') return value

  return ''
}

async function deleteImageValue(imageId: string) {
  const db = await openImageDb()
  const transaction = db.transaction(IMAGE_STORE_NAME, 'readwrite')

  transaction.objectStore(IMAGE_STORE_NAME).delete(imageId)

  await transactionDone(transaction)
}

async function clearImageStore() {
  const db = await openImageDb()
  const transaction = db.transaction(IMAGE_STORE_NAME, 'readwrite')

  transaction.objectStore(IMAGE_STORE_NAME).clear()

  await transactionDone(transaction)
}

function requestResult<T>(request: IDBRequest) {
  return new Promise<T>((resolve, reject) => {
    request.onsuccess = () => resolve(request.result as T)
    request.onerror = () => reject(request.error)
  })
}

function transactionDone(transaction: IDBTransaction) {
  return new Promise<void>((resolve, reject) => {
    transaction.oncomplete = () => resolve()
    transaction.onerror = () => reject(transaction.error)
    transaction.onabort = () => reject(transaction.error)
  })
}
</script>

<style scoped>
.image-map-editor {
  --bg: #f6f7fb;
  --panel: rgba(255, 255, 255, 0.92);
  --text: #111827;
  --muted: #6b7280;
  --line: #e6e8ef;
  --primary: #2563eb;
  --primary-soft: rgb(37 99 235 / 14%);
  --accent: #f97316;
  --accent-soft: rgb(249 115 22 / 20%);
  --danger: #dc2626;
  --danger-dark: #b91c1c;
  --shadow: 0 18px 60px rgb(15 23 42 / 8%);
  --radius-xl: 24px;
  --radius-lg: 18px;

  display: grid;
  gap: 24px;
  max-width: 1280px;
  margin: 0 auto;
  padding: 32px 16px;
  color: var(--text);
  font-family: Inter, Manrope, ui-sans-serif, system-ui, -apple-system, BlinkMacSystemFont, "Segoe UI", sans-serif;
  background:
      radial-gradient(circle at 12% 4%, rgb(37 99 235 / 10%), transparent 26%),
      radial-gradient(circle at 88% 10%, rgb(249 115 22 / 10%), transparent 28%),
      var(--bg);
}

.surface-panel {
  border: 1px solid var(--line);
  border-radius: var(--radius-xl);
  background: var(--panel);
  box-shadow: var(--shadow);
  backdrop-filter: blur(16px);
}

.toolbar,
.image-block-header,
.table-header,
.rect-form-header,
.status-row {
  display: flex;
  align-items: center;
  justify-content: space-between;
  gap: 16px;
}

.toolbar {
  padding: 24px;
}

.toolbar-copy {
  display: grid;
  gap: 8px;
}

h1,
h2,
h3,
p {
  margin: 0;
}

h1 {
  max-width: 760px;
  font-size: clamp(30px, 4vw, 44px);
  line-height: 1.05;
  letter-spacing: -0.045em;
}

h2 {
  font-size: 21px;
  line-height: 1.2;
  letter-spacing: -0.02em;
}

h3 {
  font-size: 16px;
  line-height: 1.2;
  letter-spacing: -0.01em;
}

.eyebrow {
  color: var(--primary);
  font-size: 12px;
  font-weight: 800;
  letter-spacing: 0.16em;
  text-transform: uppercase;
}

.toolbar p:not(.eyebrow),
.table-header p,
.panel-title p {
  color: var(--muted);
  font-size: 14px;
  line-height: 1.55;
}

button,
.upload-label {
  display: inline-flex;
  align-items: center;
  justify-content: center;
  gap: 8px;
  border: 0;
  border-radius: 999px;
  padding: 11px 16px;
  font: inherit;
  font-weight: 700;
  cursor: pointer;
  background: linear-gradient(135deg, #111827, #1f2937);
  color: #fff;
  box-shadow: 0 10px 22px rgb(17 24 39 / 14%);
  transition:
      transform 160ms ease,
      box-shadow 160ms ease,
      background 160ms ease,
      opacity 160ms ease;
}

button:hover:not(:disabled),
.upload-label:hover:not(.disabled) {
  box-shadow: 0 14px 28px rgb(17 24 39 / 18%);
}

button:disabled,
.upload-label.disabled {
  cursor: not-allowed;
  opacity: 0.55;
}

button.secondary {
  background: #eef2f7;
  color: #111827;
  box-shadow: none;
}

button.danger {
  background: linear-gradient(135deg, var(--danger), var(--danger-dark));
}

button.small {
  justify-self: start;
  padding: 8px 12px;
  font-size: 13px;
}

.upload-card,
.image-block,
.empty-state {
  padding: 18px;
}

.upload-card {
  display: grid;
  gap: 12px;
  justify-items: start;
}

.upload-label input {
  display: none;
}

.hint,
.empty-state,
.image-block-header p {
  color: var(--muted);
  font-size: 14px;
  line-height: 1.5;
}

.status-row {
  justify-content: flex-start;
  min-height: 24px;
}

.status {
  display: inline-flex;
  align-items: center;
  gap: 8px;
  border-radius: 999px;
  padding: 6px 10px;
  font-size: 13px;
  font-weight: 700;
}

.status.success {
  background: #ecfdf5;
  color: #047857;
}

.status.error {
  background: #fef2f2;
  color: var(--danger-dark);
}

.status.muted {
  background: #f3f4f6;
  color: #4b5563;
}

.inline-loader {
  width: 14px;
  height: 14px;
  border: 2px solid currentColor;
  border-right-color: transparent;
  border-radius: 50%;
  animation: spin 700ms linear infinite;
}

.image-block {
  display: grid;
  gap: 20px;
}

.editor-layout {
  display: grid;
  grid-template-columns: minmax(300px, 640px) minmax(280px, 1fr);
  gap: 22px;
  align-items: start;
}

.canvas-wrap {
  position: relative;
  width: 100%;
  aspect-ratio: 1 / 1;
  overflow: hidden;
  border: 1px solid var(--line);
  border-radius: var(--radius-xl);
  background: linear-gradient(135deg, #eef2f7, #f8fafc);
  box-shadow: inset 0 1px 0 rgb(255 255 255 / 70%);
  user-select: none;
  cursor: crosshair;
  touch-action: none;
}

.canvas-wrap img {
  display: block;
  width: 100%;
  height: 100%;
  object-fit: cover;
  pointer-events: none;
}

.rect {
  position: absolute;
  border: 2px solid var(--primary);
  border-radius: 10px;
  background: var(--primary-soft);
  box-sizing: border-box;
  cursor: pointer;
  transition:
      border-color 160ms ease,
      background 160ms ease,
      box-shadow 160ms ease;
}

.rect.active {
  border-color: var(--accent);
  background: var(--accent-soft);
  box-shadow: 0 0 0 4px rgb(249 115 22 / 16%);
}

.rect.draft {
  border-style: dashed;
  pointer-events: none;
}

.rect-index {
  position: absolute;
  top: -2px;
  left: -2px;
  display: grid;
  min-width: 24px;
  height: 24px;
  place-items: center;
  border-radius: 8px 0 8px 0;
  background: var(--primary);
  color: #fff;
  font-size: 12px;
  font-weight: 800;
}

.rect.active .rect-index {
  background: var(--accent);
}

.resize-handle {
  position: absolute;
  z-index: 4;
  width: 14px;
  height: 14px;
  min-width: 14px;
  min-height: 14px;
  padding: 0;
  border: 2px solid #fff;
  border-radius: 999px;
  background: #111827;
  box-shadow: 0 4px 12px rgb(17 24 39 / 24%);
  transform: translate(-50%, -50%);
}

.handle-nw {
  top: 0;
  left: 0;
  cursor: nwse-resize;
}

.handle-n {
  top: 0;
  left: 50%;
  cursor: ns-resize;
}

.handle-ne {
  top: 0;
  left: 100%;
  cursor: nesw-resize;
}

.handle-e {
  top: 50%;
  left: 100%;
  cursor: ew-resize;
}

.handle-se {
  top: 100%;
  left: 100%;
  cursor: nwse-resize;
}

.handle-s {
  top: 100%;
  left: 50%;
  cursor: ns-resize;
}

.handle-sw {
  top: 100%;
  left: 0;
  cursor: nesw-resize;
}

.handle-w {
  top: 50%;
  left: 0;
  cursor: ew-resize;
}

.side-panel {
  display: grid;
  gap: 12px;
}

.panel-title {
  display: grid;
  gap: 4px;
}

.note-box {
  border: 1px dashed var(--line);
  border-radius: var(--radius-lg);
  padding: 14px;
  background: #f9fafb;
}

.rect-form {
  display: grid;
  gap: 12px;
  padding: 14px;
  border: 1px solid var(--line);
  border-radius: var(--radius-lg);
  background: #f9fafb;
  transition:
      border-color 160ms ease,
      box-shadow 160ms ease,
      background 160ms ease;
}

.rect-form.active {
  border-color: rgb(249 115 22 / 54%);
  background: #fff7ed;
  box-shadow: 0 10px 26px rgb(249 115 22 / 12%);
}

.rect-title {
  justify-content: flex-start;
  padding: 0;
  border-radius: 0;
  background: transparent;
  color: var(--text);
  box-shadow: none;
  font-weight: 800;
}

.rect-title:hover:not(:disabled) {
  transform: none;
  box-shadow: none;
}

.order-actions {
  display: inline-flex;
  gap: 6px;
}

.icon-button {
  width: 32px;
  height: 32px;
  padding: 0;
  border-radius: 10px;
  background: #ffffff;
  color: var(--text);
  box-shadow: inset 0 0 0 1px var(--line);
}

.rect-form label {
  display: grid;
  gap: 7px;
  font-size: 13px;
  font-weight: 700;
  color: #374151;
}

.rect-form input {
  width: 100%;
  box-sizing: border-box;
  border: 1px solid #d1d5db;
  border-radius: 12px;
  padding: 11px 12px;
  font: inherit;
  background: #fff;
  outline: none;
  transition:
      border-color 160ms ease,
      box-shadow 160ms ease;
}

.rect-form input:focus {
  border-color: var(--primary);
  box-shadow: 0 0 0 4px rgb(37 99 235 / 12%);
}

.metrics {
  display: grid;
  grid-template-columns: repeat(2, minmax(0, 1fr));
  gap: 8px;
  font-size: 13px;
  color: #4b5563;
}

.metrics span {
  border-radius: 10px;
  padding: 8px 10px;
  background: #fff;
  box-shadow: inset 0 0 0 1px #eef2f7;
}

.result-table {
  display: grid;
  gap: 12px;
}

.table-scroll {
  overflow-x: auto;
  border: 1px solid var(--line);
  border-radius: var(--radius-lg);
}

table {
  width: 100%;
  border-collapse: collapse;
  overflow: hidden;
  font-size: 14px;
}

th,
td {
  border-bottom: 1px solid var(--line);
  padding: 12px;
  text-align: left;
  vertical-align: top;
}

th {
  background: #f8fafc;
  color: #374151;
  font-size: 12px;
  font-weight: 800;
  letter-spacing: 0.06em;
  text-transform: uppercase;
}

tbody tr:last-child td {
  border-bottom: 0;
}

@keyframes spin {
  to {
    transform: rotate(360deg);
  }
}

@media (max-width: 900px) {
  .toolbar,
  .image-block-header,
  .table-header {
    align-items: flex-start;
    flex-direction: column;
  }

  .toolbar {
    padding: 20px;
  }

  .editor-layout {
    grid-template-columns: 1fr;
  }
}
</style>
