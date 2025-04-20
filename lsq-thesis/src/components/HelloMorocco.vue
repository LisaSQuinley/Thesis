<template>
  <div class="hello">
    <h1 class="droughts">Droughts</h1>
    <div class="floods-fires-line">
      <h1 class="floods">Floods</h1>
      <h2 class="rotated">and</h2>
      <h1 class="fires">Fires</h1>
    </div>
    <h2 class="subtitle">A Case Study of Morocco and Climate Change</h2>
    <h3 class="byline">Lisa Sakai Quinley</h3>
  </div>
</template>




<style scoped>
html, body {
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
  align-self: center; /* Ensures it stays aligned with the text */
  position: relative;
  top: 0.15em; /* Fine-tune positioning to match h1 */
}


/* === Colored Headings === */
.droughts {
  color: #DEB887;
}

.floods {
  color: #089c9d;
}

.fires {
  color: rgb(153, 0, 0);
}
</style>

