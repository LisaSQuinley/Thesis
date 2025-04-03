<script setup>
import { onMounted, ref, computed, watch } from "vue";
import * as d3 from "d3";
import "leaflet/dist/leaflet.css";
import { LMap, LTileLayer, LGeoJson } from "@vue-leaflet/vue-leaflet";

// Reactive state
const zoom = ref(2);
const geoJsonData = ref(null);
const chartData = ref([]);
const sortOption = ref("descending"); // Default sorting order

// Load GeoJSON when component mounts
onMounted(() => {
  fetch("/data/country-boundaries.geojson")
    .then((response) => response.json())
    .then((data) => {
      geoJsonData.value = data;
    })
    .catch((error) => console.error("Error loading GeoJSON file:", error));

  // Load and process CSV data
  d3.csv("./data/INFORM_Risk_2025_v069_index.csv")
    .then((data) => {
      data = data.map((row) => {
        let normalizedRow = {};
        for (const [key, value] of Object.entries(row)) {
          const normalizedKey = key.toLowerCase().replace(/\s+/g, "_");
          normalizedRow[normalizedKey] = value;
        }
        normalizedRow.inform_risk = +normalizedRow.inform_risk;
        return normalizedRow;
      });

      chartData.value = data;
    })
    .catch((error) => console.error("Error loading CSV:", error));
});

// Sorting function
const sortData = (data, option) => {
  if (option === "alphabetical") {
    return [...data].sort((a, b) => a.country.localeCompare(b.country));
  } else if (option === "ascending") {
    return [...data].sort((a, b) => a.inform_risk - b.inform_risk);
  } else {
    return [...data].sort((a, b) => b.inform_risk - a.inform_risk);
  }
};

// Computed property for sorted data
const sortedData = computed(() => (chartData.value.length ? sortData(chartData.value, sortOption.value) : []));

// Function to create/update the bar chart
const createChart = (data) => {
  d3.select("#INFORM-chart").select("svg").remove();

  const margin = { top: 40, right: 30, bottom: 40, left: 150 };
  const width = 500 - margin.left - margin.right;
  const barHeight = 25;
  const svgHeight = data.length * barHeight + margin.top + margin.bottom;

  const svg = d3.select("#INFORM-chart")
    .append("svg")
    .attr("width", width + margin.left + margin.right)
    .attr("height", svgHeight)
    .append("g")
    .attr("transform", `translate(${margin.left},${margin.top})`);

  const maxRisk = d3.max(data, (d) => d.inform_risk);

  const colorScale = d3.scaleOrdinal()
    .domain(["Very Low", "Low", "Medium", "High", "Very High"])
    .range(["#7FFF00", "#FFFF00", "#FF8000", "#FF0000", "#820747"]);

  const x = d3.scaleLinear().domain([0, maxRisk]).nice().range([0, width]);

  const y = d3.scaleBand()
    .domain(data.map((d) => d.country))
    .range([0, svgHeight - margin.top - margin.bottom])
    .padding(0.1);

  svg
    .selectAll(".bar")
    .data(data)
    .enter()
    .append("rect")
    .attr("class", "bar")
    .attr("x", 0)
    .attr("y", (d) => y(d.country))
    .attr("width", (d) => x(d.inform_risk))
    .attr("height", y.bandwidth())
    .attr("fill", (d) => colorScale(d.risk_class));

  svg.append("g").attr("class", "x-axis").call(d3.axisTop(x));
  svg.append("g").attr("class", "y-axis").call(d3.axisLeft(y));
};

// Watch for changes and update chart
watch(sortedData, (newData) => {
  if (newData.length > 0) {
    createChart(newData);
  }
}, { immediate: true });

</script>

<template>
  <div class="container">
    <!-- Leaflet Map -->
    <div class="map-container">
      <l-map ref="map" v-model:zoom="zoom" :center="[0, 0]" :zoom="zoom">
        <l-tile-layer
          url="https://{s}.basemaps.cartocdn.com/light_all/{z}/{x}/{y}{r}.png"
          layer-type="base"
          attribution="OpenStreetMap | contributors: CartoDB"
          subdomains="abcd"
        ></l-tile-layer>

        <l-geo-json :geojson="geoJsonData"></l-geo-json>
      </l-map>
    </div>

    <!-- INFORM Risk Index Chart -->
    <div class="chart-container">
      <h3 class="chart-title">INFORM Risk Index 2025</h3>

      <div class="filter-controls">
        <label>
          <input type="radio" v-model="sortOption" value="alphabetical" /> Alphabetical
        </label>
        <label>
          <input type="radio" v-model="sortOption" value="ascending" /> Increasing Risk
        </label>
        <label>
          <input type="radio" v-model="sortOption" value="descending" /> Decreasing Risk
        </label>
      </div>

      <div id="INFORM-chart"></div>
    </div>
  </div>
</template>

<style scoped>
h3 {
  margin: 0px;
}
/* Main container layout */
.container {
  display: flex;
  justify-content: space-between;
  align-items: flex-start;
  height: 700px;
  width: 100%;
}

/* Map container */
.map-container {
  width: 50%;
  height: 100%;
}

l-map {
  height: 100%;
  width: 100%;
}

/* Chart container */
.chart-container {
  width: 50%;
  height: 100%;
  overflow-y: auto;
}

#INFORM-chart {
  width: 100%;
  overflow-y: auto;
}

.bar {
  transition: all 0.3s ease;
}
.bar:hover {
  fill: orange;
}

.filter-controls {
  margin: 10px 0;
  text-align: center;
}

.filter-controls label {
  margin: 0 10px;
  cursor: pointer;
}
</style>
