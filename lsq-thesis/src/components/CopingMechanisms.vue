<template>
    <div class="coping-container">
      <h3 class="title">Coping Mechanisms</h3>
      <div class="tabs">
        <button
          v-for="(tab, index) in tabs"
          :key="index"
          :class="{ active: selectedTab === index }"
          @click="selectedTab = index"
        >
          {{ tab.title }}
        </button>
      </div>
      <div class="image-viewer">
        <img :src="tabs[selectedTab].image" :alt="tabs[selectedTab].title" />
      </div>
    </div>
  </template>  

<script setup>
import { ref } from 'vue'

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
</script>
  
<style scoped>
.coping-container {
  position: relative;
  height: 100%;
  width: 100%;
  overflow: hidden; /* no scroll anywhere */
}

.title {
  position: absolute;
  top: 1rem;
  left: 1rem;
  font-size: 1.5rem;
  color: white;
  text-shadow: 0 0 5px black;
  z-index: 2;
}

.tabs {
  position: absolute;
  top: 1rem;
  left: 50%;
  transform: translateX(-50%);
  display: flex;
  gap: 0.5rem;
  z-index: 2;
}

.tabs button {
  background: rgba(255, 255, 255, 0.7);
  border: none;
  padding: 0.5rem 1rem;
  border-radius: 20px;
  cursor: pointer;
  transition: all 0.3s ease;
  font-weight: bold;
}

.tabs button.active {
  background: rgba(0, 0, 0, 0.7);
  color: white;
}

.image-viewer {
  height: 100%;
  width: 100%;
  display: flex;
  justify-content: center;
  align-items: center;
  overflow: hidden; /* no scroll */
}

.image-viewer img {
  height: 100%;
  width: auto;
  max-width: 100%;
  object-fit: contain;
}
</style>