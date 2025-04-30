<template>
  <div class="sheep-cattle-herds" ref="herdArea">
    <!-- Background canvas -->
    <canvas class="background-canvas" ref="noiseCanvas"></canvas>

    <!-- White padded boundary -->
    <div class="herd-padding-wrapper">
      <div class="herd-text" ref="herdText" :class="{ 'visible': textStage !== 'initial' }">
        <h3 v-if="textStage === 'reducing'">2014: The Grazeful Era</h3>
        <h3 v-else-if="textStage === 'complete'">2025: Desperately Seeking Grass</h3>
        <p v-if="textStage === 'reducing'">
          Imagine a herd of sheep and cattle, grazing peacefully in the fields of Morocco.
        </p>
        <div v-else-if="textStage === 'complete'">
        <p>
          According to official figures, Morocco's cattle and sheep herds have decreased
          by 38% in 2025 since the last census nine years ago, in 2014, due to consecutive
          droughts.</p>
          <p>
          And with Morocco in the sixth year of the drought, the Moroccan government has even
          asked its citizens not to slaughter sheep for the upcoming holiday Eid al-Adha.
        </p>
      </div>
      </div>

      <!-- Visual placement zone -->
      <div class="herd-visual-area" ref="herdVisual">
        <img v-for="(animal) in displayedHerd" :key="animal.id" :src="require(`@/assets/herds/${animal.src}`)"
          class="herd-image" :style="animal.style" />
      </div>
      <div class="background-overlay" :style="{ backgroundColor: backgroundTransitionColor }" />
      <!-- Text display instead of SVG -->
      <div class="countdown-container">
        <p class="countdown-text">{{ finalPercentage }}%</p>
      </div>

    </div>
  </div>
</template>

<script>
export default {
  data() {
    return {
      herd: [],
      displayedHerd: [],
      animalTypes: [
        "black-speckle-cow.png",
        "brown-cow.png",
        "brown-ram.png",
        "brown-sheep.png",
        "brown-speckle-cow.png",
        "gray-cow.png",
        "white-ram-2.png",
        "white-sheep-2.png",
      ],
      observer: null,
      hasReduced: false,
      textStage: 'initial', // 'initial', 'reducing', 'complete'
      finalPercentage: 100, // Add this property to control the percentage
    };
  },

  mounted() {
    this.generateHerd();
    this.setupIntersectionObserver();
    this.$nextTick(() => {
      this.drawBackgroundNoise();
    });
  },

  watch: {
    textStage() {
      this.$nextTick(() => {
        this.drawBackgroundNoise();
      });
    }
  },

  beforeUnmount() {
    if (this.observer) {
      this.observer.disconnect();
    }
  },
  computed: {
    backgroundTransitionColor() {
      // Interpolate between green and brown
      const pct = (100 - this.finalPercentage) / (100 - 62);
      const r = Math.round(217 + pct * (245 - 217)); // d9 -> f5
      const g = Math.round(240 + pct * (240 - 240)); // f0 -> f0
      const b = Math.round(217 + pct * (225 - 217)); // d9 -> e1
      return `rgb(${r},${g},${b})`;
    }
  },
  methods: {
    generateHerd() {
      let idCounter = 0;
      this.herd = [];

      // 4 types get 13, 4 types get 12
      const animalCounts = [13, 13, 13, 13, 12, 12, 12, 12];

      this.animalTypes.forEach((type, i) => {
        for (let j = 0; j < animalCounts[i]; j++) {
          this.herd.push({ id: idCounter++, src: type });
        }
      });

      // Delay placement to ensure layout is ready
      this.$nextTick(() => {
        setTimeout(() => {
          this.placeAnimals();
        }, 50);
      });
    },
    placeAnimals() {
      const visualArea = this.$refs.herdVisual;
      const textBox = this.$refs.herdText;
      const areaRect = visualArea.getBoundingClientRect();
      const textRect = textBox.getBoundingClientRect();

      const maxWidth = areaRect.width;
      const maxHeight = areaRect.height;

      const animalSize = 100; // size of .herd-image (width/height)
      const padding = 5;
      const cellSize = animalSize + padding;

      const textLeft = textRect.left - areaRect.left;
      const textTop = textRect.top - areaRect.top;
      const textBoxBounds = {
        left: textLeft - 10,
        top: textTop - 10,
        right: textLeft + textRect.width + 10,
        bottom: textTop + textRect.height + 10,
      };

      const cols = Math.floor(maxWidth / cellSize);
      const rows = Math.floor(maxHeight / cellSize);
      const grid = [];

      for (let y = 0; y < rows; y++) {
        for (let x = 0; x < cols; x++) {
          const posX = x * cellSize + cellSize / 2;
          const posY = y * cellSize + cellSize / 2;

          // Check if inside text bounds
          const overlapsText =
            posX + animalSize / 2 > textBoxBounds.left &&
            posX - animalSize / 2 < textBoxBounds.right &&
            posY + animalSize / 2 > textBoxBounds.top &&
            posY - animalSize / 2 < textBoxBounds.bottom;

          if (!overlapsText) {
            grid.push({ x: posX, y: posY });
          }
        }
      }

      // Shuffle grid positions
      for (let i = grid.length - 1; i > 0; i--) {
        const j = Math.floor(Math.random() * (i + 1));
        [grid[i], grid[j]] = [grid[j], grid[i]];
      }

      const positioned = this.herd.map((animal, i) => {
        const pos = grid[i];
        return {
          ...animal,
          style: {
            position: "absolute",
            left: `${pos.x - animalSize / 15}px`,
            top: `${pos.y - animalSize / 15}px`,
          },
        };
      });

      this.displayedHerd = positioned;
    },

    reduceHerd() {
      const totalToRemove = Math.floor(this.herd.length * 0.38); // 38% of the original herd
      let remainingToRemove = totalToRemove;

      const interval = setInterval(() => {
        if (remainingToRemove <= 0 || this.displayedHerd.length <= 0) {
          clearInterval(interval);

          // When reduction is done, move to the final stage
          this.textStage = 'complete';
          return;
        }

        // Remove one animal at a time
        this.displayedHerd.splice(Math.floor(Math.random() * this.displayedHerd.length), 1);
        remainingToRemove--;

        // Update the percentage accordingly
        const remainingPercent = Math.round((this.displayedHerd.length / this.herd.length) * 100);
        this.finalPercentage = Math.max(remainingPercent, 62); // Don’t go below 62%
      }, 100);
    },

    startCountdown() {
      const interval = setInterval(() => {
        if (this.finalPercentage > 62) {
          this.finalPercentage -= 1; // Decrease the percentage by 1 each interval
        } else {
          this.finalPercentage = 62; // Ensure it stays at 62% when reached
          clearInterval(interval); // Stop the countdown when it reaches 62%
        }
      }, 50); // Update every 50ms (for a smoother animation)
    },

    drawBackgroundNoise() {
      const canvas = this.$refs.noiseCanvas;
      const ctx = canvas.getContext('2d');
      const dpr = window.devicePixelRatio || 1;

      const width = this.$refs.herdArea.offsetWidth;
      const height = this.$refs.herdArea.offsetHeight;

      canvas.width = width * dpr;
      canvas.height = height * dpr;
      canvas.style.width = `${width}px`;
      canvas.style.height = `${height}px`;
      ctx.scale(dpr, dpr);

      // 🟢 or 🟤: Determine background and blade color based on stage
      const stage = this.textStage;
      console.log("Drawing background for stage:", stage); // Debugging

      const isReducing = stage === 'reducing';

      // 🌱 Green for initial, dry brown for reducing
      canvas.style.backgroundColor = isReducing ? '#d9f0d9' : '#f5f0e1';

      const bladeCount = 10000;
      for (let i = 0; i < bladeCount; i++) {
        const x = Math.random() * width;
        const y = Math.random() * height;

        const length = 8 + Math.random() * 12;
        const angle = (-Math.PI / 2) + (Math.random() - 0.5) * 0.6;
        const xEnd = x + length * Math.cos(angle);
        const yEnd = y + length * Math.sin(angle);

        ctx.beginPath();
        ctx.moveTo(x, y);
        ctx.lineTo(xEnd, yEnd);

        ctx.strokeStyle = isReducing
          ? `rgba(70, 120, 70, ${0.02 + Math.random() * 0.4})` // green-ish
          : `rgba(139, 69, 19, ${0.04 + Math.random() * 0.3})`; // brown-ish

        ctx.lineWidth = 1;
        ctx.stroke();
      }
    },

    resetAndAnimate() {
  // Reset all reactive states
  this.hasReduced = false;
  this.textStage = 'initial';
  this.finalPercentage = 100;
  this.displayedHerd = [];

  this.generateHerd(); // Regenerate the herd and place animals
  this.$nextTick(() => {
    this.drawBackgroundNoise();

    // Small delay to allow "initial" visuals before starting reduction
    setTimeout(() => {
      this.textStage = 'reducing';
      this.reduceHerd();
    }, 1000);
  });
},
setupIntersectionObserver() {
  this.observer = new IntersectionObserver(
    (entries) => {
      entries.forEach((entry) => {
        if (entry.isIntersecting) {
          this.resetAndAnimate();

          if (!this.hasReduced) {
            this.hasReduced = true;
            this.textStage = 'reducing'; // 👈 Set middle message when herd starts reducing
          }
        }
      });
    },
    {
      threshold: 0.1, // Trigger when 10% of the component is visible
    }
  );

  this.observer.observe(this.$refs.herdArea);
},
  },
};
</script>

<style scoped>
.sheep-cattle-herds {
  position: relative;
  width: calc(100vw - 11rem);
  height: calc(100vh - 10rem - 20px);
  overflow: hidden;
}

.background-canvas {
  position: absolute;
  top: 0;
  left: 0;
  z-index: 0;
  /* Background canvas stays at the bottom */
  width: 100%;
  height: 100%;
}

.herd-padding-wrapper {
  position: relative;
  width: 100%;
  height: 100%;
  box-sizing: border-box;
  z-index: 2;
  /* Place the padded area above the background */
}

.herd-text {
  width: 40%;
  margin: 0 auto;
  opacity: 0;
  visibility: hidden;
  transition: opacity 1s ease, visibility 1s ease;
  position: absolute;
  top: 0;
  left: 0;
  padding-top: 5px;
  padding-left: 10px;
  z-index: 3;
  /* Place the text above everything else */
  text-align: left;
}

.herd-text.visible {
  opacity: 1;
  visibility: visible;
}

.svg-container {
  transition: opacity 1s ease;
  position: absolute;
  top: 20%;
  /* Adjust the position relative to the text */
  left: 50%;
  /* Center horizontally */
  transform: translateX(-50%);
  /* Center horizontally */
  opacity: 1;
  /* Adjust opacity for visibility */
  z-index: 2;
  /* Ensure the SVG appears above the background, but below the herd images */
  width: 95%;
  /* Set width to desired value */
  opacity: 0.75;
}

.herd-visual-area {
  position: relative;
  width: 100%;
  height: 100%;
  overflow: hidden;
  z-index: 3;
  /* Ensure the herd images are above the SVG */
}

.herd-image {
  object-fit: contain;
  position: absolute;
  transition: transform 0.5s ease, opacity 0.5s ease;
}

.countdown-container {
  position: absolute;
  top: 10%;
  left: 50%;
  transform: translateX(-50%);
  font-size: 2rem;
  color: black;
  z-index: 2;
  /* Ensures it is above everything */
}

.countdown-text {
  opacity: 0.10;
  transition: none;
  text-align: center;
  font-weight: 700;
  font-size: 30ch;
  white-space: nowrap;
  overflow: hidden;
  text-overflow: ellipsis;
}

.background-overlay {
  position: absolute;
  top: 0;
  left: 0;
  width: 100%;
  height: 100%;
  opacity: 0.5;
  z-index: 1;
  /* Above canvas, below everything else */
  transition: background-color 1s linear;
  pointer-events: none;
}

@media screen and (min-width: 1901px) {
  .herd-image {
    width: 100px;
    height: 100px;
  }

}

@media screen and (max-width: 1900px) and (min-width: 1001px) {
  .herd-image {
    width: 70px;
    height: 70px;
  }
  .countdown-text {
  font-size: 20ch;
}
}

@media screen and (max-width: 1000px) {
  .herd-image {
    width: 50px;
    height: 50px;
  }
  .countdown-text {
  font-size: 10ch;
}
}

</style>
