<template>
  <div>
    <HelloMorocco msg="Droughts! Floods! Heat Waves! Oh My!"/>
    <div class="charts-wrapper">
      <MoroccoMapView />
    </div>
    <div class="charts-wrapper">
      <WorldMapViewTimeline />
      <INFORMRiskIndex />
    </div>
  </div>
</template>

<script>
import HelloMorocco from './components/HelloMorocco.vue'
import MoroccoMapView from './components/MoroccoMapView.vue'
import WorldMapViewTimeline from './components/WorldMapViewTimeline.vue'
import INFORMRiskIndex from './components/INFORMRiskIndex.vue'

export default {
  name: 'App',
  components: {
    HelloMorocco, 
    MoroccoMapView,
    WorldMapViewTimeline,
    INFORMRiskIndex,
  }
}
</script>

<style>
#app {
  font-family: Avenir, Helvetica, Arial, sans-serif;
  -webkit-font-smoothing: antialiased;
  -moz-osx-font-smoothing: grayscale;
  text-align: center;
  color: #2c3e50;
  margin-top: 30px;
}

.charts-wrapper {
  display: flex; /* Use flexbox to align items horizontally */
  justify-content: space-between; /* Optional: Space items evenly */
  gap: 20px; /* Optional: Add space between components */
  margin-top: 20px; /* Optional: Add some space above */
  width: 90%;
  justify-content: center;
  align-items: center;
  margin: 20px auto;
}
</style>
