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
        <p v-else-if="textStage === 'complete'">
          According to official figures, Morocco's cattle and sheep herds have decreased
          by 38% in 2025 since the last census nine years ago, in 2014, due to consecutive
          droughts.<br />
          And with Morocco in the sixth year of the drought, the Moroccan government has even
          asked its citizens not to slaughter sheep for the upcoming holiday Eid al-Adha.
        </p>
      </div>

      <!-- Visual placement zone -->
      <div class="herd-visual-area" ref="herdVisual">
        <img
          v-for="(animal) in displayedHerd"
          :key="animal.id"
          :src="require(`@/assets/herds/${animal.src}`)"
          class="herd-image"
          :style="animal.style"
        />
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

      // Adjust textRect relative to visualArea
      const textLeft = textRect.left - areaRect.left;
      const textTop = textRect.top - areaRect.top;

      // Add padding around the text box boundary
      const textPaddingX = 60;
      const textPaddingY = 60;

      const textBoxBounds = {
        left: textLeft - textPaddingX,
        top: textTop - textPaddingY,
        right: textLeft + textRect.width + textPaddingX,
        bottom: textTop + textRect.height + textPaddingY,
      };

      const placedAnimals = [];
      const animalSize = 75;
      const padding = 6;
      const radius = animalSize / 2 + padding;

      const canPlace = (x, y) => {
        for (let a of placedAnimals) {
          const dx = a.x - x;
          const dy = a.y - y;
          const dist = Math.sqrt(dx * dx + dy * dy);
          if (dist < radius * 2) return false;
        }

        const animalLeft = x - radius;
        const animalRight = x + radius;
        const animalTop = y - radius;
        const animalBottom = y + radius;

        const overlapsText =
          animalRight > textBoxBounds.left &&
          animalLeft < textBoxBounds.right &&
          animalBottom > textBoxBounds.top &&
          animalTop < textBoxBounds.bottom;

        return (
          x > radius &&
          y > radius &&
          x < maxWidth - radius &&
          y < maxHeight - radius &&
          !overlapsText
        );
      };

      const positioned = [];

      for (let animal of this.herd) {
        let tries = 0;
        let x, y;
        do {
          x = radius + Math.random() * (maxWidth - 2 * radius);
          y = radius + Math.random() * (maxHeight - 2 * radius);
          tries++;
          if (tries > 300) break;
        } while (!canPlace(x, y));

        placedAnimals.push({ x, y });
        positioned.push({
          ...animal,
          style: {
            position: "absolute",
            left: `${x - animalSize / 2}px`,
            top: `${y - animalSize / 2}px`,
          },
        });
      }

      this.displayedHerd = positioned;
    }
    ,

    reduceHerd() {
      const toRemovePerType = [5, 5, 5, 5, 5, 5, 4, 4];
      let newHerd = [...this.displayedHerd];

      this.animalTypes.forEach((type, i) => {
        let removed = 0;
        newHerd = newHerd.filter((animal) => {
          if (animal.src === type && removed < toRemovePerType[i]) {
            removed++;
            return false;
          }
          return true;
        });
      });

      this.displayedHerd = newHerd;
      console.log("Reduced herd count:", this.displayedHerd.length);

      // Delay text appearance by 2 seconds after herd reduction
      setTimeout(() => {
        this.showText = true;
      }, 2500);
      setTimeout(() => {
        this.textStage = 'complete'; // 👈 Show the final message
      }, 0);

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

      const bladeCount = 7000;
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

    setupIntersectionObserver() {
      this.observer = new IntersectionObserver(
        (entries) => {
          entries.forEach((entry) => {
            if (entry.isIntersecting && !this.hasReduced) {
              this.hasReduced = true;
              this.textStage = 'reducing'; // 👈 Set middle message when herd starts reducing
              setTimeout(() => {
                this.reduceHerd(); // herd will reduce and then trigger final text
              }, 2500);
            }
          });
        },
        {
          threshold: 0.1,
        }
      );

      this.observer.observe(this.$refs.herdArea);
    },

  },
};
</script>

<style scoped>
h3 {
  margin-bottom: 1rem;
}
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
  width: 100%;
  height: 100%;
}

.herd-padding-wrapper {
  position: relative;
  width: 100%;
  height: 100%;
  padding: 2rem 2rem 2rem 2rem;
  box-sizing: border-box;
  z-index: 1;
}

.herd-text {
  width: 40%;
  margin: 0 auto;
  opacity: 0;
  visibility: hidden;
  transition: opacity 1s ease, visibility 1s ease;
  position: absolute;
  top: 50%;
  left: 50%;
  transform: translate(-50%, -50%);
  z-index: 2;
  text-align: center;
}

.herd-text.visible {
  opacity: 1;
  visibility: visible;
}

.herd-visual-area {
  position: relative;
  width: 100%;
  height: 100%;
  overflow: hidden;
}

.herd-image {
  width: 100px;
  height: 100px;
  object-fit: contain;
  position: absolute;
  transition: transform 0.5s ease, opacity 0.5s ease;
}

</style>
