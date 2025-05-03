<template>
  <div id="scrollyteller">
    <div class="left-tab-container">
      <div v-for="(title, i) in visibleTitles" :key="i" :class="['tab', { active: i + 1 === activeStep }]"
        :style="i + 1 === activeStep ? { color: getTabColor(i), borderLeftColor: getTabColor(i) } : {}"
        @click="goToStep(i + 1)">
        {{ title }}
      </div>
    </div>
    <div v-for="(stepComponent, index) in steps" :key="index" class="charts-wrapper"
      v-intersect="() => setActiveStep(index)">
      <component :is="stepComponent" />
    </div>

    <div class="right-tab-container">
      <div class="tab" :class="{ active: showMethodology }" @click="showMethodology = !showMethodology">
        Methodology
      </div>

      <a class="tab" href="https://lsq.design" target="_blank" rel="noopener">lsq.design</a>
    </div>

    <!-- Methodology Panel -->
    <div v-if="showMethodology" class="methodology-panel">
      <h5>Methodology</h5>
      <h6>Zooming Into Risk</h6>
      <p>This section describes the methodology behind the visualizations...</p>
      <h6>Raincheck Forever</h6>
      <p>You can include multiple <code>&lt;h&gt;</code> and <code>&lt;p&gt;</code> tags here.</p>
      <h6>Future Heat</h6>
      <p>This section describes the methodology behind the visualizations...</p>
      <h6>Pasture Panic</h6>
      <p>You can include multiple <code>&lt;h&gt;</code> and <code>&lt;p&gt;</code> tags here.</p>
      <h6>Wheat Drop</h6>
      <p>This section describes the methodology behind the visualizations...</p>
      <h6>Climate Survival Kit</h6>
      <p>You can include multiple <code>&lt;h&gt;</code> and <code>&lt;p&gt;</code> tags here.</p>
      <h6>Disaster Time Machine</h6>
      <p>You can include multiple <code>&lt;h&gt;</code> and <code>&lt;p&gt;</code> tags here.</p>
    </div>
  </div>
</template>

<script>
import HelloMorocco from './components/HelloMorocco.vue'
import RainfallPrecipitation from './components/RainfallPrecipitation.vue'
import MeanTemperature from './components/MeanTemperature.vue'
import SheepCattleHerds from './components/SheepCattleHerds.vue'
import LandPloughedWheat from './components/LandPloughedWheat.vue'
import INFORMRiskIndex from './components/INFORMRiskIndex.vue'
import CopingMechanisms from './components/CopingMechanisms.vue'
import DoomedForever from './components/DoomedForever.vue'

export default {
  name: 'App',
  components: {
    HelloMorocco,
    RainfallPrecipitation,
    MeanTemperature,
    SheepCattleHerds,
    LandPloughedWheat,
    CopingMechanisms,
    INFORMRiskIndex,
    DoomedForever,
  },
  computed: {
    visibleSteps() {
      return this.steps.slice(1)
      /* return this.steps.slice(1, -1) */
    },
    visibleTitles() {
      return this.stepTitles.slice(1)
      /* return this.stepTitles.slice(1, -1) */
    },
  },
  data() {
    return {
      showMethodology: false,
      steps: [
        'HelloMorocco',
        'INFORMRiskIndex',
        'RainfallPrecipitation',
        'MeanTemperature',
        'SheepCattleHerds',
        'LandPloughedWheat',
        'CopingMechanisms',
        'DoomedForever',
      ],
      stepTitles: [
        'Hello Morocco',
        'Zooming Into Risk',
        'Raincheck Forever',
        'Future Heat',
        'Pasture Panic',
        'Wheat Drop',
        'Climate Survival Kit',
        'Disaster Time Machine',
      ],
      activeStep: 0,
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
    },
    getTabColor(index) {
      const component = this.steps[index + 1]; // +1 to match your visibleSteps slice
      switch (component) {
        case 'INFORMRiskIndex':
          return '#089d9d';
        case 'RainfallPrecipitation':
          return '#40E0D0';
        case 'MeanTemperature':
          return '#FF0000';
        case 'SheepCattleHerds':
          return '#d4bb97';
        case 'LandPloughedWheat':
          return '#B28F04';
        case 'CopingMechanisms':
          return '#85B738';
        case 'DoomedForever':
          return '#820747';
        default:
          return '#000'; // default
      }
    },
    formatStepName(step) {
      return step.replace(/([A-Z])/g, ' $1').trim()
    },
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
#scrollyteller {
  scroll-snap-type: y mandatory;
  overflow-y: scroll;
  height: 100vh;
}


body,
html {
  height: 100%;
  margin: 0;
  overflow: hidden;
}


.charts-wrapper {
  scroll-snap-align: start;
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
  text-transform: uppercase;
  font-size: 60px;
  font-weight: 300;
  margin: 0;
}

h5 {
  text-align: left;
  margin: 0;
  font-size: 28px;
  font-weight: 300;
}

h6 {
  padding-top: 1rem;
  text-align: left;
  font-weight: 300;
  font-size: 24px;
  margin: 0;
}

p {
  font-size: 16px;
  margin: 0;
  text-align: left;
  font-weight: 400;
}

text {
  font-family: "Parkinsans", sans-serif;
  margin: 0;
}

.left-tab-container {
  position: fixed;
  top: 50%;
  left: 0;
  transform: translateY(-50%);
  display: flex;
  flex-direction: column;
  gap: 10px;
  z-index: 10;
  padding-left: 10px;
}

.tab {
  writing-mode: vertical-rl;
  text-orientation: mixed;
  transform: rotate(180deg);
  padding: 10px 5px;
  cursor: pointer;
  font-size: 0.9em;
  color: #2c3e50;
  border-left: 2px solid transparent;
  transition: all 0.2s ease;
}

.tab.active {
  color: #000;
  border-left: 2px solid #000;
  font-weight: bold;
}

.right-tab-container {
  position: fixed;
  top: 50%;
  right: 0;
  transform: translateY(-50%);
  display: flex;
  flex-direction: column;
  gap: 10px;
  z-index: 10;
  padding-right: 10px;
}

.right-tab-container .tab {
  writing-mode: vertical-rl;
  text-orientation: mixed;
  transform: rotate(0deg);
  padding: 10px 5px;
  cursor: pointer;
  font-size: 0.9em;
  color: #2c3e50;
  border-left: 2px solid transparent;
  transition: all 0.2s ease;
  text-decoration: none;
}

.right-tab-container .tab.active {
  color: #000;
  border-left: 2px solid #000;
  font-weight: bold;
}


@keyframes slideInFromRight {
  from {
    transform: translateX(100%);
    opacity: 0;
  }

  to {
    transform: translateX(0%);
    opacity: 1;
  }
}

.methodology-panel {
  position: fixed;
  right: 70px;
  top: 25%;
  max-width: 1000px;
  background-color: white;
  padding: 13px 20px 17px 20px;
  z-index: 20;
  border: 1px solid #ccc;
  box-shadow: 0 0 10px rgba(0, 0, 0, 0.1);
  overflow-y: auto;
  max-height: 80vh;
  z-index: 1000;

  animation: slideInFromRight 0.5s ease-out forwards;
}
</style>
