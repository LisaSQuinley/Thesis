<template>
  <div>
    <HelloMorocco msg="Droughts! Floods! Fires!"/>
    <div class="charts-wrapper">
      <MoroccoMapView />
    </div>
    <div class="charts-wrapper" >
      <RainfallPrecipitation />
    </div>
    <div class="charts-wrapper" >
      <MeanTemperature />
    </div>
    <div class="charts-wrapper">
      <SheepCattleHerds />
      <LandPloughedWheat />
    </div>
    <div class="charts-wrapper">
      <INFORMRiskIndex />
    </div>
    <div class="charts-wrapper">
      <WorldMapViewTimeline />
    </div>
  </div>
  <div class="footer">
    <p>
      To check out other projects Lisa has created, visit
      <a href="https://lsq.design" target="_blank" rel="noopener">lsq.design</a>.
    </p>
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

export default {
  name: 'App',
  components: {
    HelloMorocco, 
    MoroccoMapView,
    RainfallPrecipitation,
    MeanTemperature,
    SheepCattleHerds,
    LandPloughedWheat,
    WorldMapViewTimeline,
    INFORMRiskIndex,
  }
}
</script>

<style>
#app {
  font-family: "Parkinsans", sans-serif;
  /* font-family: Avenir, Helvetica, Arial, sans-serif; */
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
}
text {
  font-family: "Parkinsans", sans-serif;
  margin: 0;
}

</style>
