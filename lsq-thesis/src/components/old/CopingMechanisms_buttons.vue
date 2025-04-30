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
  padding: 4rem 5rem 5rem 5rem;
  position: relative;
  height: calc(100vh - 10rem);
  width: 100vw;
  overflow: hidden; /* no scroll anywhere */
}

.title {
  position: absolute;
  padding: 5px 15px;
  top: 4rem;
  left: 5rem;
  font-size: 1.2em;
  z-index: 2;
}

.tabs {
  position: absolute;
  top: 4rem;
  right: 5rem;
  display: flex;
  gap: 0.5rem;
  z-index: 2;
}

.tabs button {
  font-family: "Parkinsans", sans-serif;
  background: rgba(255, 255, 255, 0.7);
  border: none;
  padding: 0.5rem 1rem;
  cursor: pointer;
  transition: all 0.3s ease;
  font-weight: bold;
  color: #2c3e50;
}

.tabs button.active {
  background: #089d9d82;
  color: white;
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
  object-fit: contain; /* Ensures the image fits without being cropped */
  transition: all 0.3s ease;
}

</style>