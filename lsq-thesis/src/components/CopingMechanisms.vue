<template>
  <div class="coping-container">
    <h3 class="title">Climate Survival Kit</h3>
    <div class="image-viewer">
      <img
        :src="tabs[selectedTab].image"
        :alt="tabs[selectedTab].title"
        class="fade-image"
      />
    </div>
  </div>
</template>

<script setup>
import { ref, onMounted, onUnmounted } from 'vue'

import phase1 from '@/assets/coping-mechanisms/Coping-Mechanisms_Phase-1.png'
import phase2 from '@/assets/coping-mechanisms/Coping-Mechanisms_Phase-2.png'
import phase3 from '@/assets/coping-mechanisms/Coping-Mechanisms_Phase-3.png'
import phase4 from '@/assets/coping-mechanisms/Coping-Mechanisms_Phase-4.png'

const tabs = [
  { title: 'Dams: Water Storage', image: phase1 },
  { title: 'Wells', image: phase2 },
  { title: 'Traditional water management', image: phase3 },
  { title: 'Desalination', image: phase4 },
  { title: 'Other Methods: Water Conservation', image: phase4 },
]

const selectedTab = ref(0)

let interval

onMounted(() => {
  interval = setInterval(() => {
    selectedTab.value = (selectedTab.value + 1) % tabs.length
  }, 5000) // Change phase every 3 seconds
})

onUnmounted(() => {
  clearInterval(interval)
})
</script>

<style scoped>
.coping-container {
  padding: 4rem 5rem 5rem 5rem;
  position: relative;
  height: calc(100vh - 10rem);
  width: 100vw;
  overflow: hidden; /* no scroll anywhere */
}

.title {
  position: absolute;
  font-weight: 800;
  padding: 5px 10px;
  top: 4rem;
  left: 5rem;
  font-size: 1.2em;
  z-index: 2;
  text-transform: uppercase;
  color: #ffffffb6;
  text-transform: uppercase;
  line-height: 1;
  font-size: 5em;
  position: absolute;
  text-align: left;
  margin: 0;
  z-index: 999;
}

.image-viewer {
  display: flex;
  justify-content: center;
  align-items: center;
  width: 100%;
  height: 100%;
  overflow: hidden;
}

.image-viewer img {
  width: 100%;
  height: 100%;
  object-fit: cover; /* Fill the container without stretching */
  transition: opacity 1s ease-in-out;
  opacity: 0;
}

.image-viewer img.fade-image {
  opacity: 1;
}
</style>
