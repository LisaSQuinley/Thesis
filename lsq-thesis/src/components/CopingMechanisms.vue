<template>
  <div class="coping-container" ref="containerRef">
    <h3 class="title">Climate Survival Kit</h3>

    <div class="video-viewer">
      <video ref="copingVideo" loop muted playsinline class="coping-video">
        <source src="@/assets/coping-mechanisms/Coping-Mechanisms_Animation_Final.mp4" type="video/mp4" />
        Your browser does not support the video tag.
      </video>

      <p class="video-caption">{{ videoText }}</p>

      <button class="video-toggle" @click="togglePlay">
        {{ isPlaying ? 'Pause' : 'Play' }}
      </button>
    </div>
  </div>
</template>

<script setup>
import { ref, onMounted, onUnmounted, computed } from 'vue'

const copingVideo = ref(null)
const containerRef = ref(null)
const isPlaying = ref(false)
const currentSecond = ref(0)

let observer
let delayTimeout

const DELAY_MS = 1500

// Caption text mapped to specific seconds
const textBySecond = {
  0: 'A lush landscape of Morocco...',
  1: 'A lush landscape of Morocco...but this is going to be changed',
  2: 'with sheep and cattle grazing, and wheat growing, but this is going to be changed',
  3: 'a well may be dug to access groundwater',
  4: 'another well and even a dam',
  5: 'and then we have this at 5 seconds',
  6: 'yet another message at 6 seconds, but this is going to be changed',
  7: 'and then we have this at 7 seconds, but this is going to be changed',
  8: 'yet another message at 8 seconds',
  9: 'and then we have this at 9 seconds',
  10: 'yet another message at 10 seconds',
  11: 'and then we have this at 11 seconds',
  12: 'and finally this at 12 seconds, but this is going to be changed',
  13: 'and finally this at 13 seconds, but this is going to be changed',
  14: 'and finally this at 14 seconds, but this is going to be changed',
  15: 'and finally this at 15 seconds, but this is going to be changed',
}

const videoText = computed(() => {
  const seconds = Object.keys(textBySecond).map(Number).sort((a, b) => b - a)
  for (const sec of seconds) {
    if (currentSecond.value >= sec) {
      return textBySecond[sec]
    }
  }
  return ''
})

function togglePlay() {
  const video = copingVideo.value
  if (!video) return

  clearTimeout(delayTimeout)

  if (video.paused) {
    video.play().then(() => {
      isPlaying.value = true
    }).catch(err => {
      console.warn('Playback blocked:', err)
    })
  } else {
    video.pause()
    isPlaying.value = false
  }
}

function handleTimeUpdate() {
  if (copingVideo.value) {
    currentSecond.value = Math.floor(copingVideo.value.currentTime)
  }
}

onMounted(() => {
  if (copingVideo.value) {
    copingVideo.value.addEventListener('timeupdate', handleTimeUpdate)
  }

  observer = new IntersectionObserver(
    ([entry]) => {
      const video = copingVideo.value
      if (!video) return

      if (entry.isIntersecting) {
        delayTimeout = setTimeout(() => {
          video.play().then(() => {
            isPlaying.value = true
          }).catch(err => {
            console.warn('Autoplay blocked on delay:', err)
          })
        }, DELAY_MS)
      } else {
        clearTimeout(delayTimeout)
        video.pause()
        isPlaying.value = false
      }
    },
    { threshold: 0.25 }
  )

  if (containerRef.value) {
    observer.observe(containerRef.value)
  }
})

onUnmounted(() => {
  if (observer && containerRef.value) {
    observer.unobserve(containerRef.value)
  }

  if (copingVideo.value) {
    copingVideo.value.removeEventListener('timeupdate', handleTimeUpdate)
  }

  clearTimeout(delayTimeout)
})
</script>

<style scoped>

html, body {
  margin: 0;
  padding: 0;
  overflow: hidden; /* Prevent scrolling */
  height: 100%;
  width: 100%;
}

.coping-container {
  padding: 4rem 5rem 5rem 5rem;
  position: relative;
  height: calc(100vh - 11rem);
  width: calc(100vw - 14rem);
  overflow: hidden;
}

.title {
  position: absolute;
  font-weight: 800;
  padding: 5px 10px;
  top: 4rem;
  left: 5.25rem;
  font-size: 5em;
  z-index: 999;
  text-transform: uppercase;
  color: #ffffff46;
  line-height: 1;
  text-align: left;
  margin: 0;
}

.video-viewer {
  position: relative;
  display: flex;
  justify-content: center;
  align-items: center;
  height: 100%;
  width: 100%;
  overflow: hidden;
}


.coping-video {
  width: 100%;
  height: 100%;
  object-fit: cover;
  display: block;
}

.video-toggle {
  font-family: 'Parkinsans', sans-serif;
  font-weight: 600;
  position: absolute;
  top: 1rem;
  right: 1rem;
  background: #ffffffb6;
  color: #2c3e50;
  border: none;
  padding: 0.6rem 1.2rem;
  font-size: 1rem;
  cursor: pointer;
  border-radius: 8px;
  z-index: 1000;
  transition: background 0.3s;
}

.video-toggle:hover {
  background: #2c3e50b6;
  color: #ffffff;
}

.video-caption {
  position: absolute;
  line-height: 1.3;
  max-width: 40%;
  top: 0.5rem;
  left: 61rem;
  font-size: 28px;
  color: #ffffff;
  font-weight: 300;
  z-index: 999;
  transition: opacity 0.5s ease-in-out;
}

</style>
