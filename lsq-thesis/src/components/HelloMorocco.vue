<template>
  <div class="hello-morocco">
    <div class="hello">
      <h1 class="droughts" ref="droughtsText">Droughts</h1>
      <div class="floods-fires-line">
        <h1 class="floods" ref="floodsText">Floods</h1>
        <h4 class="rotated">and</h4>
        <div class="fires-placeholder" ref="firesText"></div>
      </div>
      <h4 class="subtitle">A Case Study of Morocco and Climate Change</h4>
      <h5 class="byline">Lisa Sakai Quinley</h5>

      <canvas ref="noiseCanvas" style="display:none;"></canvas>

      <!-- Droughts mask applied via SVG -->
      <svg ref="droughtsSvg" class="droughts-svg">
        <defs>
          <clipPath id="droughts-clip">
            <text x="0" :y="droughtsHeight * 0.75" :font-size="droughtsHeight" font-family="Parkinsans"
              font-weight="800">
              DROUGHTS
            </text>
          </clipPath>
        </defs>
        <image :href="noiseDataUrl" width="1000.740" height="176" clip-path="url(#droughts-clip)" />
      </svg>

      <!-- Sheep and Wheat Animation -->
      <div class="png-animation-container">
        <img v-for="(style, i) in wheatStyles" :key="i" :src="getWheatSrc()" class="png-frame wheat" :style="style"
          alt="Wheat" />
        <img :src="getSheepSrc()" class="png-frame sheep" :style="sheepStyle" alt="Sheep" />
      </div>

      <!-- SVG for Raindrops -->
      <svg ref="floodsSvg" class="floods-svg">
        <g ref="raindropGroup">
          <!-- Raindrops will fall inside this rect, which will match the bounding box of the "FLOODS" heading -->
          <rect ref="raindropRect" width="100%" height="100%" fill="#089c9d" />
        </g>
      </svg>

      <!-- SVG for Sparks -->
      <svg ref="firesSvg" class="fires-svg">
        <defs>
          <clipPath id="text-clip">
            <!-- Dynamically placed text for clipping -->
            <text ref="clipText" font-family="Parkinsans" font-weight="800" id="firesText" fill="black">
              FIRES
            </text>
          </clipPath>
        </defs>
        <g clip-path="url(#text-clip)" ref="sparkGroup">
          <rect width="100%" height="100%" fill="rgb(153, 0, 0)" />
        </g>
      </svg>
    </div>
  </div>
  <div class="morocco-location">
    <l-map ref="map" v-model:zoom="zoom" :center="center" :zoom-control="false" class="morocco-map">
      <l-geo-json ref="geoJsonLayer" v-if="geoJsonData" :geojson="geoJsonData" :options-style="geoJsonStyle" />
    </l-map>
  </div>
</template>

<script setup>
import { onMounted, onUnmounted, ref, nextTick, computed } from 'vue'
import * as d3 from 'd3'
import "leaflet/dist/leaflet.css";
import { LMap, LGeoJson } from "@vue-leaflet/vue-leaflet";

// Refs to DOM nodes
const firesSvg = ref(null)
const sparkGroup = ref(null)
const firesText = ref(null)
const clipText = ref(null)

const floodsSvg = ref(null)
const floodsText = ref(null)
const raindropGroup = ref(null)
const raindropRect = ref(null)

const droughtsSvg = ref(null)
const noiseCanvas = ref(null)
const noiseDataUrl = ref('')
const droughtsText = ref(null)

let sparkTimer = null
let firesObserver = null
let floodsObserver = null
let droughtColorTimer = null

// Droughts specific
let droughtsHeight = 176 // Adjust based on DOM or specific requirements

const zoom = ref(3);
const geoJsonData = ref(null);
const map = ref(null);
const geoJsonLayer = ref(null);
const center = ref([20, -5]);

const highlightMorocco = () => {
  const moroccoFeature = geoJsonLayer.value?.geojson.features.find(
    (f) => f.properties.ADMIN === "Morocco"
  );

  if (!moroccoFeature) {
    console.warn("Morocco not found in GeoJSON.");
    return;
  }

  const layer = geoJsonLayer.value?.leafletObject.getLayers().find((l) =>
    l.feature?.properties.ADMIN === "Morocco"
  );

  if (layer) {
    layer.setStyle({
      fillColor: "#d4bb9749",
      color: "#e2d3b4",
      weight: 2,
      fillOpacity: 0.9,
    });

    const bounds = layer.getBounds();

    const offset = [700, 0];

    map.value?.leafletObject.fitBounds(bounds, {
      maxZoom: 7, // Prevent zooming in too far
      paddingTopLeft: offset, // Apply offset on top-left
    });

    center.value = bounds.getCenter();
    zoom.value = map.value?.leafletObject.getZoom();
  }
};


onMounted(() => {
  loadGeoJsonMap();

  // Zoom and highlight Morocco after 10 seconds
  setTimeout(() => {
    highlightMorocco();
  }, 35000);
});


const loadGeoJsonMap = async () => {
  try {
    const geoJsonResponse = await fetch("./data/country-boundaries.geojson");
    const geoJson = await geoJsonResponse.json();
    geoJsonData.value = geoJson;
    console.log('GeoJSON Data:', geoJsonData.value);  // Debug log
  } catch (error) {
    console.error("Error loading data:", error);
  }
};

const geoJsonStyle = () => {
  return {
    fillColor: "#f5f0e168",
    color: "#e2d3b4",
    weight: 1,
    fillOpacity: 0.7,
  };
};


function getWheatSrc() {
  return new URL('@/assets/wheat/wheat-title.svg', import.meta.url).href;
}
function getSheepSrc() {
  return new URL('@/assets/sheep-intro.png', import.meta.url).href;
}


const animationTriggered = ref(false)
onMounted(() => {
  setTimeout(() => {
    animationTriggered.value = true
  }, 2000)
})

const wheatStyles = computed(() => {
  return Array.from({ length: 10 }, (_, i) => {
    const n = i + 1
    return animationTriggered.value
      ? {
        animation: `wiltDown 3s ease-out forwards`,
        animationDelay: `${(10 - n) * 3}s`
      }
      : {}
  })
})

const sheepStyle = computed(() => {
  return animationTriggered.value
    ? { animation: 'slideInSheep 1s ease-out 1s forwards' }
    : {}
})

// On mounted logic
onMounted(async () => {
  await nextTick()
  if (droughtColorTimer) droughtColorTimer.stop()

  // Drought color animation
  const droughtColors = d3.scaleLinear()
    .domain([0, 0.25, 0.5, 0.75, 1])
    .range(["#DEB887", "#d4bb97", "#e2d3b4", "#d4bb97", "#DEB887"])

  droughtColorTimer = d3.timer((elapsed) => {
    const t = (Math.sin(elapsed / 3000) + 1) / 2 // smooth oscillation
    const color = droughtColors(t)
    if (droughtsText.value) {
      droughtsText.value.style.color = color
    }
  })

  // Initialize background noise for Droughts
  drawBackgroundNoise()

  // Set position for droughts SVG
  setDroughtsSvgPosition()

  // Positioning for Fires SVG
  setFiresSvgPosition()

  // Positioning for Floods SVG
  setFloodsSvgPosition()

  // Wait for DOM to stabilize
  await nextTick()

  // Intersection Observer setup for Droughts, Fires, and Floods
  setupIntersectionObservers()
})

// On unmounted cleanup
onUnmounted(() => {
  if (sparkTimer) sparkTimer.stop()
  if (firesObserver) firesObserver.disconnect()
  if (floodsObserver) floodsObserver.disconnect()
  if (droughtColorTimer) droughtColorTimer.stop()
  if (droughtsSvg.value) {
    droughtsSvg.value.remove()
  }
})

// Functions for positioning and rendering
function setDroughtsSvgPosition() {
  const droughtsSvgNode = droughtsSvg.value
  const droughtsTextRect = droughtsText.value.getBoundingClientRect()

  droughtsSvgNode.style.position = 'absolute'
  droughtsSvgNode.style.left = `${droughtsTextRect.left - 359}px`
  droughtsSvgNode.style.top = `${droughtsTextRect.top - 142}px`
  droughtsSvgNode.style.width = `${droughtsTextRect.width}px`
  droughtsSvgNode.style.height = `${droughtsHeight}px`

  // Set clipText position relative to the SVG height
  clipText.value.setAttribute('x', 0)
  clipText.value.setAttribute('y', droughtsHeight)
  clipText.value.setAttribute('font-size', droughtsHeight)
}

function setFiresSvgPosition() {
  const svgNode = firesSvg.value
  // eslint-disable-next-line no-unused-vars
  const group = d3.select(sparkGroup.value)
  const { x, y } = firesText.value.getBoundingClientRect()

  svgNode.setAttribute('width', 550)
  svgNode.setAttribute('height', 170)
  svgNode.style.position = 'absolute'
  svgNode.style.left = `${x - 360}px`
  svgNode.style.top = `${y - 185}px`

  clipText.value.setAttribute('x', 0)
  clipText.value.setAttribute('y', 170 * 0.75)
  clipText.value.setAttribute('font-size', 170)
}

function setFloodsSvgPosition() {
  const floodsNode = floodsSvg.value
  const floodsRectBox = floodsText.value.getBoundingClientRect()

  floodsNode.setAttribute('width', floodsRectBox.width)
  floodsNode.setAttribute('height', floodsRectBox.height)
  floodsNode.style.position = 'absolute'
  floodsNode.style.left = `-${floodsRectBox.left}px/2`
  floodsNode.style.top = `${floodsRectBox.top - 310}px`

  raindropRect.value.setAttribute('width', floodsRectBox.width)
  raindropRect.value.setAttribute('height', floodsRectBox.height)
}

// Draw background noise for Droughts
function drawBackgroundNoise() {
  const canvas = noiseCanvas.value
  const ctx = canvas.getContext('2d')
  const width = 1000.740
  const height = droughtsHeight

  canvas.width = width
  canvas.height = height

  ctx.fillStyle = 'transparent'
  ctx.fillRect(0, 0, width, height)

  const bladeCount = 2500
  for (let i = 0; i < bladeCount; i++) {
    const x = Math.random() * width
    const y = Math.random() * height
    const length = 8 + Math.random() * 12
    const angle = (-Math.PI / 2) + (Math.random() - 0.5) * 0.6
    const xEnd = x + length * Math.cos(angle)
    const yEnd = y + length * Math.sin(angle)

    ctx.beginPath()
    ctx.moveTo(x, y)
    ctx.lineTo(xEnd, yEnd)
    ctx.strokeStyle = `rgba(139, 69, 19, ${0.04 + Math.random() * 0.3})`
    ctx.lineWidth = 1
    ctx.stroke()
  }

  // Convert to dataURL and assign
  noiseDataUrl.value = canvas.toDataURL()
}

// Setup Intersection Observers for Droughts, Fires, and Floods
function setupIntersectionObservers() {
  // Droughts Observer
  const droughtsObserver = new IntersectionObserver(([entry]) => {
    if (droughtsSvg.value && droughtsText.value) {
      droughtsSvg.value.style.display = entry.isIntersecting ? 'block' : 'none'
    }
  }, { threshold: 0.1 })

  if (droughtsText.value instanceof Element) {
    droughtsObserver.observe(droughtsText.value)
  }

  // Fires Observer
  firesObserver = new IntersectionObserver(([entry]) => {
    if (entry.isIntersecting && !sparkTimer) {
      sparkTimer = createSparks(d3.select(sparkGroup.value), firesSvg.value)
      firesSvg.value.style.display = 'block'
    } else if (!entry.isIntersecting && sparkTimer) {
      sparkTimer.stop()
      sparkTimer = null
      firesSvg.value.style.display = 'none'
    }
  }, { threshold: 0.1 })

  if (firesText.value instanceof Element) {
    firesObserver.observe(firesText.value)
  }

  // Floods Observer
  let isRaining = false
  floodsObserver = new IntersectionObserver(([entry]) => {
    if (entry.isIntersecting && !isRaining) {
      isRaining = true
      floodsSvg.value.style.display = 'block'
      renderRaindrops(floodsSvg.value)
    } else if (!entry.isIntersecting && isRaining) {
      isRaining = false
      d3.select(raindropGroup.value).selectAll('*').interrupt().remove()
      floodsSvg.value.style.display = 'none'
    }
  }, { threshold: 0.1 })

  if (floodsText.value instanceof Element) {
    floodsObserver.observe(floodsText.value)
  }
}

// Function to create sparks
function createSparks(group, svgNode) {
  const { width, height } = svgNode.getBoundingClientRect()

  function spawnSpark() {
    const x1 = Math.random() * width
    const y1 = Math.random() * height
    const x2 = x1 + (Math.random() - 0.5) * 15
    const y2 = y1 + (Math.random() - 0.5) * 15

    const spark = group.append('path')
      .attr('d', `M${x1},${y1} L${x2},${y2}`)
      .attr('stroke', Math.random() < 0.5 ? '#FFFF99' : '#FFFFFF')
      .attr('stroke-width', 1 + Math.random() * 5)
      .attr('stroke-dasharray', '5 5')
      .attr('stroke-dashoffset', 5)
      .attr('opacity', 1)

    spark.transition()
      .duration(200)
      .attr('stroke-dashoffset', 0)
      .attr('opacity', 0)
      .remove()
  }

  return d3.timer(() => {
    for (let i = 0; i < 20; i++) {
      if (Math.random() < 0.7) spawnSpark()
    }
  })
}

// Function to render raindrops
function renderRaindrops(svgEl, numDrops = 100) {
  if (!svgEl) return

  const width = 699.953
  const height = droughtsHeight
  const svg = d3.select(svgEl)

  svg.attr('width', width)
    .attr('height', height)
    .attr('viewBox', `0 0 ${width} ${height}`)

  const group = d3.select(svgEl.querySelector('g'))
  group.selectAll('*').remove()

  const dropWidth = 6
  const dropHeight = 10

  for (let i = 0; i < numDrops; i++) {
    const x = Math.random() * width
    const startY = -Math.random() * 100
    const endY = height + dropHeight

    const delay = Math.random() * 2000
    const duration = 2000 + Math.random() * 2000

    const dropPath = d3.path()
    dropPath.moveTo(0, dropHeight / 2)
    dropPath.bezierCurveTo(dropWidth / 2, dropHeight / 2, dropWidth / 2, -dropHeight / 4, 0, -dropHeight / 2)
    dropPath.bezierCurveTo(-dropWidth / 2, -dropHeight / 4, -dropWidth / 2, dropHeight / 2, 0, dropHeight / 2)

    group.append('path')
      .attr('d', dropPath.toString())
      .attr('fill', '#40E0D0')
      .attr('transform', `translate(${x}, ${startY})`)
      .transition()
      .delay(delay)
      .duration(duration)
      .ease(d3.easeLinear)
      .attr('transform', `translate(${x}, ${endY})`)
      .on('end', function repeat() {
        const newX = Math.random() * width
        d3.select(this)
          .attr('transform', `translate(${newX}, ${-dropHeight})`)
          .transition()
          .delay(Math.random() * 2000)
          .duration(2000 + Math.random() * 2000)
          .ease(d3.easeLinear)
          .attr('transform', `translate(${newX}, ${endY})`)
          .on('end', repeat)
      })
  }
}
</script>

<style scoped>
html,
body {
  height: 100vh;
  width: 100vw;
  display: flex;
  flex-direction: column;
  box-sizing: border-box;
  overflow: hidden;
}

.hello-morocco {
  position: relative;
}

.hello {
  display: flex;
  flex-direction: column;
  align-items: flex-start;
  margin-bottom: 150px;
  position: relative;
  overflow: visible;
  z-index: 1;
}

.leaflet-container {
  background-color: white;
}

.morocco-map {
  height: calc(100vh - 12rem);
  width: calc(100vw - 14rem);
}

.morocco-location {
  position: absolute;
  height: calc(100vh - 12rem);
  width: calc(100vw - 14rem);
  z-index: 0;
  top: 4rem;
}

h1 {
  text-transform: uppercase;
  font-size: 11rem;
  margin: 0;
  line-height: 1;
}

h4 {
  text-transform: uppercase;
  font-size: 54px;
  margin: 0;
  font-weight: 300;
}

.subtitle {
  margin-top: 0.25em;
}

.byline {
  font-size: 28px;
  font-weight: 300;
  margin-top: 0.75em;
}

.floods-fires-line {
  display: flex;
  align-items: baseline;
  gap: 0.75em;
}

.rotated {
  transform: rotate(-90deg);
  transform-origin: center;
  font-size: 3.5em;
  align-self: center;
  position: relative;
  top: 0.15em;
}

.droughts {
  margin-bottom: -40px;
}

.floods {
  color: #089c9d;
}

.floods-svg {
  display: block;
  pointer-events: none;
  position: absolute;
}

.fires {
  color: rgb(153, 0, 0);
  font-family: 'Parkinsans', sans-serif;
}

.fires-placeholder {
  width: 500px;
  height: 176px;
  position: relative;
}

.png-animation-container {
  position: absolute;
  top: 4.3rem;
  left: -0.5rem;
  width: 100%;
  height: 5rem;
  display: flex;
  justify-content: left;
  align-items: flex-end;
  pointer-events: none;
  z-index: 5;
}

.png-frame {
  width: 100px;
  margin-right: -5px;
  height: auto;
  opacity: 1;
  transform: scaleX(1);
  transform-origin: right bottom;
}

.png-frame.wheat {
  opacity: 0.9;
  transform-origin: bottom center;
  z-index: 0;
}

.wilt {
  animation: wiltDown 3s ease-out forwards;
}

.png-frame.sheep {
  padding-left: 25px;
  width: 90px;
  opacity: 0;
  z-index: 1;
}
</style>

<style>
.leaflet-control-attribution {
  opacity: 0.1;
}

@keyframes wiltDown {
  0% {
    opacity: 0.9;
    transform: scaleY(1);
  }

  100% {
    opacity: 0;
    transform: scaleY(0);
  }
}

@keyframes slideInSheep {
  from {
    transform: translateX(100%);
    opacity: 0;
  }

  to {
    transform: translateX(0%);
    opacity: 1;
  }
}

.leaflet-control-zoom {
  display: none;
}
</style>