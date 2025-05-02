<template>
  <div class="map-container">
    <l-map ref="map" v-model:zoom="zoom" :center="[20, 0]" :zoom="zoom" :max-bounds="[[-60, -180], [85, 180]]">
  <l-geo-json v-if="geoJsonData" :geojson="geoJsonData" />
</l-map>
  </div>
</template>

<script>
import "leaflet/dist/leaflet.css";
import { LMap, LGeoJson } from "@vue-leaflet/vue-leaflet";

export default {
  components: {
    LMap,
    LGeoJson,
  },
  data() {
    return {
      zoom: 3,
      geoJsonData: null,  // We initialize it as null and will set it after fetching the file
    };
  },
  mounted() {
  fetch("./data/country-boundaries.geojson")
    .then(response => response.json())
    .then(data => {
      const filtered = {
        ...data,
        features: data.features.filter(f => f.properties.ADMIN !== "Antarctica"),
      };
      this.geoJsonData = filtered;
    })
    .catch(error => {
      console.error("Error loading GeoJSON file:", error);
    });
},
};
</script>

<style>
/* Full screen height and width */
html,
body {
  height: 100%;
  margin: 0;
}

/* Flexbox container to center the map */
.map-container {
  display: flex;
  justify-content: center;
  align-items: center;
  height: calc(100vh - 11rem);
  /* Full height minus some padding */
  width: calc(100vw - 13rem);
  /* Full width of the viewport */
}

l-map {
  height: 600px;
  width: 50%;
}
</style>
