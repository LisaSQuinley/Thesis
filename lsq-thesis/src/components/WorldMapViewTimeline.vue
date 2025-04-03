<script>
import "leaflet/dist/leaflet.css";
import { LMap, LTileLayer, LGeoJson } from "@vue-leaflet/vue-leaflet";

export default {
  components: {
    LMap,
    LTileLayer,
    LGeoJson,
  },
  data() {
    return {
      zoom: 2,
      geoJsonData: null,  // We initialize it as null and will set it after fetching the file
    };
  },
  mounted() {
    // Load the GeoJSON file when the component is mounted
    fetch("./data/country-boundaries.geojson")
      .then(response => response.json())
      .then(data => {
        this.geoJsonData = data;  // Assign the loaded GeoJSON data to the component data
      })
      .catch(error => {
        console.error("Error loading GeoJSON file:", error);
      });
  },
};
</script>

<template>
  <div class="map-container">
    <l-map ref="map" v-model:zoom="zoom" :center="[0, 0]" :zoom="zoom">
      <!-- Tile Layer -->
      <l-tile-layer
        url="https://{s}.basemaps.cartocdn.com/light_all/{z}/{x}/{y}{r}.png"
        layer-type="base"
        attribution="OpenStreetMap | contributors: CartoDB"
        subdomains="abcd"
      ></l-tile-layer>

      <!-- GeoJSON Layer -->
      <l-geo-json :geojson="geoJsonData"></l-geo-json>
    </l-map>
  </div>
</template>

<style>
/* Full screen height and width */
html, body {
  height: 100%;
  margin: 0;
}

/* Flexbox container to center the map */
.map-container {
  display: flex;
  justify-content: center;
  align-items: center;
  height: 700px;
  width: 50%;    /* Full width of the viewport */
}

l-map {
  height: 600px;
  width: 50%;
}
</style>
