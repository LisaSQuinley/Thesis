<template>
  <div class="coping-container" ref="containerRef">
    <h3 class="title">Climate Survival Kit</h3>

    <div class="video-viewer">
      <video ref="copingVideo" loop muted playsinline class="coping-video">
        <source src="@/assets/coping-mechanisms/Coping-Mechanisms-Animation_Final.mp4" type="video/mp4" />
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
const playbackRate = 0.5 // Fixed playback speed

let observer
let delayTimeout

const DELAY_MS = 1500

const textBySecond = {
  0: 'Morocco is full of oases, mountains, and rivers, with skies that rain down to water the pastures and fields.',
  1: 'Fields and pastures that are green and lush, with sheep and cattle grazing, and wheat growing.',
  2: 'With a little digging, we can find water underground.',
  3: 'Another well can be dug...',
  4: '...and even a dam can be built to store water.',
  5: 'We can increase our crops and livestock.',
  6: 'Our towns and cities can grow.',
  7: 'This can affect our water table.',
  8: 'Our oases can start to dry up, our skies get a little less rain, our mountains get a little less snow.',
  10: 'We may add more wells...',
  11: '...and even more dams.',
  12: 'This can affect our water table.',
  13: 'Global warming can make our skies rain less, which means less snow in our mountains, less water in our rivers, our oases get smaller, and our land is affected too.',
  14: 'This can affect the size of our livestock and the amount of crops we can grow.',
  15: 'But our towns and cities can continue to grow.',
  16: 'But we need more wells and to dig deeper.',
  17: 'More dams and to build bigger ones.',
  18: 'Which can lower our water table.',
  19: 'Our oases can dry up, our skies rain less, our mountains get less snow, our pastures are drier.',
  20: 'Our herds and fields are smaller.',
  21: 'But our cities and towns can continue to grow.',
  22: 'So we dig deeper wells.',
  23: 'Our dams are bigger but reduced to a trickle.',
  24: 'The water table is even lower.',
  25: 'Herds and fields are even smaller, mountains are bare, rivers are drying, skies are dry.',
  26: 'Water conservation is a must, in the cities and near the coast we can desalinate seawater.',
  27: 'If we\'re not careful, our oases can dry up.',
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
      video.playbackRate = playbackRate
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
            video.playbackRate = playbackRate
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
  color: #ffffff96;
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
  max-width: 45%;
  bottom: 1rem;
  left: 1.5rem;
  font-size: 28px;
  color: #ffffff;
  font-weight: 300;
  z-index: 999;
  transition: opacity 0.5s ease-in-out;
}
</style>
