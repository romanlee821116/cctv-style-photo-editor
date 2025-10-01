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
          <div class="upload-icon">
            <img src="/add.png" alt="Upload" />
          </div>
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
        <div v-if="baseImage" class="remove-base-image-btn" @click="clearBaseImage">
          <img src="/cross.png" alt="Close" />
        </div>

        <!-- 多個框框 -->
        <div
          v-for="(box, index) in boxes"
          :key="box.id"
          :style="getBoxStyle(box)"
          :class="['detection-box', { 'selected': selectedBoxIndex === index }]"
          @mousedown="startDrag($event, index)"
          @click.stop="selectBox(index)"
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
          
          <!-- 刪除按鈕 -->
          <div class="box-delete-btn" @click.stop="deleteBox(index)">
            <img src="/cross.png" alt="Delete" />
          </div>
          
          <!-- 框框圖片上傳 - 只在 hover 時顯示 -->
          <div class="box-image-upload" v-if="!box.image" @click.stop>
            <input
              :data-box-index="index"
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
          
          <!-- 框框文字 -->
          <div
            v-if="box.text"
            class="box-text"
            :style="{ backgroundColor: box.color }"
          >
            {{ box.text }}
          </div>
        </div>
        
        <!-- 掃描網格 -->
        <div class="scan-grid" v-if="baseImage"></div>
        
        <!-- 掃描線 -->
        <div class="scan-line" v-if="baseImage"></div>
        
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
        <!-- 邊緣標尺 -->
        <div class="edge-rulers">
          <div class="ruler-left">
            <div class="ruler-mark" v-for="i in 10" :key="i" :style="{ top: `${i * 10}%` }"></div>
          </div>
          <div class="ruler-right">
            <div class="ruler-mark" v-for="i in 10" :key="i" :style="{ top: `${i * 10}%` }"></div>
          </div>
        </div>
        
        <!-- 角落標記 -->
        <div class="corner-markers">
          <div class="corner-marker top-left"></div>
          <div class="corner-marker top-right"></div>
          <div class="corner-marker bottom-left"></div>
          <div class="corner-marker bottom-right"></div>
        </div>
      </div>
    </div>
    
    <!-- 框框控制面板 -->
    <div v-if="showControlPanel && selectedBoxIndex >= 0" class="box-control-panel">
      <div class="panel-header">
        <h3>BOX SETTING</h3>
      </div>
      
      <div class="panel-content">
        <div class="control-group">
          <label>COLOR</label>
          <input 
            type="color" 
            :value="boxes[selectedBoxIndex]?.color || '#c0c0c0'"
            @input="handleColorChange"
          />
        </div>
        
        <div class="control-group">
          <label>TEXT</label>
          <input 
            type="text" 
            :value="boxes[selectedBoxIndex]?.text || ''"
            @input="handleTextChange"
            placeholder="ENTER BOX TEXT..."
          />
        </div>
        
        <div class="control-group">
          <label>BORDER WIDTH: {{ boxes[selectedBoxIndex]?.borderSize || 2 }}px</label>
          <input 
            type="range" 
            min="1" 
            max="10" 
            :value="boxes[selectedBoxIndex]?.borderSize || 2"
            @input="handleBorderSizeChange"
          />
        </div>
      </div>
    </div>
    <button @click="saveImage" class="save-btn">
      💾 SAVE
    </button>
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
  color: string
  text: string
  borderSize: number
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
const selectedBoxIndex = ref(-1)
const showControlPanel = ref(false)
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
  height: `${box.height}px`,
  borderColor: box.color,
  borderWidth: `${box.borderSize}px`,
  color: box.color
})

const triggerFileUpload = () => {
  fileInput.value?.click()
}

const triggerBoxUpload = (index: number) => {
  const input = document.querySelector(`input[data-box-index="${index}"]`) as HTMLInputElement
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
    height: 100,
    color: '#c0c0c0',
    text: '',
    borderSize: 2
  }
  boxes.value.push(newBox)
}

const clearAllBoxes = () => {
  boxes.value = []
  connections.value = []
}

const deleteBox = (index: number) => {
  if (index >= 0 && index < boxes.value.length) {
    // 刪除與該框框相關的所有連接線
    connections.value = connections.value.filter(connection => 
      connection.fromBox !== index && connection.toBox !== index
    )

    connections.value.forEach(connection => {
      if (connection.fromBox > index) {
        connection.fromBox--
      }
      if (connection.toBox > index) {
        connection.toBox--
      }
    })

    // 刪除框框
    boxes.value.splice(index, 1)

    // 如果刪除的是當前選中的框框，關閉控制面板
    if (selectedBoxIndex.value === index) {
      selectedBoxIndex.value = -1
      showControlPanel.value = false
    }
  }
}

const selectBox = (index: number) => {
  selectedBoxIndex.value = index
  showControlPanel.value = true
}

const updateBoxProperty = (property: keyof Box, value: any) => {
  if (selectedBoxIndex.value >= 0 && selectedBoxIndex.value < boxes.value.length) {
    const box = boxes.value[selectedBoxIndex.value]
    if (box) {
      (box as any)[property] = value
    }
  }
}

const handleColorChange = (event: Event) => {
  const target = event.target as HTMLInputElement
  updateBoxProperty('color', target.value)
}

const handleTextChange = (event: Event) => {
  const target = event.target as HTMLInputElement
  updateBoxProperty('text', target.value)
}

const handleBorderSizeChange = (event: Event) => {
  const target = event.target as HTMLInputElement
  updateBoxProperty('borderSize', parseInt(target.value))
}

const saveImage = async () => {
  if (!baseImageRef.value) {
    alert('PLEASE UPLOAD IMG FIRST')
    return
  }

  try {
    // 創建一個 canvas 元素
    const canvas = document.createElement('canvas')
    const ctx = canvas.getContext('2d')
    if (!ctx) return

    // 設置 canvas 尺寸與圖片相同
    const img = baseImageRef.value
    canvas.width = img.naturalWidth
    canvas.height = img.naturalHeight

    // 繪製底圖
    ctx.drawImage(img, 0, 0)

    // 計算縮放比例
    const scaleX = img.naturalWidth / img.offsetWidth
    const scaleY = img.naturalHeight / img.offsetHeight

    // 繪製框框
    boxes.value.forEach(box => {
      const x = box.x * scaleX
      const y = box.y * scaleY
      const width = box.width * scaleX
      const height = box.height * scaleY

      // 繪製框框邊框
      ctx.strokeStyle = box.color
      ctx.lineWidth = box.borderSize * scaleX
      ctx.strokeRect(x, y, width, height)

      // 繪製框框文字
      if (box.text) {
        ctx.fillStyle = box.color
        ctx.font = `${12 * scaleX}px monospace`
        ctx.textAlign = 'center'
        ctx.textBaseline = 'middle'
        ctx.fillText(box.text, x + width / 2, y + height / 2)
      }

      // 繪製框框圖片
      if (box.image) {
        const boxImg = new Image()
        boxImg.onload = () => {
          ctx.drawImage(boxImg, x, y, width, height)
        }
        boxImg.src = box.image
      }
    })

    // 繪製連線
    connections.value.forEach(connection => {
      ctx.strokeStyle = '#c0c0c0'
      ctx.lineWidth = 2 * scaleX
      ctx.beginPath()
      ctx.moveTo(connection.x1 * scaleX, connection.y1 * scaleY)
      ctx.lineTo(connection.x2 * scaleX, connection.y2 * scaleY)
      ctx.stroke()
    })

    // 轉換為圖片並下載
    canvas.toBlob((blob) => {
      if (blob) {
        const url = URL.createObjectURL(blob)
        const a = document.createElement('a')
        a.href = url
        a.download = `cctv-editor-${Date.now()}.png`
        document.body.appendChild(a)
        a.click()
        document.body.removeChild(a)
        URL.revokeObjectURL(url)
      }
    }, 'image/png')
  } catch (error) {
    console.error('SAVE IMG FAILED:', error)
    alert('SAVE IMG FAILED, PLEASE TRY AGAIN')
  }
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
  border: 2px dashed #c0c0c0;
  border-radius: 8px;
  padding: 2rem;
  text-align: center;
  background: rgba(192, 192, 192, 0.05);
  cursor: pointer;
  transition: all 0.3s ease;
  min-height: 400px;
  display: flex;
  align-items: center;
  justify-content: center;
}

.upload-area:hover {
  border-color: #e0e0e0;
  background: rgba(192, 192, 192, 0.1);
}

.upload-placeholder {
  color: #c0c0c0;
}

.upload-icon {
  opacity: 0.7;
  width: 60px;
  height: 60px;
  background-color: rgba(0, 0, 0, .3);
  display: flex;
  align-items: center;
  justify-content: center;
  border-radius: 12px;
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
  position: relative;
}

.remove-base-image-btn {
  position: absolute;
  top: 10px;
  right: 10px;
  cursor: pointer;
}

.box-delete-btn {
  position: absolute;
  top: -8px;
  right: -8px;
  border-radius: 50%;
  cursor: pointer;
  display: flex;
  align-items: center;
  justify-content: center;
  z-index: 20;
  opacity: 0;
  transition: opacity 0.2s ease;
}

.box-delete-btn img {
  width: 12px;
  height: 12px;
  filter: brightness(0) invert(1);
}

.detection-box:hover .box-delete-btn {
  opacity: 1;
}

.box-delete-btn:hover {
  background: #ff6666;
  transform: scale(1.1);
}

.box-text {
  position: absolute;
  top: 0;
  left: 0;
  font-size: 12px;
  color: white;
  text-align: center;
  pointer-events: none;
  max-width: calc(100% - 10px);
  word-wrap: break-word;
}

.box-control-panel {
  position: fixed;
  top: 20px;
  right: 20px;
  width: 300px;
  background: rgba(0, 0, 0, 0.9);
  border: 2px solid #c0c0c0;
  border-radius: 8px;
  padding: 0;
  z-index: 1000;
}

.panel-header {
  display: flex;
  justify-content: space-between;
  align-items: center;
  padding: 15px;
  border-bottom: 1px solid #c0c0c0;
  background: rgba(192, 192, 192, 0.1);
}

.panel-header h3 {
  color: #c0c0c0;
  margin: 0;
  font-size: 16px;
}

.close-panel-btn {
  background: transparent;
  border: 1px solid #c0c0c0;
  color: #c0c0c0;
  width: 4px;
  border-radius: 50%;
  cursor: pointer;
  font-size: 16px;
  display: flex;
  align-items: center;
  justify-content: center;
}

.close-panel-btn:hover {
  background: #c0c0c0;
  color: #000;
}

.panel-content {
  padding: 15px;
}

.control-group {
  margin-bottom: 15px;
}

.control-group:last-child {
  margin-bottom: 0;
}

.control-group label {
  display: block;
  color: #c0c0c0;
  margin-bottom: 5px;
  font-size: 14px;
  font-weight: bold;
}

.control-group input[type="color"] {
  width: 100%;
  height: 40px;
  border: 1px solid #c0c0c0;
  background: transparent;
  cursor: pointer;
}

.control-group input[type="text"] {
  width: 100%;
  padding: 8px;
  background: rgba(0, 0, 0, 0.8);
  border: 1px solid #c0c0c0;
  color: #c0c0c0;
  border-radius: 4px;
  font-family: inherit;
}

.control-group input[type="text"]:focus {
  outline: none;
  box-shadow: 0 0 5px #c0c0c0;
}

.control-group input[type="range"] {
  width: 100%;
  height: 5px;
  background: rgba(192, 192, 192, 0.2);
  outline: none;
  border-radius: 5px;
  cursor: pointer;
}

.control-group input[type="range"]::-webkit-slider-thumb {
  appearance: none;
  width: 15px;
  height: 15px;
  background: #c0c0c0;
  border-radius: 50%;
  cursor: pointer;
}

.control-group input[type="range"]::-moz-range-thumb {
  width: 15px;
  height: 15px;
  background: #c0c0c0;
  border-radius: 50%;
  cursor: pointer;
  border: none;
}

.save-btn {
  width: fit-content;
  padding: 12px;
  background: linear-gradient(45deg, #c0c0c0, #e0e0e0);
  color: #000;
  border: none;
  border-radius: 6px;
  cursor: pointer;
  font-family: inherit;
  font-size: 14px;
  font-weight: bold;
  transition: all 0.3s ease;
  text-transform: uppercase;
  letter-spacing: 1px;
}

.save-btn:hover {
  background: linear-gradient(45deg, #e0e0e0, #c0c0c0);
  transform: translateY(-2px);
  box-shadow: 0 4px 12px rgba(192, 192, 192, 0.4);
}

.save-btn:active {
  transform: translateY(0);
  box-shadow: 0 2px 6px rgba(192, 192, 192, 0.3);
}

.detection-box {
  position: absolute;
  border: 2px solid #c0c0c0;
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
  box-shadow: 0 0 10px #c0c0c0;
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
  background: #c0c0c0;
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
  box-shadow: 0 0 8px #c0c0c0;
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
  stroke: #c0c0c0;
  stroke-width: 2;
  fill: none;
  stroke-dasharray: none;
}

.connection-line.dragging {
  stroke: #c0c0c0;
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
  color: #c0c0c0;
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
  background: #c0c0c0;
  color: #000;
  transform: scale(1.1);
  box-shadow: 0 0 10px #c0c0c0;
}

.box-image {
  width: 100%;
  height: 100%;
  object-fit: cover;
  border-radius: 2px;
}





/* 科幻監控風格效果 */
.scan-grid {
  position: absolute;
  top: 0;
  left: 0;
  width: 100%;
  height: 100%;
  background-image: 
    linear-gradient(rgba(192, 192, 192, 0.1) 1px, transparent 1px),
    linear-gradient(90deg, rgba(192, 192, 192, 0.1) 1px, transparent 1px);
  background-size: 20px 20px;
  pointer-events: none;
  z-index: 1;
  animation: gridPulse 3s ease-in-out infinite;
}

.scan-line {
  position: absolute;
  top: 0;
  left: 0;
  width: 100%;
  height: 2px;
  background: linear-gradient(90deg, transparent, #c0c0c0, transparent);
  pointer-events: none;
  z-index: 3;
  box-shadow: 0 0 10px #c0c0c0;
}

.scan-status-panel {
  position: absolute;
  bottom: 20px;
  left: 50%;
  transform: translateX(-50%);
  background: rgba(0, 0, 0, 0.8);
  border: 1px solid #c0c0c0;
  padding: 10px 20px;
  display: flex;
  align-items: center;
  gap: 15px;
  z-index: 10;
  font-family: 'Courier New', monospace;
}

.scan-text {
  color: #c0c0c0;
  font-size: 12px;
  font-weight: bold;
  letter-spacing: 2px;
}

.progress-bar {
  width: 100px;
  height: 4px;
  background: rgba(192, 192, 192, 0.2);
  border: 1px solid #c0c0c0;
  position: relative;
  overflow: hidden;
}

.progress-fill {
  position: absolute;
  top: 0;
  left: 0;
  height: 100%;
  background: linear-gradient(90deg, #c0c0c0, #e0e0e0);
  width: 60%;
}

.scan-indicators {
  display: flex;
  align-items: center;
  gap: 8px;
}

.indicator-dot {
  width: 8px;
  height: 8px;
  background: #c0c0c0;
  border-radius: 50%;
}

.indicator-lines {
  width: 20px;
  height: 2px;
  background: repeating-linear-gradient(
    90deg,
    #c0c0c0 0px,
    #c0c0c0 2px,
    transparent 2px,
    transparent 4px
  );
}

.edge-rulers {
  position: absolute;
  top: 0;
  left: 0;
  width: 100%;
  height: 100%;
  pointer-events: none;
  z-index: 1;
}

.ruler-left,
.ruler-right {
  position: absolute;
  top: 0;
  width: 20px;
  height: 100%;
  border: 1px solid rgba(192, 192, 192, 0.3);
}

.ruler-left {
  left: 0;
}

.ruler-right {
  right: 0;
}

.ruler-mark {
  position: absolute;
  left: 0;
  width: 100%;
  height: 1px;
  background: rgba(192, 192, 192, 0.5);
}

.ruler-left .ruler-mark {
  left: 15px;
  width: 5px;
}

.ruler-right .ruler-mark {
  right: 15px;
  width: 5px;
}

.corner-markers {
  position: absolute;
  top: 0;
  left: 0;
  width: 100%;
  height: 100%;
  pointer-events: none;
  z-index: 1;
}

.corner-marker {
  position: absolute;
  width: 20px;
  height: 20px;
  border: 2px solid #c0c0c0;
}

.corner-marker.top-left {
  top: 10px;
  left: 10px;
  border-right: none;
  border-bottom: none;
}

.corner-marker.top-right {
  top: 10px;
  right: 10px;
  border-left: none;
  border-bottom: none;
}

.corner-marker.bottom-left {
  bottom: 10px;
  left: 10px;
  border-right: none;
  border-top: none;
}

.corner-marker.bottom-right {
  bottom: 10px;
  right: 10px;
  border-left: none;
  border-top: none;
}

/* 動畫效果 */
@keyframes gridPulse {
  0%, 100% { opacity: 0.3; }
  50% { opacity: 0.6; }
}

</style>
