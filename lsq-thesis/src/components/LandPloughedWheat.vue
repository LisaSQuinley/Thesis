<template>
  <div ref="container" class="land-ploughed-wheat" :style="backgroundStyle">
    <!-- Base Message Block -->
    <div class="phase-pair phase-pair-base visible" :class="{ 'fade-in': firstPhaseStarted }">
      <div class="phase-content">
        <div class="text-block column-block">
          <h4 class="base-title">The Feast Before the Flame</h4>
          <h5 class="subtitle subtitle-base">Average wheat tonnage and hectares of land from 2014–2019</h5>
          <h6 class="subtitle subtitle-base">When rains were kind and wheat flowed freely.</h6>
          <div class="phase-message phase-message-base"></div>
        </div>
        <div class="text-block row-block">
          <h4 class="wheat-number">{{ shownWheatBase }}</h4>
          <p>ten thousand <br>tons of wheat</p>
          <h4 class="land-number">299</h4>
          <p>ten thousand <br>hectares</p>
        </div>
      </div>
      <div class="wheat-field">
        <div class="wheat-grid wheat-grid-base">
          <img v-for="index in wheatBaseArray.slice(0, shownWheatBase)" :key="'base-' + index"
            :src="require(`@/assets/wheat/wheat-base.svg`)" alt="Wheat"
            :style="{ width: imageSize + 'px', height: 'auto' }" />
        </div>
        <div class="graphic-title">
          <h3>Wheat Drop</h3>
        </div>
      </div>
    </div>

    <!-- Phase 1 Message Block -->
    <div class="phase-pair phase-pair-88" :class="{ 'fade-in': firstPhaseStarted }">
      <div class="phase-content">
        <div class="text-block column-block">
          <h4 class="title title-88">Scorched Rhythm</h4>
          <h5 class="subtitle subtitle-88">2019 through 2024 average wheat tonnage and hectares of land</h5>
          <h6 class="subtitle subtitle-88">Drought tested our resolve, but the harvest held.</h6>
          <div class="phase-message phase-message-88"></div>
        </div>
        <div class="text-block row-block">
          <h4 class="wheat-number">{{ shownWheat69 }}</h4>
          <p>ten thousand <br>tons of wheat</p>
          <h4 class="land-number">{{ landValue88 }}</h4>
          <p>ten thousand <br>hectares</p>
        </div>
      </div>
      <div class="wheat-field">
        <div class="wheat-grid wheat-grid-69">
          <img v-for="index in wheat69Array.slice(0, shownWheat69)" :key="'69-' + index"
            :src="require(`@/assets/wheat/wheat-69.svg`)" alt="Wheat-69"
            :style="{ width: imageSize + 'px', height: 'auto' }" />
        </div>
      </div>
      <div class="graphic-title">
        <h3>Wheat Drop</h3>
      </div>
    </div>

    <!-- Phase 2 Message Block -->
    <div class="phase-pair phase-pair-74" :class="{ 'fade-in': secondPhaseStarted }">
      <div class="phase-content">
        <div class="text-block column-block">
          <h4 class="title title-74">Echoes of a Vanished Spring</h4>
          <h5 class="subtitle subtitle-74">The wheat yield and land usage for 2025</h5>
          <h6 class="subtitle subtitle-74">A sudden drop — the fields rest, the future waits.</h6>
          <div class="phase-message phase-message-74"></div>
        </div>
        <div class="text-block row-block">
          <h4 class="wheat-number">{{ shownWheat41 }}</h4>
          <p>ten thousand <br>tons of wheat</p>
          <h4 class="land-number">{{ landValue74 }}</h4>
          <p>ten thousand <br>hectares</p>
        </div>
      </div>
      <div class="wheat-field">
        <div class="wheat-grid wheat-grid-41">
          <img v-for="index in wheat41Array.slice(0, shownWheat41)" :key="'41-' + index"
            :src="require(`@/assets/wheat/wheat-41.svg`)" alt="Wheat-41"
            :style="{ width: imageSize + 'px', height: 'auto' }" />
        </div>
      </div>
      <div class="graphic-title graphic-title-41">
        <h3>Wheat<br>Drop</h3>
      </div>
    </div>
  </div>
</template>


<script>
export default {
  data() {
    return {
      firstPhaseStarted: false,
      secondPhaseStarted: false,
      imageSize: 70,
      wheatBaseCount: 607,
      wheat69Count: 420,
      wheat41Count: 246,
      shownWheatBase: 0,  // << ADD THIS
      shownWheat69: 0,
      shownWheat41: 0,
      landValue88: 299, // Will count down to 262
      landValue74: 262,  // Will count down to 220
      backgroundStyle: {
        backgroundColor: '#d9f0d9',
        transition: 'background-color 1s ease'
      }
    };
  },

  computed: {
    wheatBaseArray() {
      return Array.from({ length: this.wheatBaseCount }, (_, i) => i + 1);
    },
    wheat69Array() {
      return Array.from({ length: this.wheat69Count }, (_, i) => i + 1);
    },
    wheat41Array() {
      return Array.from({ length: this.wheat41Count }, (_, i) => i + 1);
    }
  },


  beforeUnmount() {
    window.removeEventListener('resize', this.updateImageSize);
  },

  mounted() {
    this.updateImageSize();
    window.addEventListener('resize', this.updateImageSize);
    this.logWheatCounts();
    this.observeIntersection();
  },

  methods: {
    updateImageSize() {
      const containerWidth = window.innerWidth - 200;
      const containerHeight = window.innerHeight - 200;
      const totalWheats = this.wheatBaseCount;
      const gridCols = Math.ceil(Math.sqrt(totalWheats));
      const imageMaxWidth = containerWidth / gridCols;
      const imageMaxHeight = containerHeight / gridCols;
      this.imageSize = Math.floor(Math.min(imageMaxWidth, imageMaxHeight, 80));
    },

    observeIntersection() {
      const observer = new IntersectionObserver(
        (entries) => {
          if (entries[0].isIntersecting) {
            this.animateSections();
          }
          if (entries[0].isIntersecting) {
            this.restartAnimations();
          }
        },
        { threshold: 0.5 }
      );
      observer.observe(this.$refs.container);
    },
    animateSections() {
      setTimeout(() => {
        this.backgroundStyle.backgroundColor = '#f5f0e1';
        setTimeout(() => {
          console.time('Wheat Base Growth');
          this.incrementShownWheatBase(() => {
            console.timeEnd('Wheat Base Growth');

            setTimeout(() => {
              this.firstPhaseStarted = true;
              this.countDownLand88(); // ← Trigger countdown to 262

              this.incrementShownWheat69(() => {
                console.timeEnd('Wheat 69 Growth');

                setTimeout(() => {
                  this.secondPhaseStarted = true;
                  this.countDownLand74(); // ← Trigger countdown to 220

                  this.incrementShownWheat41(() => {
                    console.timeEnd('Wheat 41 Growth');
                  });
                }, 2000);
              });
            }, 2000);
          });
        }, 1000);
      }, 1000);
    },

    restartAnimations() {
      // Reset state
      this.shownWheatBase = 0;
      this.shownWheat69 = 0;
      this.shownWheat41 = 0;
      this.firstPhaseStarted = false;
      this.secondPhaseStarted = false;

      // Optional: reset background color immediately
      this.backgroundStyle.backgroundColor = '#d9f0d9';

      // Restart animation chain
      this.animateSections();
    },

    // Increment functions

    incrementShownWheatBase(callback) {
      const interval = setInterval(() => {
        if (this.shownWheatBase < this.wheatBaseCount) {
          this.shownWheatBase++;
        } else {
          clearInterval(interval);
          if (callback) callback();
        }
      }, 5);
    },

    incrementShownWheat69(callback) {
      const interval = setInterval(() => {
        if (this.shownWheat69 < this.wheat69Count) {
          this.shownWheat69++;
        } else {
          clearInterval(interval);
          if (callback) callback();
        }
      }, 5);
    },

    incrementShownWheat41(callback) {
      const interval = setInterval(() => {
        if (this.shownWheat41 < this.wheat41Count) {
          this.shownWheat41++;
        } else {
          clearInterval(interval);
          if (callback) callback();
        }
      }, 5);
    },

    countDownLand88() {
      const target = 262;
      const interval = setInterval(() => {
        if (this.landValue88 > target) {
          this.landValue88--;
        } else {
          clearInterval(interval);
        }
      }, 20);
    },

    countDownLand74() {
      const target = 220;
      const interval = setInterval(() => {
        if (this.landValue74 > target) {
          this.landValue74--;
        } else {
          clearInterval(interval);
        }
      }, 20);
    },


    logWheatCounts() {
      console.log(`Base Wheat Count: ${this.wheatBaseCount} wheat-base.svg images in phase-pair-base`);
      console.log(`Wheat-69 Count: ${this.wheat69Count} wheat-69.svg images in phase-pair-88`);
      console.log(`Wheat-41 Count: ${this.wheat41Count} wheat-41.svg images in phase-pair-74`);
    }
  }
};
</script>


<style scoped>
.graphic-title {
  position: absolute;
  bottom: 1rem;
  right: 6rem;
  font-size: 5em;
  color: #B28F042f;
  text-transform: uppercase;
  line-height: 0.9;
  transform: rotate(90deg);
  transform-origin: bottom right;
  z-index: 20;
  pointer-events: none;
  text-align: right;
}

h4 {
  text-align: left;
  text-transform: uppercase;
  font-size: 60px;
  margin: 0;
  font-weight: 300;
  line-height: 1;
}

h5 {
  text-align: left;
  font-size: 28px;
  font-weight: 300;
  margin: 0;
  line-height: 1.25;
}

h6 {
  padding-top: 0.25rem;
  text-align: left;
  font-weight: 300;
  font-size: 24px;
  margin: 0;
}

.graphic-title h3 {
  font-weight: 800;
}

.phase-pair-74 .graphic-title-41 {
  position: absolute;
  bottom: 1rem;
  right: 11rem;
  color: #f5f0e14f;
}

.graphic-title-41 {
  margin-right: 5px;
}

.land-ploughed-wheat {
  position: relative;
  width: calc(100vw - 12rem);
  height: calc(100vh - 13rem);
  overflow: hidden;
  display: flex;
  flex-direction: column;
  padding: 0.5rem;
  box-sizing: border-box;
}

.phase-pair.visible {
  opacity: 1;
  transform: translateY(0);
}

/* Add fade-in animation */
.fade-in {
  opacity: 1 !important;
  transform: translateY(0) !important;
}

.phase-pair {
  text-align: left;
  position: absolute;
  left: 0;
  width: 100%;
  opacity: 0;
  transform: translateY(-50px);
  transition: opacity 1s ease-out, transform 1s ease-out, z-index 0.3s ease;
  display: flex;
  flex-direction: column;
  justify-content: space-between;
  box-sizing: border-box;
  height: 100%;
}

.text-block {
  padding: 5px 10px;
}

.phase-content {
  display: flex;
  flex-direction: row;
  padding-top: 5px;
}

/* first text-block: flex column */
.column-block {
  margin-left: 5px;
  width: 50%;
  display: flex;
  flex-direction: column;
}

/* second text-block: flex row */
.row-block {
  width: 50%;
  display: flex;
  flex-direction: row;
  flex-wrap: wrap;
  justify-content: flex-end;
  text-align: right;
  font-size: 22px;
  font-weight: 400;
  margin-right: 5px;
}

.row-block p {
  padding-left: 10px;
}

.phase-pair-base .row-block p,
.phase-pair-base .row-block h4 {
  color: #A67A05;
  /* color: #FF8000; */
}

.phase-pair-88 .row-block p,
.phase-pair-88 .row-block h4 {
  color: #D09910;
  /* color: #FF8000; */
}

.phase-pair-74 .row-block p,
.phase-pair-74 .row-block h4 {
  color: white;
  /* color: #820747; */
}

.wheat-number,
.land-number {
  line-height: 1em;
  margin: 0;
  margin-left: 10px;
}

.phase-pair-base:hover {
  z-index: 10;
  background-color: #f5f0e1;
}

.phase-pair:hover {
  z-index: 10;
}

.phase-pair-base {
  top: 0;
  height: 100%;
}

.phase-pair-88 {
  top: 12%;
  background-color: #e2d3b4;
  height: 88%;
}

.phase-pair-74 {
  top: 26%;
  background-color: #d4bb97;
  height: 74%;
}

.wheat-field {
  padding-left: 10px;
  display: flex;
  flex-direction: column;
  justify-content: flex-end;
  align-items: center;
  padding-bottom: 1rem;
  position: relative;
}

.wheat-grid {
  position: relative;
  display: flex;
  flex-wrap: wrap-reverse;
  align-items: flex-end;
  align-content: flex-end;
  height: 100%;
  justify-content: space-evenly;
  max-width: 100%;
}

.wheat-grid img {
  position: relative;
  z-index: 1;
  padding-top: 5px;
}

img.hidden {
  opacity: 0;
  transition: opacity 0.3s;
}

.wheat-grid-base,
.wheat-grid-69 {
  padding-right: 6rem;
}

.wheat-grid-41 {
  padding-right: 11.5rem;
}
</style>
