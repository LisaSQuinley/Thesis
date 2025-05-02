<template>
  <div class="hello">
    <h1 class="droughts" ref="droughtsText">Droughts</h1>
    <div class="floods-fires-line">
      <h1 class="floods" ref="floodsText">Floods</h1>
      <h4 class="rotated">and</h4>
      <div class="fires-placeholder" ref="firesText"></div>
    </div>
    <h4 class="subtitle">A Case Study of Morocco and Climate Change</h4>
    <h5 class="byline">Lisa Sakai Quinley</h5>
  </div>

  <canvas ref="noiseCanvas" style="display:none;"></canvas>

  <!-- Droughts mask applied via SVG -->
  <svg ref="droughtsSvg" class="droughts-svg">
    <defs>
      <clipPath id="droughts-clip">
        <text x="0" :y="droughtsHeight * 0.75" :font-size="droughtsHeight" font-family="Parkinsans" font-weight="800">
          DROUGHTS
        </text>
      </clipPath>
    </defs>
    <image :href="noiseDataUrl" width="1000.740" height="176" clip-path="url(#droughts-clip)" />
  </svg>
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
</template>

<script setup>
import { onMounted, onUnmounted, ref, nextTick } from 'vue'
import * as d3 from 'd3'

// Refs to DOM nodes
const firesSvg = ref(null)
const sparkGroup = ref(null)
const firesText = ref(null)
const clipText = ref(null)

const floodsSvg = ref(null)
const floodsText = ref(null)
const raindropGroup = ref(null)
const raindropRect = ref(null)

let sparkTimer = null
let firesObserver = null
let floodsObserver = null

let droughtColorTimer = null

const droughtsSvg = ref(null)
const noiseCanvas = ref(null)
const noiseDataUrl = ref('')
const droughtsText = ref(null)
let droughtsHeight = 176 // or calculate from DOM

onMounted(async () => {

  if (droughtColorTimer) droughtColorTimer.stop();

  const droughtColors = d3.scaleLinear()
    .domain([0, 0.25, 0.5, 0.75, 1])
    .range(["#DEB887", "#d4bb97", "#e2d3b4", "#d4bb97", "#DEB887"]);

  droughtColorTimer = d3.timer((elapsed) => {
    const t = (Math.sin(elapsed / 3000) + 1) / 2; // smooth oscillation between 0 and 1
    const color = droughtColors(t);
    if (droughtsText.value) {
      droughtsText.value.style.color = color;
    }
  });

  // Draw background noise for Droughts
  drawBackgroundNoise();

  // Position the DROUGHTS SVG directly above the Droughts heading
  const droughtsSvgNode = droughtsSvg.value; // Renamed to avoid conflict
  const droughtsTextRect = droughtsText.value.getBoundingClientRect();

  // Set the position of the DROUGHTS SVG to be directly above the heading
  droughtsSvgNode.style.position = 'absolute';
  droughtsSvgNode.style.left = `${droughtsTextRect.left+300}px`;
  droughtsSvgNode.style.top = `${droughtsTextRect.top+17}px`; // Adjust the distance as needed
  droughtsSvgNode.style.width = `${droughtsTextRect.width}px`;
  droughtsSvgNode.style.height = '176px'; // Match the height of the SVG

  // Set the clipPath's position relative to the SVG height
  clipText.value.setAttribute('x', 0);
  clipText.value.setAttribute('y', 176);
  clipText.value.setAttribute('font-size', 176);

  function drawBackgroundNoise() {
    const canvas = noiseCanvas.value
    const ctx = canvas.getContext('2d')
    const width = 1000.740 // set based on text bounds
    const height = 176

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

  drawBackgroundNoise();

  // Position FIRES SVG
  const svgNode = firesSvg.value
  const group = d3.select(sparkGroup.value)
  const width = 550
  const height = 176
  const { x, y } = firesText.value.getBoundingClientRect()

  svgNode.setAttribute('width', width)
  svgNode.setAttribute('height', height)
  svgNode.style.position = 'fixed'
  svgNode.style.left = `${x + 150}px`
  svgNode.style.top = `${y + 42}px`

  clipText.value.setAttribute('x', 0)
  clipText.value.setAttribute('y', height * 0.75)
  clipText.value.setAttribute('font-size', height)

  // Position FLOODS SVG
  const floodsNode = floodsSvg.value
  const floodsRectBox = floodsText.value.getBoundingClientRect()
  floodsNode.setAttribute('width', floodsRectBox.width)
  floodsNode.setAttribute('height', floodsRectBox.height)
  floodsNode.style.position = 'fixed'
  floodsNode.style.left = `${floodsRectBox.left}px`
  floodsNode.style.top = `${floodsRectBox.top}px`

  raindropRect.value.setAttribute('width', floodsRectBox.width)
  raindropRect.value.setAttribute('height', floodsRectBox.height)

  // Wait for DOM to stabilize
  await nextTick()

  // Droughts Intersection Observer
  const droughtsObserver = new IntersectionObserver(([entry]) => {
    if (entry.isIntersecting) {
      droughtsSvg.value.style.display = 'block'
    } else {
      droughtsSvg.value.style.display = 'none'
    }
  }, { threshold: 0.1 })
  if (droughtsText.value instanceof Element) {
    droughtsObserver.observe(droughtsText.value)
  }

  // Fires Intersection Observer
  firesObserver = new IntersectionObserver(([entry]) => {
    if (entry.isIntersecting && !sparkTimer) {
      sparkTimer = createSparks(group, svgNode)
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

  // Floods Intersection Observer
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
})

onUnmounted(() => {
  if (sparkTimer) sparkTimer.stop()
  if (firesObserver) firesObserver.disconnect()
  if (floodsObserver) floodsObserver.disconnect()
  if (droughtColorTimer) droughtColorTimer.stop()
  if (droughtsSvg.value) {
    droughtsSvg.value.remove()
  }
})

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

function renderRaindrops(svgEl, numDrops = 100) {
  if (!svgEl) return

  const width = 699.953
  const height = 176
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
    dropPath.moveTo(0, dropHeight / 2) // start at bottom (rounded part)
    dropPath.bezierCurveTo(
      dropWidth / 2, dropHeight / 2,
      dropWidth / 2, -dropHeight / 4,
      0, -dropHeight / 2 // pointy top
    )
    dropPath.bezierCurveTo(
      -dropWidth / 2, -dropHeight / 4,
      -dropWidth / 2, dropHeight / 2,
      0, dropHeight / 2
    )

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

.hello {
  display: flex;
  flex-direction: column;
  align-items: flex-start;
  margin-bottom: 150px;
  position: relative;
  overflow: visible;
  /* Makes sure SVG is positioned over the text */
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

/* === Colored Headings === */

/* 
.droughts {
  color: #DEB887;
}
 */

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
</style>
