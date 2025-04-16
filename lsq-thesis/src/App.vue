<template>
  <div id="scrollyteller">
    <HelloMorocco msg="Droughts! Floods! Fires!" />

    <div
      v-for="(stepComponent, index) in steps"
      :key="index"
      class="charts-wrapper"
      v-intersect="() => setActiveStep(index)"
    >
      <component :is="stepComponent" />
    </div>

    <div class="footer">
      <p>
        To check out other projects Lisa has created, visit
        <a href="https://lsq.design" target="_blank" rel="noopener">lsq.design</a>.
      </p>
    </div>
  </div>
</template>

<script>
import HelloMorocco from './components/HelloMorocco.vue'
import MoroccoMapView from './components/MoroccoMapView.vue'
import RainfallPrecipitation from './components/RainfallPrecipitation.vue'
import MeanTemperature from './components/MeanTemperature.vue'
import SheepCattleHerds from './components/SheepCattleHerds.vue'
import LandPloughedWheat from './components/LandPloughedWheat.vue'
import WorldMapViewTimeline from './components/WorldMapViewTimeline.vue'
import INFORMRiskIndex from './components/INFORMRiskIndex.vue'
import CopingMechanisms from './components/CopingMechanisms.vue'

export default {
  name: 'App',
  components: {
    HelloMorocco,
    MoroccoMapView,
    RainfallPrecipitation,
    MeanTemperature,
    SheepCattleHerds,
    LandPloughedWheat,
    CopingMechanisms,
    WorldMapViewTimeline,
    INFORMRiskIndex,
  },
  data() {
    return {
      steps: [
        'MoroccoMapView',
        'RainfallPrecipitation',
        'MeanTemperature',
        'SheepCattleHerds',
        'LandPloughedWheat',
        'CopingMechanisms',
        'INFORMRiskIndex',
        'WorldMapViewTimeline',
      ],
      activeStep: 0, // Start from 0
    }
  },
  mounted() {
    window.addEventListener('keydown', this.handleKeyPress)
  },
  beforeUnmount() {
    window.removeEventListener('keydown', this.handleKeyPress)
  },
  methods: {
    setActiveStep(index) {
  if (index === this.activeStep) return // prevent duplicate updates
  this.activeStep = index
}
,
handleKeyPress(e) {
  if (e.code === 'Space') e.preventDefault()

  if (e.code === 'ArrowDown' || e.code === 'Space') {
    this.goToStep(this.activeStep + 1)
  } else if (e.code === 'ArrowUp') {
    this.goToStep(this.activeStep - 1)
  }
}
,
    goToStep(index) {
  const stepEls = document.querySelectorAll('.charts-wrapper')
  if (index < 0 || index >= stepEls.length) return

  const el = stepEls[index]
  if (el) {
    el.scrollIntoView({ behavior: 'smooth', block: 'start' })
    this.activeStep = index
  }
}
,
  },
  directives: {
    intersect: {
      mounted(el, binding) {
        const observer = new IntersectionObserver(
          ([entry]) => {
            if (entry.isIntersecting) {
              binding.value()
            }
          },
          {
            rootMargin: '0px',
            threshold: 0.5,
          }
        )
        observer.observe(el)
      },
    },
  },
}
</script>


<style>

body, html, #app {
  height: 100%;
  margin: 0;
  scroll-behavior: smooth;
  overflow-x: hidden;
}

.charts-wrapper {
  height: 100vh;
  width: 100vw;
  display: flex;
  justify-content: center;
  align-items: center;
  flex-wrap: wrap;
  padding: 20px;
  box-sizing: border-box;
}


#app {
  font-family: "Parkinsans", sans-serif;
  /* font-family: Avenir, Helvetica, Arial, sans-serif; */
  -webkit-font-smoothing: antialiased;
  -moz-osx-font-smoothing: grayscale;
  text-align: center;
  color: #2c3e50;
  margin-top: 30px;
}

.footer {
  width: 100%;
  margin: 0px;
  text-align: center;
  padding: 10px 0;
}

h1 {
  font-size: 2.6em;
  margin: 0;
  font-weight: 800;
}
h2 {
  font-size: 1.4em;
  margin: 0;
  font-weight: 300;
}
h3 {
  font-size: 1.2em;
  margin: 0;
}
h4 {
  font-size: 1em;
  margin: 0;
}
h5 {
  font-size: 0.8em;
  margin: 0;
}
h6 {
  font-size: 0.6em;
  margin: 0;
}
p {
  font-size: 1em;
  margin: 0;
  text-align: left;
}
text {
  font-family: "Parkinsans", sans-serif;
  margin: 0;
}

</style>
