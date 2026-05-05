<template>
  <section class="image-map-editor">
    <header class="toolbar">
      <div>
        <h1>Редактор ссылок на изображении</h1>
        <p>Загрузите квадратную картинку, выделите прямоугольники мышью и задайте ссылки.</p>
      </div>

      <button class="danger" type="button" @click="clearAll">
        Полная очистка
      </button>
    </header>

    <div class="upload-card">
      <label class="upload-label">
        <span>Добавить новую картинку</span>
        <input type="file" accept="image/*" @change="onImageUpload" />
      </label>
      <p class="hint">
        Данные хранятся в браузере пользователя через localStorage, поэтому прогресс не теряется после обновления страницы.
      </p>
    </div>

    <div v-if="images.length === 0" class="empty-state">
      Пока нет загруженных картинок.
    </div>

    <article v-for="image in images" :key="image.id" class="image-block">
      <div class="image-block-header">
        <div>
          <h2>{{ image.name }}</h2>
          <p>{{ image.rects.length }} прямоугольников</p>
        </div>

        <button type="button" class="secondary" @click="removeImage(image.id)">
          Удалить картинку
        </button>
      </div>

      <div class="editor-layout">
        <div
            class="canvas-wrap"
            @mousedown="startDrawing($event, image.id)"
            @mousemove="draw($event, image.id)"
            @mouseup="finishDrawing(image.id)"
            @mouseleave="finishDrawing(image.id)"
        >
          <img :src="image.src" :alt="image.name" draggable="false" />

          <div
              v-for="rect in image.rects"
              :key="rect.id"
              class="rect"
              :class="{ active: selectedRectId === rect.id }"
              :style="rectStyle(rect)"
              @mousedown.stop="selectRect(rect.id)"
          >
            <span>{{ rectNumber(image, rect.id) }}</span>
          </div>

          <div
              v-if="draftRect && draftRect.imageId === image.id"
              class="rect draft"
              :style="rectStyle(draftRect)"
          />
        </div>

        <aside class="side-panel">
          <h3>Точки / прямоугольники</h3>

          <div v-if="image.rects.length === 0" class="hint">
            Зажмите левую кнопку мыши на картинке и протяните область как в Paint.
          </div>

          <div v-for="rect in image.rects" :key="rect.id" class="rect-form">
            <label>
              Ссылка
              <input
                  v-model="rect.link"
                  type="text"
                  placeholder="/catalog/womens/.../"
                  @focus="selectRect(rect.id)"
                  @input="saveState"
              />
            </label>

            <div class="metrics">
              <span>Ширина: {{ round(rect.width) }}%</span>
              <span>Высота: {{ round(rect.height) }}%</span>
              <span>Сверху: {{ round(rect.top) }}%</span>
              <span>Слева: {{ round(rect.left) }}%</span>
            </div>

            <button type="button" class="danger small" @click="removeRect(image.id, rect.id)">
              Удалить
            </button>
          </div>
        </aside>
      </div>

      <section class="result-table">
        <div class="table-header">
          <h3>Ссылки</h3>
          <button type="button" class="secondary" @click="copyTable(image.id)">
            Скопировать таблицу
          </button>
        </div>

        <div class="table-scroll">
          <table>
            <thead>
            <tr>
              <th>Ссылка</th>
              <th>Ширина</th>
              <th>Высота</th>
              <th>Отступ сверху</th>
              <th>Отступ слева</th>
            </tr>
            </thead>
            <tbody>
            <tr v-for="rect in image.rects" :key="rect.id">
              <td>{{ rect.link }}</td>
              <td>{{ round(rect.width) }}</td>
              <td>{{ round(rect.height) }}</td>
              <td>{{ round(rect.top) }}</td>
              <td>{{ round(rect.left) }}</td>
            </tr>
            </tbody>
          </table>
        </div>
      </section>
    </article>
  </section>
</template>

<script setup lang="ts">
import { computed, onMounted, ref, watch } from 'vue'

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

type Point = {
  x: number
  y: number
}

const STORAGE_KEY = 'image-link-map-editor:v1'
const MIN_RECT_SIZE_PERCENT = 0.5

const images = ref<ImageItem[]>([])
const selectedRectId = ref<string | null>(null)
const drawingImageId = ref<string | null>(null)
const startPoint = ref<Point | null>(null)
const draftRect = ref<Rect | null>(null)

const hasDraft = computed(() => Boolean(draftRect.value && drawingImageId.value))

onMounted(() => {
  const saved = localStorage.getItem(STORAGE_KEY)

  if (!saved) return

  try {
    const parsed = JSON.parse(saved) as ImageItem[]
    images.value = Array.isArray(parsed) ? parsed : []
  } catch {
    images.value = []
  }
})

watch(
    images,
    () => {
      saveState()
    },
    { deep: true }
)

function onImageUpload(event: Event) {
  const input = event.target as HTMLInputElement
  const file = input.files?.[0]

  if (!file) return

  const reader = new FileReader()

  reader.onload = () => {
    const src = String(reader.result)

    validateSquareImage(src)
        .then((isSquare) => {
          if (!isSquare) {
            alert('Картинка не квадратная. Лучше загрузить изображение с одинаковой шириной и высотой.')
          }

          images.value.push({
            id: crypto.randomUUID(),
            name: file.name,
            src,
            rects: [],
          })

          input.value = ''
        })
        .catch(() => {
          alert('Не удалось прочитать изображение.')
        })
  }

  reader.readAsDataURL(file)
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

function startDrawing(event: MouseEvent, imageId: string) {
  const point = getPercentPoint(event)

  if (!point) return

  drawingImageId.value = imageId
  startPoint.value = point
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

function draw(event: MouseEvent, imageId: string) {
  if (!startPoint.value || drawingImageId.value !== imageId) return

  const currentPoint = getPercentPoint(event)

  if (!currentPoint) return

  const left = Math.min(startPoint.value.x, currentPoint.x)
  const top = Math.min(startPoint.value.y, currentPoint.y)
  const width = Math.abs(currentPoint.x - startPoint.value.x)
  const height = Math.abs(currentPoint.y - startPoint.value.y)

  draftRect.value = {
    id: 'draft',
    imageId,
    link: '',
    left: clamp(left),
    top: clamp(top),
    width: clamp(width),
    height: clamp(height),
  }
}

function finishDrawing(imageId: string) {
  if (!hasDraft.value || !draftRect.value || drawingImageId.value !== imageId) {
    resetDrawing()
    return
  }

  if (
      draftRect.value.width < MIN_RECT_SIZE_PERCENT ||
      draftRect.value.height < MIN_RECT_SIZE_PERCENT
  ) {
    resetDrawing()
    return
  }

  const image = images.value.find((item) => item.id === imageId)

  if (!image) {
    resetDrawing()
    return
  }

  const rect: Rect = {
    ...draftRect.value,
    id: crypto.randomUUID(),
  }

  image.rects.push(rect)
  selectedRectId.value = rect.id
  resetDrawing()
  saveState()
}

function resetDrawing() {
  drawingImageId.value = null
  startPoint.value = null
  draftRect.value = null
}

function getPercentPoint(event: MouseEvent): Point | null {
  const target = event.currentTarget as HTMLElement | null

  if (!target) return null

  const bounds = target.getBoundingClientRect()
  const x = ((event.clientX - bounds.left) / bounds.width) * 100
  const y = ((event.clientY - bounds.top) / bounds.height) * 100

  return {
    x: clamp(x),
    y: clamp(y),
  }
}

function rectStyle(rect: Pick<Rect, 'left' | 'top' | 'width' | 'height'>) {
  return {
    left: `${rect.left}%`,
    top: `${rect.top}%`,
    width: `${rect.width}%`,
    height: `${rect.height}%`,
  }
}

function selectRect(rectId: string) {
  selectedRectId.value = rectId
}

function removeRect(imageId: string, rectId: string) {
  const image = images.value.find((item) => item.id === imageId)

  if (!image) return

  image.rects = image.rects.filter((rect) => rect.id !== rectId)

  if (selectedRectId.value === rectId) {
    selectedRectId.value = null
  }

  saveState()
}

function removeImage(imageId: string) {
  images.value = images.value.filter((image) => image.id !== imageId)
  saveState()
}

function clearAll() {
  const confirmed = confirm('Удалить все картинки и все прямоугольники?')

  if (!confirmed) return

  images.value = []
  selectedRectId.value = null
  resetDrawing()
  localStorage.removeItem(STORAGE_KEY)
}

function rectNumber(image: ImageItem, rectId: string) {
  return image.rects.findIndex((rect) => rect.id === rectId) + 1
}

function round(value: number) {
  return Math.round(value)
}

function clamp(value: number) {
  return Math.min(100, Math.max(0, value))
}

function saveState() {
  localStorage.setItem(STORAGE_KEY, JSON.stringify(images.value))
}

function tableText(image: ImageItem) {
  const header = ['Ссылка', 'Ширина', 'Высота', 'Отступ сверху', 'Отступ слева'].join('\t')
  const rows = image.rects.map((rect) => {
    return [
      rect.link,
      round(rect.width),
      round(rect.height),
      round(rect.top),
      round(rect.left),
    ].join('\t')
  })

  return [header, ...rows].join('\n')
}

async function copyTable(imageId: string) {
  const image = images.value.find((item) => item.id === imageId)

  if (!image) return

  await navigator.clipboard.writeText(tableText(image))
}
</script>

<style scoped>
.image-map-editor {
  display: grid;
  gap: 24px;
  max-width: 1280px;
  margin: 0 auto;
  padding: 32px 16px;
  color: #111827;
}

.toolbar,
.image-block-header,
.table-header {
  display: flex;
  align-items: center;
  justify-content: space-between;
  gap: 16px;
}

h1,
h2,
h3,
p {
  margin: 0;
}

h1 {
  font-size: 28px;
  line-height: 1.2;
}

h2 {
  font-size: 20px;
}

h3 {
  font-size: 16px;
}

button,
.upload-label {
  border: 0;
  border-radius: 10px;
  padding: 10px 14px;
  font: inherit;
  cursor: pointer;
  background: #111827;
  color: #fff;
}

button.secondary {
  background: #e5e7eb;
  color: #111827;
}

button.danger {
  background: #b91c1c;
}

button.small {
  padding: 8px 10px;
  font-size: 13px;
}

.upload-card,
.image-block,
.empty-state {
  border: 1px solid #e5e7eb;
  border-radius: 16px;
  padding: 18px;
  background: #fff;
  box-shadow: 0 8px 24px rgb(15 23 42 / 6%);
}

.upload-card {
  display: grid;
  gap: 10px;
  justify-items: start;
}

.upload-label input {
  display: none;
}

.hint,
.empty-state,
.image-block-header p {
  color: #6b7280;
  font-size: 14px;
}

.image-block {
  display: grid;
  gap: 20px;
}

.editor-layout {
  display: grid;
  grid-template-columns: minmax(300px, 640px) minmax(280px, 1fr);
  gap: 20px;
  align-items: start;
}

.canvas-wrap {
  position: relative;
  width: 100%;
  aspect-ratio: 1 / 1;
  overflow: hidden;
  border-radius: 16px;
  background: #f3f4f6;
  user-select: none;
  cursor: crosshair;
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
  border: 2px solid #2563eb;
  background: rgb(37 99 235 / 18%);
  box-sizing: border-box;
  cursor: pointer;
}

.rect.active {
  border-color: #f97316;
  background: rgb(249 115 22 / 22%);
}

.rect.draft {
  border-style: dashed;
  pointer-events: none;
}

.rect span {
  position: absolute;
  top: -2px;
  left: -2px;
  display: grid;
  min-width: 22px;
  height: 22px;
  place-items: center;
  background: #2563eb;
  color: #fff;
  font-size: 12px;
  font-weight: 700;
}

.rect.active span {
  background: #f97316;
}

.side-panel {
  display: grid;
  gap: 12px;
}

.rect-form {
  display: grid;
  gap: 10px;
  padding: 12px;
  border: 1px solid #e5e7eb;
  border-radius: 12px;
  background: #f9fafb;
}

.rect-form label {
  display: grid;
  gap: 6px;
  font-size: 13px;
  color: #374151;
}

.rect-form input {
  width: 100%;
  box-sizing: border-box;
  border: 1px solid #d1d5db;
  border-radius: 8px;
  padding: 9px 10px;
  font: inherit;
  background: #fff;
}

.metrics {
  display: grid;
  grid-template-columns: repeat(2, minmax(0, 1fr));
  gap: 6px;
  font-size: 13px;
  color: #4b5563;
}

.result-table {
  display: grid;
  gap: 12px;
}

.table-scroll {
  overflow-x: auto;
}

table {
  width: 100%;
  border-collapse: collapse;
  font-size: 14px;
}

th,
td {
  border: 1px solid #e5e7eb;
  padding: 10px;
  text-align: left;
  vertical-align: top;
}

th {
  background: #f3f4f6;
  font-weight: 700;
}

@media (max-width: 900px) {
  .toolbar,
  .image-block-header,
  .table-header {
    align-items: flex-start;
    flex-direction: column;
  }

  .editor-layout {
    grid-template-columns: 1fr;
  }
}
</style>
