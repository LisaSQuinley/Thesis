<template>
  <div>
    <HelloMorocco msg="A Hot Topic for Morocco"/>
    <MapView />  <!-- Updated name here -->
  </div>
</template>

<script>
import HelloMorocco from './components/HelloMorocco.vue'
import MapView from './components/MapView.vue'  // Updated import here

export default {
  name: 'App',
  components: {
    HelloMorocco, 
    MapView
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
  margin-top: 60px;
}
</style>
