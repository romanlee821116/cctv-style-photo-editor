<template>
  <div class="image-editor">
    <!-- 上傳區域 -->
    <div class="upload-section">
      <div v-if="!baseImage" class="upload-area" @click="triggerFileUpload" @dragover.prevent @drop="handleDrop">
        <input
          ref="fileInput"
          type="file"
          accept="image/*"
          @change="handleFileUpload"
          style="display: none"
        />
        <div class="upload-placeholder">
          <div class="upload-icon">📷</div>
          <p>點擊或拖拽上傳底圖</p>
          <p class="upload-hint">支援 JPG, PNG, GIF 格式</p>
        </div>
      </div>
      <div v-else class="image-container">
        <img
          ref="baseImageRef"
          :src="baseImage"
          alt="Base Image"
          @click="handleImageClick"
          class="base-image"
        />
        
        
        <!-- 多個框框 -->
        <div
          v-for="(box, index) in boxes"
          :key="box.id"
          :style="getBoxStyle(box)"
          class="detection-box"
          @mousedown="startDrag($event, index)"
          @click.stop
        >
          <div class="resize-handles">
            <div class="resize-handle nw" @mousedown.stop="startResize($event, index)"></div>
            <div class="resize-handle ne" @mousedown.stop="startResize($event, index)"></div>
            <div class="resize-handle sw" @mousedown.stop="startResize($event, index)"></div>
            <div class="resize-handle se" @mousedown.stop="startResize($event, index)"></div>
          </div>
          
          <!-- 錨點 -->
          <div class="anchor-points">
            <div 
              class="anchor-point top" 
              @mousedown.stop="startAnchorDrag($event, index, 'top')"
            ></div>
            <div 
              class="anchor-point right" 
              @mousedown.stop="startAnchorDrag($event, index, 'right')"
            ></div>
            <div 
              class="anchor-point bottom" 
              @mousedown.stop="startAnchorDrag($event, index, 'bottom')"
            ></div>
            <div 
              class="anchor-point left" 
              @mousedown.stop="startAnchorDrag($event, index, 'left')"
            ></div>
          </div>
          
          <!-- 框框圖片上傳 - 只在 hover 時顯示 -->
          <div class="box-image-upload" v-if="!box.image" @click.stop>
            <input
              :ref="`boxFileInput${index}`"
              type="file"
              accept="image/*"
              @change="handleBoxUpload($event, index)"
              style="display: none"
            />
            <button @click.stop="triggerBoxUpload(index)" class="upload-btn">
              +
            </button>
          </div>
          <img v-if="box.image" :src="box.image" alt="Box Image" class="box-image" @click.stop />
        </div>
        
        <!-- 連線 -->
        <svg class="connection-lines" v-if="connections.length > 0">
          <line
            v-for="(connection, index) in connections"
            :key="index"
            :x1="connection.x1"
            :y1="connection.y1"
            :x2="connection.x2"
            :y2="connection.y2"
            class="connection-line"
          />
        </svg>
        
        <!-- 拖拽中的連線 -->
        <svg class="connection-lines" v-if="isDraggingAnchor">
          <line
            :x1="dragLine.x1"
            :y1="dragLine.y1"
            :x2="dragLine.x2"
            :y2="dragLine.y2"
            class="connection-line dragging"
          />
        </svg>
      </div>
    </div>
    
    <!-- 控制面板 -->
    <div class="control-panel" v-if="baseImage">
      <div class="panel-section">
        <h3>底圖控制</h3>
        <div class="button-group">
          <button @click="triggerFileUpload" class="control-btn primary">
            📷 改變底圖
          </button>
          <button @click="clearBaseImage" class="control-btn danger">
            🗑️ 移除底圖
          </button>
        </div>
      </div>
    </div>
  </div>
</template>

<script setup lang="ts">
import { ref, onMounted, onUnmounted } from 'vue'

interface Box {
  id: string
  x: number
  y: number
  width: number
  height: number
  image?: string
}

interface Connection {
  x1: number
  y1: number
  x2: number
  y2: number
  fromBox: number
  fromSide: string
  toBox: number
  toSide: string
}

const fileInput = ref<HTMLInputElement>()
const baseImageRef = ref<HTMLImageElement>()

const baseImage = ref<string>('')
const boxes = ref<Box[]>([])
const connections = ref<Connection[]>([])

const isResizing = ref(false)
const isDragging = ref(false)
const isDraggingAnchor = ref(false)
const currentBoxIndex = ref(-1)
const resizeStart = ref({ x: 0, y: 0, width: 0, height: 0 })
const dragStart = ref({ x: 0, y: 0, boxX: 0, boxY: 0 })

// 錨點拖拽相關
const anchorDragStart = ref({ 
  boxIndex: -1, 
  side: '', 
  x: 0, 
  y: 0 
})
const dragLine = ref({ x1: 0, y1: 0, x2: 0, y2: 0 })

const getBoxStyle = (box: Box) => ({
  left: `${box.x}px`,
  top: `${box.y}px`,
  width: `${box.width}px`,
  height: `${box.height}px`
})















const triggerFileUpload = () => {
  fileInput.value?.click()
}

const triggerBoxUpload = (index: number) => {
  const input = document.querySelector(`input[ref="boxFileInput${index}"]`) as HTMLInputElement
  input?.click()
}

const handleFileUpload = (event: Event) => {
  const file = (event.target as HTMLInputElement).files?.[0]
  if (file) {
    const reader = new FileReader()
    reader.onload = (e) => {
      baseImage.value = e.target?.result as string
    }
    reader.readAsDataURL(file)
  }
}

const handleDrop = (event: DragEvent) => {
  event.preventDefault()
  const file = event.dataTransfer?.files[0]
  if (file && file.type.startsWith('image/')) {
    const reader = new FileReader()
    reader.onload = (e) => {
      baseImage.value = e.target?.result as string
    }
    reader.readAsDataURL(file)
  }
}

const handleBoxUpload = (event: Event, index: number) => {
  const file = (event.target as HTMLInputElement).files?.[0]
  if (file && boxes.value[index]) {
    const reader = new FileReader()
    reader.onload = (e) => {
      if (boxes.value[index]) {
        boxes.value[index].image = e.target?.result as string
      }
    }
    reader.readAsDataURL(file)
  }
}

const handleImageClick = (event: MouseEvent) => {
  if (!baseImageRef.value) return
  
  const rect = baseImageRef.value.getBoundingClientRect()
  const x = event.clientX - rect.left
  const y = event.clientY - rect.top
  
  // 檢查是否點擊在現有框框內
  for (const box of boxes.value) {
    if (isPointInBox(x, y, box)) {
      return
    }
  }
  
  // 新增框框
  addBoxAt(x, y)
}

const isPointInBox = (x: number, y: number, box: Box) => {
  return x >= box.x && x <= box.x + box.width && y >= box.y && y <= box.y + box.height
}

const addBoxAt = (x: number, y: number) => {
  const newBox: Box = {
    id: `box-${Date.now()}`,
    x: x - 50,
    y: y - 50,
    width: 100,
    height: 100
  }
  boxes.value.push(newBox)
}

const clearAllBoxes = () => {
  boxes.value = []
  connections.value = []
}

// 錨點拖拽功能
const startAnchorDrag = (event: MouseEvent, boxIndex: number, side: string) => {
  event.preventDefault()
  isDraggingAnchor.value = true
  anchorDragStart.value = {
    boxIndex,
    side,
    x: event.clientX,
    y: event.clientY
  }
  
  const box = boxes.value[boxIndex]
  if (box && baseImageRef.value) {
    const rect = baseImageRef.value.getBoundingClientRect()
    const anchorPos = getAnchorPosition(box, side)
    
    // 將錨點位置轉換為相對於圖片容器的座標
    dragLine.value = {
      x1: anchorPos.x,
      y1: anchorPos.y,
      x2: event.clientX - rect.left,
      y2: event.clientY - rect.top
    }
  }
}

const getAnchorPosition = (box: Box, side: string) => {
  switch (side) {
    case 'top':
      return { x: box.x + box.width / 2, y: box.y }
    case 'right':
      return { x: box.x + box.width, y: box.y + box.height / 2 }
    case 'bottom':
      return { x: box.x + box.width / 2, y: box.y + box.height }
    case 'left':
      return { x: box.x, y: box.y + box.height / 2 }
    default:
      return { x: box.x, y: box.y }
  }
}

const getClosestAnchor = (x: number, y: number) => {
  let closestBox = -1
  let closestSide = ''
  let minDistance = Infinity
  
  for (let i = 0; i < boxes.value.length; i++) {
    const box = boxes.value[i]
    if (!box || i === anchorDragStart.value.boxIndex) continue
    
    const sides = ['top', 'right', 'bottom', 'left']
    for (const side of sides) {
      const anchorPos = getAnchorPosition(box, side)
      const distance = Math.sqrt(
        Math.pow(x - anchorPos.x, 2) + Math.pow(y - anchorPos.y, 2)
      )
      
      if (distance < minDistance && distance < 20) { // 20px 吸附範圍
        minDistance = distance
        closestBox = i
        closestSide = side
      }
    }
  }
  
  return { boxIndex: closestBox, side: closestSide }
}

// 更新連線位置
const updateConnections = (boxIndex: number) => {
  const box = boxes.value[boxIndex]
  if (!box) return
  
  // 更新所有與該框框相關的連線
  connections.value.forEach(connection => {
    if (connection.fromBox === boxIndex) {
      // 更新起始點
      const fromPos = getAnchorPosition(box, connection.fromSide)
      connection.x1 = fromPos.x
      connection.y1 = fromPos.y
    }
    
    if (connection.toBox === boxIndex) {
      // 更新終點
      const toPos = getAnchorPosition(box, connection.toSide)
      connection.x2 = toPos.x
      connection.y2 = toPos.y
    }
  })
}

const clearBaseImage = () => {
  baseImage.value = ''
  clearAllBoxes()
}

const startDrag = (event: MouseEvent, index: number) => {
  event.preventDefault()
  isDragging.value = true
  currentBoxIndex.value = index
  
  const box = boxes.value[index]
  if (box) {
    dragStart.value = {
      x: event.clientX,
      y: event.clientY,
      boxX: box.x,
      boxY: box.y
    }
  }
}

const startResize = (event: MouseEvent, index: number) => {
  event.preventDefault()
  isResizing.value = true
  currentBoxIndex.value = index
  
  const box = boxes.value[index]
  if (box) {
    resizeStart.value = {
      x: event.clientX,
      y: event.clientY,
      width: box.width,
      height: box.height
    }
  }
}

const handleMouseMove = (event: MouseEvent) => {
  if (isResizing.value && currentBoxIndex.value >= 0) {
    const deltaX = event.clientX - resizeStart.value.x
    const deltaY = event.clientY - resizeStart.value.y
    
    const box = boxes.value[currentBoxIndex.value]
    if (box) {
      box.width = Math.max(50, resizeStart.value.width + deltaX)
      box.height = Math.max(50, resizeStart.value.height + deltaY)
      
      // 更新與該框框相關的連線
      updateConnections(currentBoxIndex.value)
    }
  }
  
  if (isDragging.value && currentBoxIndex.value >= 0) {
    const deltaX = event.clientX - dragStart.value.x
    const deltaY = event.clientY - dragStart.value.y
    
    const box = boxes.value[currentBoxIndex.value]
    if (box) {
      box.x = Math.max(0, dragStart.value.boxX + deltaX)
      box.y = Math.max(0, dragStart.value.boxY + deltaY)
      
      // 更新與該框框相關的連線
      updateConnections(currentBoxIndex.value)
    }
  }
  
  if (isDraggingAnchor.value && baseImageRef.value) {
    // 更新拖拽中的連線
    const rect = baseImageRef.value.getBoundingClientRect()
    dragLine.value.x2 = event.clientX - rect.left
    dragLine.value.y2 = event.clientY - rect.top
  }
}

const handleMouseUp = (event: MouseEvent) => {
  if (isDraggingAnchor.value && baseImageRef.value) {
    // 檢查是否拖拽到另一個錨點
    const rect = baseImageRef.value.getBoundingClientRect()
    const relativeX = event.clientX - rect.left
    const relativeY = event.clientY - rect.top
    const closest = getClosestAnchor(relativeX, relativeY)
    
    if (closest.boxIndex >= 0) {
      // 創建連線
      const fromBox = boxes.value[anchorDragStart.value.boxIndex]
      const toBox = boxes.value[closest.boxIndex]
      
      if (fromBox && toBox) {
        const fromPos = getAnchorPosition(fromBox, anchorDragStart.value.side)
        const toPos = getAnchorPosition(toBox, closest.side)
        
        const connection: Connection = {
          x1: fromPos.x,
          y1: fromPos.y,
          x2: toPos.x,
          y2: toPos.y,
          fromBox: anchorDragStart.value.boxIndex,
          fromSide: anchorDragStart.value.side,
          toBox: closest.boxIndex,
          toSide: closest.side
        }
        
        connections.value.push(connection)
      }
    }
    
    isDraggingAnchor.value = false
    anchorDragStart.value = { boxIndex: -1, side: '', x: 0, y: 0 }
  }
  
  isResizing.value = false
  isDragging.value = false
  currentBoxIndex.value = -1
}

onMounted(() => {
  document.addEventListener('mousemove', handleMouseMove)
  document.addEventListener('mouseup', handleMouseUp)
})

onUnmounted(() => {
  document.removeEventListener('mousemove', handleMouseMove)
  document.removeEventListener('mouseup', handleMouseUp)
})
</script>

<style scoped>
.image-editor {
  max-width: 1200px;
  margin: 0 auto;
}

.upload-section {
  margin-bottom: 2rem;
}

.upload-area {
  border: 2px dashed #00ff00;
  border-radius: 8px;
  padding: 2rem;
  text-align: center;
  background: rgba(0, 255, 0, 0.05);
  cursor: pointer;
  transition: all 0.3s ease;
  min-height: 400px;
  display: flex;
  align-items: center;
  justify-content: center;
}

.upload-area:hover {
  border-color: #00ff88;
  background: rgba(0, 255, 0, 0.1);
}

.upload-placeholder {
  color: #00ff00;
}

.upload-icon {
  font-size: 4rem;
  margin-bottom: 1rem;
  opacity: 0.7;
}

.upload-hint {
  font-size: 0.9rem;
  opacity: 0.6;
  margin-top: 0.5rem;
}

.image-container {
  position: relative;
  display: inline-block;
  max-width: 100%;
}

.base-image {
  max-width: 100%;
  max-height: 70vh;
  border-radius: 4px;
  cursor: crosshair;
}

.detection-box {
  position: absolute;
  border: 2px solid #00ff00;
  border-radius: 4px;
  cursor: move;
  user-select: none;
  min-width: 50px;
  min-height: 50px;
  transition: all 0.2s ease;
  z-index: 2;
  background: transparent;
}

.detection-box:hover {
  background: rgba(0, 0, 0, 0.5);
  box-shadow: 0 0 10px #00ff00;
}

.box-label {
  position: absolute;
  top: -25px;
  left: 0;
  background: currentColor;
  color: #000;
  padding: 2px 8px;
  font-weight: bold;
  font-size: 12px;
  border-radius: 2px;
}

.box-a .box-label {
  background: #00ff00;
}

.box-b .box-label {
  background: #0088ff;
}

.resize-handles {
  position: absolute;
  top: 0;
  left: 0;
  right: 0;
  bottom: 0;
  pointer-events: none;
}

.resize-handle {
  position: absolute;
  width: 8px;
  height: 8px;
  background: currentColor;
  border: 1px solid #000;
  pointer-events: all;
  cursor: nw-resize;
}

.resize-handle.nw {
  top: -4px;
  left: -4px;
  cursor: nw-resize;
}

.resize-handle.ne {
  top: -4px;
  right: -4px;
  cursor: ne-resize;
}

.resize-handle.sw {
  bottom: -4px;
  left: -4px;
  cursor: sw-resize;
}

.resize-handle.se {
  bottom: -4px;
  right: -4px;
  cursor: se-resize;
}

.box-image-upload {
  position: absolute;
  top: 50%;
  left: 50%;
  transform: translate(-50%, -50%);
  text-align: center;
  opacity: 0;
  transition: opacity 0.2s ease;
}

.detection-box:hover .box-image-upload {
  opacity: 1;
}

/* 錨點樣式 */
.anchor-points {
  position: absolute;
  top: 0;
  left: 0;
  width: 100%;
  height: 100%;
  pointer-events: none;
}

.anchor-point {
  position: absolute;
  width: 12px;
  height: 12px;
  background: #00ff00;
  border: 2px solid #000;
  border-radius: 50%;
  cursor: pointer;
  pointer-events: all;
  transition: all 0.2s ease;
  z-index: 10;
}

.anchor-point:hover {
  background: #fff;
  transform: scale(1.2);
  box-shadow: 0 0 8px #00ff00;
}

.anchor-point.top {
  top: -6px;
  left: 50%;
  transform: translateX(-50%);
}

.anchor-point.right {
  right: -6px;
  top: 50%;
  transform: translateY(-50%);
}

.anchor-point.bottom {
  bottom: -6px;
  left: 50%;
  transform: translateX(-50%);
}

.anchor-point.left {
  left: -6px;
  top: 50%;
  transform: translateY(-50%);
}

/* 連線樣式 */
.connection-lines {
  position: absolute;
  top: 0;
  left: 0;
  width: 100%;
  height: 100%;
  pointer-events: none;
  z-index: 1;
}

.connection-line {
  stroke: #00ff00;
  stroke-width: 2;
  fill: none;
  stroke-dasharray: none;
}

.connection-line.dragging {
  stroke: #00ff00;
  stroke-width: 2;
  stroke-dasharray: 5,5;
  animation: dash 1s linear infinite;
}

@keyframes dash {
  to {
    stroke-dashoffset: -10;
  }
}

.upload-btn {
  background: rgba(0, 0, 0, 0.8);
  color: #00ff00;
  border: 1px solid #00ff00;
  padding: 8px;
  border-radius: 50%;
  cursor: pointer;
  font-family: inherit;
  font-size: 20px;
  font-weight: bold;
  width: 40px;
  height: 40px;
  display: flex;
  align-items: center;
  justify-content: center;
  transition: all 0.3s ease;
}

.upload-btn:hover {
  background: #00ff00;
  color: #000;
  transform: scale(1.1);
  box-shadow: 0 0 10px #00ff00;
}

.box-image {
  width: 100%;
  height: 100%;
  object-fit: cover;
  border-radius: 2px;
}

.control-panel {
  background: rgba(0, 0, 0, 0.8);
  border: 1px solid #00ff00;
  border-radius: 8px;
  padding: 1.5rem;
  margin-top: 2rem;
}

.panel-section {
  margin-bottom: 1.5rem;
}

.panel-section:last-child {
  margin-bottom: 0;
}

.panel-section h3 {
  color: #00ff00;
  margin-bottom: 1rem;
  font-size: 1.2rem;
  text-shadow: 0 0 5px #00ff00;
}

.button-group {
  display: flex;
  gap: 1rem;
  flex-wrap: wrap;
}

.control-btn {
  background: transparent;
  color: #00ff00;
  border: 1px solid #00ff00;
  padding: 10px 20px;
  border-radius: 4px;
  cursor: pointer;
  font-family: inherit;
  font-size: 14px;
  transition: all 0.3s ease;
}

.control-btn:hover:not(:disabled) {
  background: #00ff00;
  color: #000;
}

.control-btn:disabled {
  opacity: 0.5;
  cursor: not-allowed;
}

.control-btn.danger {
  border-color: #ff4444;
  color: #ff4444;
}

.control-btn.danger:hover:not(:disabled) {
  background: #ff4444;
  color: #000;
}

.control-btn.primary {
  border-color: #0088ff;
  color: #0088ff;
}

.control-btn.primary:hover:not(:disabled) {
  background: #0088ff;
  color: #000;
}

.box-info {
  display: grid;
  grid-template-columns: repeat(auto-fit, minmax(200px, 1fr));
  gap: 0.5rem;
}

.info-item {
  display: flex;
  justify-content: space-between;
  padding: 0.5rem;
  background: rgba(0, 255, 0, 0.05);
  border-radius: 4px;
}

.label {
  font-weight: bold;
  color: #00ff88;
}


@media (max-width: 768px) {
  .button-group {
    flex-direction: column;
  }
  
  .control-btn {
    width: 100%;
  }
  
  .box-info {
    grid-template-columns: 1fr;
  }
}
</style>
