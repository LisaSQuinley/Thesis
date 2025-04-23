<template>
  <div ref="container" class="land-ploughed-wheat" :style="backgroundStyle">
    <div class="land-bar" :style="{ backgroundColor: landBarColor, height: landBarHeight + '%' }">
    </div>

    <!-- Phase 1 Message Block -->
    <div class="phase-pair phase-pair-88" :class="{ visible: firstPhaseStarted }">
      <h3 class="title title-88">First Shift: Trimming the Fields</h3>
      <h4 class="subtitle subtitle-88">2019–2024: Drought hits, but the grind doesn’t stop</h4>
      <div class="phase-message phase-message-88">
        <p>We’re cutting back a bit — still growing strong with over 4 million tons of wheat on the way!</p>
      </div>
    </div>

    <!-- Phase 2 Message Block -->
    <div class="phase-pair phase-pair-74" :class="{ visible: secondPhaseStarted }">
      <h3 class="title title-74">Second Shift: Fields Take a Breather</h3>
      <h4 class="subtitle subtitle-74">2025: This year's harvest — a tough one, but not the end</h4>
      <div class="phase-message phase-message-74">
        <p>Less land, fewer crops — but over 2 million tons still pushing through!</p>
      </div>
    </div>

    <!-- Base Message Block -->
    <div class="phase-pair phase-pair-base visible">
      <h3 class="base-title">Wheat World: The Land of Plenty</h3>
      <h4 class="subtitle subtitle-base">2014–2019: Back when the fields were full and rains came easy</h4>
      <div class="phase-message phase-message-base">
        <p>Almost 3,000 thousand hectares prepped — over 6 million tons ready to grow!</p>
      </div>
    </div>
    
    <div class="wheat-grid">
      <!-- Updated image source to wheat.svg -->
      <img v-for="(wheat, index) in wheatImages" :key="index" src="@/assets/wheat.svg" alt="Wheat"
        :class="{ hidden: isHidden(index) }" />
    </div>
  </div>
</template>




<script>
export default {
  data() {
    return {
      inView: false,
      secondPhase: false, // Second phase flag
      firstPhaseStarted: false,
      secondPhaseStarted: false,
      landBarHeight: 100, // default full height
      landBarColor: '#d9f0d9', // Initial green color
      wheatImages: [], // Array of wheat images
      firstReductionPercent: 0.69, // First reduction to 69%
      secondReductionPercent: 0.41, // Second reduction to 41%
      firstReductionTime: 4500, // Time for first phase
      secondReductionTime: 6500, // Time for second phase
      message: 'Preparing the land for ploughing...', // Initial message
      phaseTitle: 'Amount of Land Ploughed for Wheat Graphic', // Initial title
    };
  },
  computed: {
    // Dynamically calculate the number of visible wheat images based on the current phase
    visibleWheatCount() {
      if (this.secondPhase) {
        return Math.floor(this.wheatImages.length * this.secondReductionPercent); // Reduce to 41%
      } else if (this.inView) {
        return Math.floor(this.wheatImages.length * this.firstReductionPercent); // Reduce to 69%
      }
      return this.wheatImages.length; // Full count before animations start
    },

    // Computed property to set the background color based on the height of the land-bar
    backgroundStyle() {
      const landBarHeight = this.secondPhase ? 74 : this.inView ? 88 : 100; // Default 100% height before animations

      // Define the dark, medium, and light brown colors
      const lightBrown = '#f5f0e1'; // Light brown color
      const mediumBrown = '#e2d3b4'; // Lightened medium brown
      const darkBrown = '#d4bb97';   // Lightened dark brown

      // Create the gradient based on the current height
      const backgroundGradient = `linear-gradient(
        to bottom,
        ${lightBrown} ${100 - landBarHeight}%, 
        ${mediumBrown} ${100 - landBarHeight}%,
        ${mediumBrown} 88%, 
        ${darkBrown} 88%,
        ${darkBrown} 74%, 
        ${lightBrown} 74%
      )`;

      return {
        background: backgroundGradient, // Apply the dynamic gradient
      };
    },
  },
  methods: {
    // Hide images based on their index
    isHidden(index) {
      return this.inView && index >= this.visibleWheatCount;
    },
    logWheatCounts() {
      console.log(`Total wheat images generated: ${this.wheatImages.length}`);
      console.log(`Wheat images after first reduction: ${Math.floor(this.wheatImages.length * this.firstReductionPercent)}`);
      console.log(`Wheat images after second reduction: ${Math.floor(this.wheatImages.length * this.secondReductionPercent)}`);
    },
    // Update total wheat images based on available space
    updateWheatImages() {
      const containerWidth = this.$refs.container.offsetWidth;
      const containerHeight = this.$refs.container.offsetHeight;

      // Calculate how many images can fit in the space
      const imagesPerRow = Math.floor(containerWidth / 60); // Assuming each image is 60px wide
      const rows = Math.floor(containerHeight / 60); // Assuming each image is 60px tall

      // Generate the wheat images dynamically based on the space
      this.wheatImages = Array(imagesPerRow * rows).fill(null);
    },
    // Update title and message based on the current phase
    updateTitleAndMessage() {
      if (this.secondPhase) {
        this.phaseTitle = 'Second Phase: Continuing the Land Ploughing';
        this.message = 'Ploughing in progress, we are almost halfway through!';
      } else if (this.inView) {
        this.phaseTitle = 'First Phase: Starting the Land Ploughing';
        this.message = 'Ploughing the land, first phase underway...';
      } else {
        this.phaseTitle = 'Amount of Land Ploughed for Wheat Graphic';
        this.message = 'Preparing to plough the land...';
      }
    },
  },
  mounted() {
    this.updateWheatImages();
    window.addEventListener('resize', this.updateWheatImages);

    const observer = new IntersectionObserver(
      ([entry]) => {
        if (entry.isIntersecting) {
          // Delay before starting the first phase
          setTimeout(() => {
            this.inView = true;
            this.firstPhaseStarted = true;
            this.updateTitleAndMessage();
            this.logWheatCounts();

            this.landBarColor = '#f5f0e1'; // Light brown
            this.landBarHeight = 88;       // Animate to 88%

            // Midway color change
            setTimeout(() => {
              this.landBarColor = '#e2d3b4'; // Medium brown
            }, this.firstReductionTime / 2);

            // Phase 2
            setTimeout(() => {
              this.secondPhase = true;
              this.secondPhaseStarted = true;
              this.updateTitleAndMessage();
              this.logWheatCounts();
              this.landBarColor = '#d4bb97'; // Dark brown
              this.landBarHeight = 74;       // Animate to 74%
            }, this.secondReductionTime);
          }, 2500);

        }
      },
      { threshold: 0.5 }
    );
    observer.observe(this.$refs.container);
  }
  ,

  unmounted() {
    // Clean up event listener
    window.removeEventListener('resize', this.updateWheatImages);
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

.land-bar {
  position: absolute;
  top: 0;
  right: 0;
  width: 100%;
  height: 100%;
  transition: height 1s ease-in-out, background-color 1s ease-in-out;
  z-index: 0;
}

.land-bar.animate {
  height: 88%;
}

.land-bar.secondPhase {
  height: 74%;
}

.wheat-grid {
  flex: 1;
  display: grid;
  grid-template-columns: repeat(auto-fit, minmax(50px, 1fr));
  grid-auto-rows: minmax(0, 1fr);
  gap: 0.25rem;
  width: 100%;
  height: 100%;
  box-sizing: border-box;
  z-index: 1;
  padding-bottom: 1rem;
}

.wheat-grid img {
  width: 100%;
  height: 100%;
  object-fit: contain;
  transition: opacity 0.5s ease-in;
  max-width: 100%;
  max-height: 100%;
}

.wheat-grid img.hidden {
  opacity: 0;
}

/* --- Phase Pair Container --- */
.phase-pair {
  position: absolute;
  right: 1rem;
  z-index: 2;
  margin: 0;
  display: flex;
  flex-direction: column;
  align-items: flex-end;
  transition: opacity 0.5s ease-in-out;
}

.phase-pair-base {
  opacity: 1;
  transition: opacity 1s ease-in;
  bottom: 0;
}

.phase-pair-88 {
  top: 88%;
  /* Adjust the positioning so that it's within the 88% block */
  transform: translateY(-100%);
}

.phase-pair-74 {
  top: 74%;
  /* Adjust the positioning so that it's within the 74% block */
  transform: translateY(-100%);
}

/* --- Titles --- */
.title,
.base-title {
  margin: 0;
  font-weight: bold;
}

/* --- Messages --- */
.phase-message {
  margin: 0;
  border-radius: 4px;
}

.phase-message-88,
.phase-message-74,
.phase-message-base {
  transform: translateY(0%);
  padding-bottom: 12px;
}

.phase-message p {
  margin: 0;
}

.phase-pair {
  opacity: 0;
  transition: opacity 1s ease-in;
  pointer-events: none;
  /* Prevents hovering on hidden ones */
}

.phase-pair.visible {
  opacity: 1;
  pointer-events: auto;
}
</style>
