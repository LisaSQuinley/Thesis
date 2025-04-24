<template>
  <div ref="container" class="land-ploughed-wheat" :style="backgroundStyle">
    <!-- Base Message Block -->
    <div class="phase-pair phase-pair-base visible" :class="{ 'fade-in': firstPhaseStarted }">
      <div class="text-block">
        <h3 class="base-title">The Feast Before the Flame</h3>
        <h4 class="subtitle subtitle-base">2014–2019: When rains were kind and wheat flowed freely</h4>
        <div class="phase-message phase-message-base">
          <p>Almost 3,000 thousand hectares prepped — over 6 million tons ready to grow!</p>
        </div>
      </div>
      <div class="wheat-grid">
        <img v-for="index in wheatBaseCount" :key="index" :src="require(`@/assets/wheat/wheat-base.svg`)" alt="Wheat"
          :style="{ width: imageSize + 'px', height: 'auto' }" />
      </div>
    </div>

    <!-- Phase 1 Message Block -->
    <div class="phase-pair phase-pair-88" :class="{ 'fade-in': firstPhaseStarted }">
      <div class="text-block">
        <h3 class="title title-88">Scorched Rhythm</h3>
        <h4 class="subtitle subtitle-88">2019–2024: Drought tested our resolve, but the harvest held</h4>
        <div class="phase-message phase-message-88">
          <p>We’re cutting back a bit — still growing strong with over 4 million tons of wheat on the way!</p>
        </div>
      </div>
      <div class="wheat-grid">
        <img v-for="index in wheat69Count" :key="index" :src="require(`@/assets/wheat/wheat-69.svg`)" alt="Wheat-69"
          :style="{ width: imageSize + 'px', height: 'auto' }" />
      </div>
    </div>

    <!-- Phase 2 Message Block -->
    <div class="phase-pair phase-pair-74" :class="{ 'fade-in': secondPhaseStarted }">
      <div class="text-block">
        <h3 class="title title-74">Echoes of a Vanished Spring</h3>
        <h4 class="subtitle subtitle-74">2025: A sudden drop — the fields rest, the future waits</h4>
        <div class="phase-message phase-message-74">
          <p>Less land, fewer crops — but over 2 million tons still pushing through!</p>
        </div>
      </div>
      <div class="wheat-grid">
        <img v-for="index in wheat41Count" :key="index" :src="require(`@/assets/wheat/wheat-41.svg`)" alt="Wheat-41"
          :style="{ width: imageSize + 'px', height: 'auto' }" />
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
      imageSize: 80, // Size of the wheat images
      wheatBaseCount: 100, // Full count for base wheat
      wheat69Count: Math.floor(0.69 * 100), // 69% of wheat-base.svgs for phase-pair-88
      wheat41Count: Math.floor(0.41 * 100), // 41% of wheat-base.svgs for phase-pair-74
      backgroundStyle: {
        backgroundColor: '#d9f0d9', // Initial green background
        transition: 'background-color 1s ease' // Smooth transition for the color change
      }
    };
  },
  beforeUnmount() {
    window.removeEventListener('resize', this.updateImageSize);
  },
  mounted() {
    this.updateImageSize();
    window.addEventListener('resize', this.updateImageSize);
    // Log the number of SVGs in each phase
    this.logWheatCounts();

    // Initialize IntersectionObserver for triggering animation
    this.observeIntersection();
  },
  methods: {
    updateImageSize() {
  const width = window.innerWidth;
  if (width < 700) this.imageSize = 40;
  else if (width < 1000) this.imageSize = 60;
  else if (width < 1400) this.imageSize = 80;
  else this.imageSize = 100;
},
    observeIntersection() {
      const observer = new IntersectionObserver(
        (entries) => {
          if (entries[0].isIntersecting) {
            this.animateSections();
          }
        },
        {
          threshold: 0.5, // Trigger when 50% of the element is visible
        }
      );
      observer.observe(this.$refs.container);
    },
    animateSections() {
      // Delay of 5 seconds before changing background color
      setTimeout(() => {
        this.backgroundStyle.backgroundColor = '#f5f0e1'; // Change to the desired color

        // Start animation for the first phase after the background change
        setTimeout(() => {
          this.firstPhaseStarted = true;

          // Trigger second phase after 1.5 seconds
          setTimeout(() => {
            this.secondPhaseStarted = true;
          }, 3000);
        }, 1000); // Wait for 1 second before starting the animation
      }, 3000); // Wait for 5 seconds before starting the background color transition
    },
    logWheatCounts() {
      console.log(`Base Wheat Count: ${this.wheatBaseCount} wheat-base.svg images in phase-pair-base`);
      console.log(`Wheat-69 Count: ${this.wheat69Count} wheat-69.svg images in phase-pair-88`);
      console.log(`Wheat-41 Count: ${this.wheat41Count} wheat-41.svg images in phase-pair-74`);
    },
  },
};
</script>

<style scoped>
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

.text-block {
  padding: 5px 10px;
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
  /* Add z-index transition */
  display: flex;
  flex-direction: column;
  justify-content: flex-end;
  /* Ensures the content stays at the bottom */
  box-sizing: border-box;
}

.phase-pair-base:hover {
  z-index: 10;
  /* Temporarily bring to the front */
  background-color: #d2f8d2;
  /* Example background color on hover */
}

/* Hover effect to change z-index */
.phase-pair:hover {
  z-index: 10;
  /* Temporarily bring to the front */
}

.phase-pair-base {
  top: 0;
  height: 100%;
}

.phase-pair-88 {
  top: 12%;
  /* Adjust based on the percentage */
  background-color: #e2d3b4;
  height: 88%;
}

.phase-pair-74 {
  top: 26%;
  /* Adjust based on the percentage */
  background-color: #d4bb97;
  height: 74%;
}

.wheat-grid {
  display: flex;
  flex-wrap: wrap;
  justify-content: center;
  gap: 0.5rem;
  padding-bottom: 0.5rem;
  margin-top: auto;
}

img.hidden {
  opacity: 0;
  transition: opacity 0.3s;
}
</style>
