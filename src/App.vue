<script setup lang="ts">
import { ref, onMounted } from 'vue'
import ImageEditor from './components/ImageEditor.vue'
import demo1 from './assets/demo1.jpg'
import demo2 from './assets/demo2.jpg'
import demo3 from './assets/demo3.jpg'
import demo4 from './assets/demo4.jpg'
const backgroundImages = [demo1, demo2, demo3, demo4]

const isLoaded = ref(false)
const mousePosition = ref({ x: 0, y: 0 })
const isHovering = ref(false)

const handleMouseMove = (event: MouseEvent) => {
  const rect = (event.target as HTMLImageElement).getBoundingClientRect()
  mousePosition.value = {
    x: event.clientX - rect.left,
    y: event.clientY - rect.top
  }
}

const handleMouseEnter = () => {
  isHovering.value = true
}

const handleMouseLeave = () => {
  isHovering.value = false
}

onMounted(() => {
  // 確保頁面載入時回到最上方
  window.scrollTo(0, 0)
  document.documentElement.scrollTop = 0
  document.body.scrollTop = 0
  
  isLoaded.value = true
})
</script>

<template>
  <div 
    class="demo"
    @mousemove="handleMouseMove"
    @mouseenter="handleMouseEnter"
    @mouseleave="handleMouseLeave"
  >
    <img 
      src="./assets/street.jpg" 
      alt="Demo 1" 
    />
    <!-- 綠色框線 -->
    <div 
      v-if="isHovering" 
      class="hover-box"
      :style="{
        left: (mousePosition.x - 50) + 'px',
        top: (mousePosition.y - 50) + 'px'
      }"
    />
  </div>
  <div class="app">
    <header class="header">
      <h1>CCTV STYLE IMAGE EDITOR</h1>
      <p>Upload image, click to create boxes, freely adjust size</p>
    </header>
    
    <main class="main">
      <!-- 背景 -->
      <div class="slider">
        <div class="slides">
          <img v-for="(image, index) in backgroundImages" :key="index" :src="image" alt="Background" />
          <!-- 重複一遍以實現無限滾動 -->
          <img v-for="(image, index) in backgroundImages" :key="index" :src="image" alt="Background" />
        </div>
      </div>
      <ImageEditor v-if="isLoaded" />
    </main>
  </div>
</template>

<style>
* {
  margin: 0;
  padding: 0;
  box-sizing: border-box;
}

html, body {
  font-family: 'Courier New', monospace;
  background: black;
  color: #e0e0e0;
  overflow-x: hidden;
  scroll-behavior: smooth;
}

body {
  margin: 0;
  padding: 0;
  scroll-behavior: smooth;
}

.demo {
  width: 100%;
  position: relative;
  display: inline-block;
}

.demo img {
  width: 100%;
  height: 100vh;
  display: block;
}

.hover-box {
  position: absolute;
  width: 200px;
  height: 200px;
  border: 2px solid #01FF06;
  pointer-events: none;
  z-index: 10;
  box-shadow: 0 0 10px rgba(1, 255, 6, 0.5);
  /* transition: all 0.2s ease; */
}


.app {
  min-height: 100vh;
  width: 100%;
  margin: 0 auto;
}

.header {
  text-align: center;
  padding: 2rem;
  border-bottom: 2px solid #c0c0c0;
  background: black;
}

.header h1 {
  font-size: 2.5rem;
  margin-bottom: 0.5rem;
  text-shadow: 0 0 10px #c0c0c0;
  letter-spacing: 2px;
}

.header p {
  font-size: 1.2rem;
  opacity: 0.8;
  letter-spacing: 1px;
}

.main {
  padding: 2rem;
  min-height: calc(100vh - 200px);
  position: relative;
}

.slider {
  position: absolute;
  top: 0;
  left: 0;
  width: 100%;
  height: 100%;
  overflow: hidden;
}

.slides {
  display: flex;
  width: calc(100% * 4); /* 8 張圖 (4張重複) */
  height: 100%;
  animation: scroll 20s linear infinite;
}

  .slides img {
    width: 12.5%; /* 100% / 8 = 12.5% */
    height: 100%;
    object-fit: cover;
    opacity: 0.7;
  }


@keyframes scroll {
  0% {
    transform: translateX(0);
  }
  100% {
    transform: translateX(-50%);
  }
}
</style>
