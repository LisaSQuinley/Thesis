<script setup>
import { onMounted, ref, watch, onBeforeUnmount } from "vue";
import * as d3 from "d3";
import "leaflet/dist/leaflet.css";
import { LMap, LTileLayer, LGeoJson, LPopup } from "@vue-leaflet/vue-leaflet";

// Reactive state for zoom and geoJSON data
const zoom = ref(2);
const geoJsonData = ref(null);
const chartData = ref([]);
const sortMethod = ref("alphabetical");
const riskColumn = ref("inform_risk"); // Default to INFORM Risk

// Track the window width
const windowWidth = ref(window.innerWidth);

// Resize event handler
const handleResize = () => {
  windowWidth.value = window.innerWidth;
};

// Add event listener for window resize
onMounted(() => {
  window.addEventListener("resize", handleResize);
});

// Clean up the event listener when the component is destroyed
onBeforeUnmount(() => {
  window.removeEventListener("resize", handleResize);
});

// Load and merge GeoJSON with INFORM risk data
onMounted(async () => {
  try {
    // Load the CSV data
    const data = await d3.csv("./data/INFORM_Risk_2025_v069_index.csv");

    // Normalize the data
    const riskData = data.map(row => {
      let normalizedRow = {};
      for (const [key, value] of Object.entries(row)) {
        const normalizedKey = key.toLowerCase().replace(/\s+/g, "_").replace(/&/g, "and");
        normalizedRow[normalizedKey] = value;
      }
      // Convert the necessary columns to numbers
      normalizedRow.inform_risk = +normalizedRow.inform_risk;
      normalizedRow.vulnerability = +normalizedRow.vulnerability;
      normalizedRow.hazard_and_exposure = +normalizedRow.hazard_and_exposure;  // Adding this column
      normalizedRow.lack_of_coping_capacity = +normalizedRow.lack_of_coping_capacity;  // Adding this column
      return normalizedRow;
    });

    chartData.value = riskData;

    // Load the GeoJSON
    const geoJsonResponse = await fetch("./data/country-boundaries.geojson");
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
        feature.properties.hazard_and_exposure = riskInfo.hazard_and_exposure; 
        feature.properties.lack_of_coping_capacity = riskInfo.lack_of_coping_capacity; 
      } else {
        feature.properties.inform_risk = null;
        feature.properties.vulnerability = null;
        feature.properties.hazard_and_exposure = null;  // Default to null if no match
        feature.properties.lack_of_coping_capacity = null;  // Default to null if no match
      }
    });

    geoJsonData.value = geoJson;
  } catch (error) {
    console.error("Error loading data:", error);
  }
});
// Color scale for selected column (either INFORM Risk or VULNERABILITY)
// Using d3.scaleThreshold to define specific color ranges
const colorScale = d3.scaleThreshold()
  .domain([0, 2, 4, 6, 8, 10]) // These are the thresholds for color ranges
  .range(["white", "#7FFF00", "#FFFF00", "#FF8000", "#FF0000", "#820747", "#D3D3D3"]); // Colors for each range


// GeoJSON styling function
const geoJsonStyle = (feature) => {
  const riskValue = feature.properties[riskColumn.value];
  return {
    fillColor: riskValue !== null && riskValue !== undefined ? colorScale(riskValue) : "#D3D3D3",
    weight: 0.5,
    opacity: 1,
    color: "black",
    fillOpacity: 1
  };
};

const createChart = (data) => {
  // Remove the previous chart
  d3.select("#INFORM-chart").select("svg").remove();

  const margin = { top: 40, right: 30, bottom: 40, left: 175 };
  const width = 0.9 * (windowWidth.value / 2) - margin.left - margin.right;
  const barHeight = 25;
  const svgHeight = data.length * barHeight + margin.top + margin.bottom;

  // Sort the data based on the selected method
  let sortedData = [...data];
  if (sortMethod.value === "ascending") {
    sortedData.sort((a, b) => a[riskColumn.value] - b[riskColumn.value]);
  } else if (sortMethod.value === "descending") {
    sortedData.sort((a, b) => b[riskColumn.value] - a[riskColumn.value]);
  } else if (sortMethod.value === "alphabetical") {
    sortedData.sort((a, b) => a.country.localeCompare(b.country));
  }

  const svg = d3.select("#INFORM-chart")
    .append("svg")
    .attr("width", width + margin.left + margin.right)
    .attr("height", svgHeight)
    .append("g")
    .attr("transform", `translate(${margin.left},${margin.top})`);

  const maxRisk = d3.max(sortedData, (d) => d[riskColumn.value]);

  const x = d3.scaleLinear().domain([0, maxRisk]).nice().range([0, width]);
  const y = d3.scaleBand()
    .domain(sortedData.map((d) => d.country))
    .range([0, svgHeight - margin.top - margin.bottom])
    .padding(0.1);

  // Bind data to bars
  const bars = svg.selectAll(".bar")
    .data(sortedData, (d) => d.country) // Use key function for updates

  // Enter phase (initial bars)
  bars.enter()
    .append("rect")
    .attr("class", "bar")
    .attr("x", 0)
    .attr("y", (d) => y(d.country))
    .attr("height", y.bandwidth())
    .attr("fill", (d) => colorScale(d[riskColumn.value]))
    .attr("width", 0) // Start width at 0 for animation
    .transition()
    .duration(1000) // Duration for animation
    .ease(d3.easeCubicOut)
    .attr("width", (d) => x(d[riskColumn.value]));

  // Update phase (bars that already exist)
  bars.transition()
    .duration(1000)
    .ease(d3.easeCubicOut)
    .attr("y", (d) => y(d.country)) // Update position
    .attr("width", (d) => x(d[riskColumn.value])) // Animate width change
    .attr("fill", (d) => colorScale(d[riskColumn.value]));

  // Remove phase (bars that no longer have data)
  bars.exit()
    .transition()
    .duration(500)
    .attr("width", 0)
    .remove();

  // Append x-axis
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

// Watch for changes to sortMethod and update the chart
watch(sortMethod, () => {
  createChart(chartData.value); // Recreate the chart with the new sort method
});

// Watch for changes to chartData and update the chart
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
        <input type="radio" v-model="riskColumn" value="inform_risk" /> INFORM Risk Index
      </label>
      <label>
        <input type="radio" v-model="riskColumn" value="hazard_and_exposure" /> Hazard & Exposure
      </label>
      <label>
        <input type="radio" v-model="riskColumn" value="vulnerability" /> Vulnerability
      </label>
      <label>
        <input type="radio" v-model="riskColumn" value="lack_of_coping_capacity" /> Lack of Coping Capacity
      </label>
    </div>

    <!-- Sorting Controls (Radio buttons) -->
    <div class="sort-controls">
      <label>
        <input type="radio" v-model="sortMethod" value="alphabetical" /> Alphabetical
      </label>
      <label>
        <input type="radio" v-model="sortMethod" value="ascending" /> Ascending
      </label>
      <label>
        <input type="radio" v-model="sortMethod" value="descending" /> Descending
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

.filter-controls {
  margin: 0 0 10px 0;
  text-align: center;
}

/* Sorting Controls */
.sort-controls {
  margin: 10px 0;
  text-align: center;
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

h3 {
  margin: 0 0 10px 0;
}
</style>
