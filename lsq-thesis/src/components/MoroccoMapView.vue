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
    width: 100%;    /* Full width of the viewport */
    height: 700px;
  }
  
  l-map {
    height: 600px;
    width: 600px;
  }
  </style>
  