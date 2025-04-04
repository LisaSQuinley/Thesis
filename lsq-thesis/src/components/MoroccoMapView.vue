<template>
  <div class="morocco-map-container">
    <l-map ref="map" v-model:zoom="zoom" :center="[28.7917, -7.9926]">
      <l-tile-layer
        url="https://{s}.basemaps.cartocdn.com/light_all/{z}/{x}/{y}{r}.png"
        layer-type="base"
        attribution="OpenStreetMap | contributors: CartoDB"
        subdomains="abcd"
      ></l-tile-layer>
    </l-map>
  </div>
</template>

<script>
import "leaflet/dist/leaflet.css";
import { LMap, LTileLayer } from "@vue-leaflet/vue-leaflet";
import * as d3 from "d3"; // Import D3.js

export default {
  components: {
    LMap,
    LTileLayer,
  },
  data() {
    return {
      zoom: 5,
    };
  },
  mounted() {
    this.loadGeoJsonMap();
  },
  methods: {
    async loadGeoJsonMap() {
  try {
    const data = await d3.csv("./data/EM-DAT-data-ALL.csv");

    // Filter for ISO = "MAR" and Disaster Subgroup is "Meteorological" or "Hydrological"
    const filteredData = data.filter(d => 
      d.ISO === "MAR" && (d["Disaster Subgroup"] === "Meteorological" || d["Disaster Subgroup"] === "Hydrological")
    );

    // Perform rollup grouping by Disaster Type, then Disaster Subtype, then Start Year
    const rolledUpData = d3.rollup(
      filteredData,
      v => v.length,  // Count occurrences
      d => d["Disaster Type"], 
      d => d["Disaster Subtype"], 
      d => d["Start Year"]
    );

    console.log("Rolled Up Data for MAR (Meteorological & Hydrological):", rolledUpData);
  } catch (error) {
    console.error("Error loading CSV:", error);
  }
}


  },
};
</script>

<style>
/* Full screen height and width */
html, body {
  height: 100%;
  margin: 0;
}

/* Flexbox container to center the map */
.morocco-map-container {
  display: flex;
  justify-content: center;
  align-items: center;
  width: 100%; /* Full width of the viewport */
  height: 700px;
}

l-map {
  height: 600px;
  width: 600px;
}
</style>
