<script setup>
import { onMounted, ref, watch } from "vue";
import * as d3 from "d3";
import "leaflet/dist/leaflet.css";
import { LMap, LTileLayer, LGeoJson, LPopup } from "@vue-leaflet/vue-leaflet";

// Reactive state for zoom and geoJSON data
const zoom = ref(2);
const geoJsonData = ref(null);
const chartData = ref([]);
const riskColumn = ref("inform_risk"); // Default to INFORM Risk

// Load and merge GeoJSON with INFORM risk data
onMounted(async () => {
  try {
    // Load the CSV data
    const data = await d3.csv("./data/INFORM_Risk_2025_v069_index.csv");

    // Normalize the data
    const riskData = data.map(row => {
      let normalizedRow = {};
      for (const [key, value] of Object.entries(row)) {
        const normalizedKey = key.toLowerCase().replace(/\s+/g, "_");
        normalizedRow[normalizedKey] = value;
      }
      normalizedRow.inform_risk = +normalizedRow.inform_risk;
      normalizedRow.vulnerability = +normalizedRow.vulnerability;
      return normalizedRow;
    });

    chartData.value = riskData;

    // Load the GeoJSON
    const geoJsonResponse = await fetch("/data/country-boundaries.geojson");
    const geoJson = await geoJsonResponse.json();

    // Merge INFORM data with GeoJSON
    geoJson.features.forEach(feature => {
      let riskInfo = null;

      // Match by country name or ISO3 code
      const countryName = feature.properties.SOVEREIGNT;
      riskInfo = riskData.find(d => d.country === countryName);

      if (!riskInfo) {
        const countryIso3 = feature.properties.SOV_A3;
        riskInfo = riskData.find(d => d.iso3 === countryIso3);
      }

      if (riskInfo) {
        feature.properties.inform_risk = riskInfo.inform_risk;
        feature.properties.vulnerability = riskInfo.vulnerability;
      } else {
        feature.properties.inform_risk = null;
        feature.properties.vulnerability = null;
      }
    });

    geoJsonData.value = geoJson;
  } catch (error) {
    console.error("Error loading data:", error);
  }
});

// Color scale for selected column (either INFORM Risk or VULNERABILITY)
const colorScale = d3.scaleLinear()
  .domain([0, 2, 4, 6, 8, 10])
  .range(["#7FFF00", "#FFFF00", "#FF8000", "#FF0000", "#820747", "#D3D3D3"]);

// GeoJSON styling function
const geoJsonStyle = (feature) => {
  const riskValue = feature.properties[riskColumn.value];
  return {
    fillColor: riskValue !== null && riskValue !== undefined ? colorScale(riskValue) : "#D3D3D3",
    weight: 1,
    opacity: 1,
    color: "black",
    fillOpacity: 1
  };
};


// Function to create/update the bar chart
const createChart = (data) => {
  // Rebuild chart using d3.js based on selected risk column
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

  const maxRisk = d3.max(data, (d) => d[riskColumn.value]);

  const x = d3.scaleLinear().domain([0, maxRisk]).nice().range([0, width]);
  const y = d3.scaleBand().domain(data.map((d) => d.country))
    .range([0, svgHeight - margin.top - margin.bottom])
    .padding(0.1);

  svg.selectAll(".bar")
    .data(data)
    .enter()
    .append("rect")
    .attr("class", "bar")
    .attr("x", 0)
    .attr("y", (d) => y(d.country))
    .attr("width", (d) => x(d[riskColumn.value]))
    .attr("height", y.bandwidth())
    .attr("fill", (d) => colorScale(d[riskColumn.value]));

  svg.append("g").attr("class", "x-axis").call(d3.axisTop(x));
  svg.append("g").attr("class", "y-axis").call(d3.axisLeft(y));
};

// Watch for changes to riskColumn and update the chart and map
watch(riskColumn, () => {
  createChart(chartData.value);
  // Reapply GeoJSON style when riskColumn changes
  geoJsonData.value && geoJsonData.value.features.forEach(feature => {
    feature.properties.fillColor = feature.properties[riskColumn.value] !== null && feature.properties[riskColumn.value] !== undefined
      ? colorScale(feature.properties[riskColumn.value])
      : "#D3D3D3";
  });
});

watch(chartData, (newData) => {
  if (newData.length > 0) {
    createChart(newData);
  }
});

</script>


<template>
  <div class="container">
    <h3 class="main-title">INFORM Risk 2025</h3>

    <!-- Filter Controls (Radio buttons) -->
    <div class="filter-controls">
      <label>
        <input type="radio" v-model="riskColumn" value="inform_risk" /> INFORM Risk
      </label>
      <label>
        <input type="radio" v-model="riskColumn" value="vulnerability" /> Vulnerability
      </label>
    </div>

    <div class="content">
      <!-- Leaflet Map -->
      <div class="map-container">
        <l-map ref="map" v-model:zoom="zoom" :center="[0, 0]" :zoom="zoom">
          <l-tile-layer
            url="https://{s}.basemaps.cartocdn.com/light_all/{z}/{x}/{y}{r}.png"
            layer-type="base"
            attribution="OpenStreetMap | contributors: CartoDB"
            subdomains="abcd"
          ></l-tile-layer>

          <l-geo-json :geojson="geoJsonData" :options-style="geoJsonStyle">
            <template #popup="{ feature }">
              <l-popup>
                <strong>{{ feature.properties.name }}</strong><br>
                Risk Level: {{ feature.properties[riskColumn.value] || "N/A" }}
              </l-popup>
            </template>
          </l-geo-json>
        </l-map>
      </div>

      <!-- INFORM Risk Index Chart -->
      <div class="chart-container">
        <div id="INFORM-chart"></div>
      </div>
    </div>
  </div>
</template>

<style scoped>
/* Layout */
.container {
  display: flex;
  flex-direction: column;
  align-items: center;
  width: 100%;
}

.main-title {
  text-align: center;
  font-size: 24px;
  font-weight: bold;
  margin-bottom: 10px;
  width: 100%;
}

.content {
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

.bar:hover {
  fill: orange;
}

.filter-controls {
  margin: 10px 0;
  text-align: center;
}
</style>
