<script setup>
import { onMounted, ref, watch, onBeforeUnmount } from "vue";
import * as d3 from "d3";
import "leaflet/dist/leaflet.css";
import { LMap, LTileLayer, LGeoJson, LPopup } from "@vue-leaflet/vue-leaflet";

const zoom = ref(2);
const geoJsonData = ref(null);
const chartData = ref([]);
const sortMethod = ref("alphabetical");
const riskColumn = ref("inform_risk");
const windowWidth = ref(window.innerWidth);

const handleResize = () => {
  windowWidth.value = window.innerWidth;
  createChart(chartData.value); // Recreate the chart with the new width
};

onMounted(() => {
  window.addEventListener("resize", handleResize);
  loadGeoJsonMap();
  createChart(chartData.value); // Create the chart initially with the loaded data
});

onBeforeUnmount(() => {
  window.removeEventListener("resize", handleResize);
});

const colorScale = d3.scaleThreshold()
  .domain([0, 2, 4, 6, 8, 10])
  .range(["white", "#7FFF00", "#FFFF00", "#FF8000", "#FF0000", "#820747", "#D3D3D3"]);

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

const loadGeoJsonMap = async () => {
  try {
    const data = await d3.csv("./data/INFORM_Risk_2025_v069_index.csv");
    const riskData = data.map(row => {
      let normalizedRow = {};
      for (const [key, value] of Object.entries(row)) {
        const normalizedKey = key.toLowerCase().replace(/\s+/g, "_").replace(/&/g, "and");
        normalizedRow[normalizedKey] = value;
      }
      normalizedRow.inform_risk = +normalizedRow.inform_risk;
      normalizedRow.vulnerability = +normalizedRow.vulnerability;
      normalizedRow.hazard_and_exposure = +normalizedRow.hazard_and_exposure;
      normalizedRow.lack_of_coping_capacity = +normalizedRow.lack_of_coping_capacity;
      return normalizedRow;
    });
    chartData.value = riskData;

    const geoJsonResponse = await fetch("./data/country-boundaries.geojson");
    const geoJson = await geoJsonResponse.json();

    geoJson.features.forEach(feature => {
      let riskInfo = riskData.find(d => d.country === feature.properties.SOVEREIGNT) ||
        riskData.find(d => d.iso3 === feature.properties.SOV_A3);
      if (riskInfo) {
        feature.properties.inform_risk = riskInfo.inform_risk;
        feature.properties.vulnerability = riskInfo.vulnerability;
        feature.properties.hazard_and_exposure = riskInfo.hazard_and_exposure;
        feature.properties.lack_of_coping_capacity = riskInfo.lack_of_coping_capacity;
      } else {
        feature.properties.inform_risk = null;
        feature.properties.vulnerability = null;
        feature.properties.hazard_and_exposure = null;
        feature.properties.lack_of_coping_capacity = null;
      }
    });
    geoJsonData.value = geoJson;
  } catch (error) {
    console.error("Error loading data:", error);
  }
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

  // not using maxRisk for the x-axis domain because I want to keep it fixed at [0, 10]
  // const maxRisk = d3.max(sortedData, (d) => d[riskColumn.value]);

  const x = d3.scaleLinear()
    .domain([0, 10]) // change domain to [0, 10] instead of [0, maxRisk]
    .nice()
    .range([0, width]);
  const y = d3.scaleBand()
    .domain(sortedData.map((d) => d.country))
    .range([0, svgHeight - margin.top - margin.bottom])
    .padding(0.1);

  // Bind data to bars
  const bars = svg.selectAll(".bar")
    .data(sortedData, (d) => d.country); // Use key function for updates

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

  // Append text for the data point on the right side of the bar
  bars.enter()
    .append("text")
    .attr("class", "bar-text")
    .attr("x", (d) => x(d[riskColumn.value]) + 5) // Position text slightly to the right of the bar
    .attr("y", (d) => y(d.country) + y.bandwidth() / 2)
    .attr("dy", ".35em") // Vertically center the text
    .text((d) => d[riskColumn.value].toFixed(1)) // Display the risk value to one decimal place
    .style("fill", "black")
    .style("font-size", "12px")
    .style("font-weight", "bold");

  // Update phase (bars that already exist)
  bars.transition()
    .duration(1000)
    .ease(d3.easeCubicOut)
    .attr("y", (d) => y(d.country)) // Update position
    .attr("width", (d) => x(d[riskColumn.value])) // Animate width change
    .attr("fill", (d) => colorScale(d[riskColumn.value]));

  // Update text for the data point
  bars.select(".bar-text")
    .transition()
    .duration(1000)
    .ease(d3.easeCubicOut)
    .attr("x", (d) => x(d[riskColumn.value]) + 5) // Adjust position with width change
    .text((d) => d[riskColumn.value].toFixed(2));

  // Remove phase (bars that no longer have data)
  bars.exit()
    .transition()
    .duration(500)
    .attr("width", 0)
    .remove();

  // Remove text for data points when the bar is removed
  bars.exit()
    .select(".bar-text")
    .transition()
    .duration(500)
    .attr("x", 0)
    .remove();

  // Append x-axis
  svg.append("g").attr("class", "x-axis").call(d3.axisTop(x));
  svg.append("g").attr("class", "y-axis").call(d3.axisLeft(y));
};




watch(riskColumn, () => {
  createChart(chartData.value);

  if (geoJsonData.value) {
    geoJsonData.value.features.forEach(feature => {
      feature.properties.fillColor = feature.properties[riskColumn.value] !== null && feature.properties[riskColumn.value] !== undefined
        ? colorScale(feature.properties[riskColumn.value])
        : "#D3D3D3";
    });

    // Force Vue to recognize the change in geoJsonData
    geoJsonData.value = { ...geoJsonData.value };
  }
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
          <l-tile-layer url="https://{s}.basemaps.cartocdn.com/light_all/{z}/{x}/{y}{r}.png" layer-type="base"
            attribution="OpenStreetMap | contributors: CartoDB" subdomains="abcd"></l-tile-layer>

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
