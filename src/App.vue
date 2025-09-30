<script setup lang="ts">
import { ref, onMounted } from 'vue'
import ImageEditor from './components/ImageEditor.vue'

const isLoaded = ref(false)

onMounted(() => {
  isLoaded.value = true
})
</script>

<template>
  <div class="app">
    <header class="header">
      <h1>CCTV Style Image Editor</h1>
      <p>Upload base image, click to create boxes, freely adjust size</p>
    </header>
    
    <main class="main">
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

body {
  font-family: 'Courier New', monospace;
  background: #0a0a0a;
  color: #00ff00;
  overflow-x: hidden;
}

.app {
  min-height: 100vh;
  background: linear-gradient(45deg, #0a0a0a 0%, #1a1a1a 100%);
}

.header {
  text-align: center;
  padding: 2rem;
  border-bottom: 2px solid #00ff00;
  background: rgba(0, 255, 0, 0.05);
}

.header h1 {
  font-size: 2.5rem;
  margin-bottom: 0.5rem;
  text-shadow: 0 0 10px #00ff00;
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
}
</style>
