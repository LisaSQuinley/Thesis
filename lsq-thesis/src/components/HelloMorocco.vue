<template>
  <div class="hello">
    <h1 class="droughts">Droughts</h1>
    <div class="floods-fires-line">
      <h1 class="floods" ref="floodsText">Floods</h1>
      <h2 class="rotated">and</h2>
      <div class="fires-placeholder" ref="firesText"></div>
    </div>
    <h2 class="subtitle">A Case Study of Morocco and Climate Change</h2>
    <h3 class="byline">Lisa Sakai Quinley</h3>
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

onMounted(async () => {
  // Position FIRES SVG
  const svgNode = firesSvg.value
  const group = d3.select(sparkGroup.value)
  const width = 465.13
  const height = 160
  const { x, y } = firesText.value.getBoundingClientRect()

  svgNode.setAttribute('width', width)
  svgNode.setAttribute('height', height)
  svgNode.style.position = 'fixed'
  svgNode.style.left = `${x + 135}px`
  svgNode.style.top = `${y + 40}px`

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

  const width = 636
  const height = 150
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
    dropPath.moveTo(0, -dropHeight / 2)
    dropPath.bezierCurveTo(
      dropWidth / 2, -dropHeight / 2,
      dropWidth / 2, dropHeight / 4,
      0, dropHeight / 2
    )
    dropPath.bezierCurveTo(
      -dropWidth / 2, dropHeight / 4,
      -dropWidth / 2, -dropHeight / 2,
      0, -dropHeight / 2
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
  font-size: 10em;
  margin: 0;
  line-height: 1;
}

h2 {
  text-transform: uppercase;
  font-size: 3em;
  margin: 0;
}

.subtitle {
  margin-top: 0.25em;
}

.byline {
  font-size: 1.5em;
  margin-top: 0.75em;
  font-weight: 300;
}

.floods-fires-line {
  display: flex;
  align-items: baseline;
  gap: 0.5em;
}

.rotated {
  transform: rotate(-90deg);
  transform-origin: center;
  font-size: 3.5em;
  align-self: center;
  position: relative;
  top: 0.15em;
}

/* === Colored Headings === */
.droughts {
  color: #DEB887;
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
  width: 465.13px;
  height: 160px;
  position: relative;
}
</style>
