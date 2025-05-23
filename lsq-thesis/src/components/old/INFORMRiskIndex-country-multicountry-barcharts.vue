<template>
  <div class="container">
    <h3 class="main-title">On the Global Map - INFORM Risk</h3>
    <div class="content">
      
      <!-- Leaflet Map -->
      <div class="map-container">
        <l-map ref="map" v-model:zoom="zoom" :center="[0, 0]" :zoom="zoom">
          <l-tile-layer url="https://{s}.basemaps.cartocdn.com/light_all/{z}/{x}/{y}{r}.png" layer-type="base"
            attribution="OpenStreetMap | contributors: CartoDB" subdomains="abcd"></l-tile-layer>
          <l-geo-json :geojson="geoJsonData" :options-style="geoJsonStyle" :options="geoJsonOptions">
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
        <div class="column-info">
          <h4>
            {{ selectedCountries.length === 1 ? selectedCountries[0].properties.SOVEREIGNT : "" }}
          </h4>
        </div>
        <div id="INFORM-chart" class="multi-chart-container"></div>
      </div>
    </div>
  </div>
</template>



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
const selectedCountries = ref([]);

let riskData = [];

const geoJsonOptions = {
  onEachFeature: (feature, layer) => {
    layer.on({
      click: (e) => handleCountryClick(e, feature) // Pass the feature explicitly
    });
  }
};

const handleCountryClick = (event, feature) => {
  const countryName = feature.properties.SOVEREIGNT;
  const alreadySelectedIndex = selectedCountries.value.findIndex(
    (c) => c.properties.SOVEREIGNT === countryName
  );

  if (alreadySelectedIndex !== -1) {
    // Deselect the country
    selectedCountries.value.splice(alreadySelectedIndex, 1);
  } else {
    if (selectedCountries.value.length >= 10) {
      alert("You can only select up to 10 countries.");
      return;
    }
    selectedCountries.value.push(feature);
  }

  geoJsonData.value = { ...geoJsonData.value }; // trigger reactivity

  if (selectedCountries.value.length === 0) {
    createChart(chartData.value);
  } else if (selectedCountries.value.length === 1) {
    createCountryCharts(selectedCountries.value.map(c => c.properties));
  } else {
    createMultiCountryCharts(selectedCountries.value.map(c => c.properties));
  }


  console.log("Selected countries:", selectedCountries.value.map(f => f.properties.SOVEREIGNT));
};

const createCountryCharts = (countries) => {
  d3.select("#INFORM-chart").selectAll("svg").remove();
  countries.forEach(country => {
    createCountryChart(country);
  });
};


const handleResize = () => {
  windowWidth.value = window.innerWidth;

  // Recreate the chart based on selected countries
  if (selectedCountries.value.length === 0) {
    createChart(chartData.value);
  } else if (selectedCountries.value.length === 1) {
    createCountryCharts(selectedCountries.value.map(c => c.properties));
  } else {
    createMultiCountryCharts(selectedCountries.value.map(c => c.properties));
  }
};


onMounted(() => {
  window.addEventListener("resize", handleResize);
  loadGeoJsonMap();
  createChart(chartData.value);
});

onBeforeUnmount(() => {
  window.removeEventListener("resize", handleResize);
});

const colorScale = d3.scaleThreshold()
  .domain([0, 2, 4, 6, 8, 10])
  .range(["white", "#7FFF00", "#FFFF00", "#FF8000", "#FF0000", "#820747", "#D3D3D3"]);

const geoJsonStyle = (feature) => {
  const riskValue = feature.properties[riskColumn.value];
  const isSelected = selectedCountries.value.some(
    (c) => c.properties.SOVEREIGNT === feature.properties.SOVEREIGNT
  );
  return {
    fillColor: riskValue !== null && riskValue !== undefined ? colorScale(riskValue) : "#D3D3D3",
    weight: isSelected ? 3 : 0.5,
    opacity: 1,
    color: isSelected ? "black" : "gray",
    fillOpacity: 1
  };
};

const loadGeoJsonMap = async () => {
  try {
    const data = await d3.csv("./data/INFORM_Risk_2025_v069_index.csv");
    riskData = data.map(row => {
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

  const margin = { top: 40, right: 30, bottom: 40, left: 195 };
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

const createCountryChart = (country) => {
  const chartContainer = d3.select("#INFORM-chart")
    .append("div")
    .attr("class", "country-chart");


  const margin = { top: 27, right: 30, bottom: 40, left: 195 };
  const width = 0.9 * (windowWidth.value / 2) - margin.left - margin.right;
  const barHeight = 30;
  const keys = [
    { key: "inform_risk", label: "INFORM Risk" },
    { key: "vulnerability", label: "Vulnerability" },
    { key: "hazard_and_exposure", label: "Hazard & Exposure" },
    { key: "lack_of_coping_capacity", label: "Lack of Coping Capacity" }
  ];

  const data = keys.map(d => ({
    label: d.label,
    value: +country[d.key],
    color: colorScale(country[d.key])
  }));

  const svgHeight = data.length * barHeight + margin.top + margin.bottom;

  const x = d3.scaleLinear().domain([0, 10]).range([0, width]).nice();
  const y = d3.scaleBand()
    .domain(data.map(d => d.label))
    .range([0, svgHeight - margin.top - margin.bottom])
    .padding(0.1);

  const svg = chartContainer.append("svg")
    .attr("width", width + margin.left + margin.right)
    .attr("height", svgHeight)
    .append("g")
    .attr("transform", `translate(${margin.left},${margin.top})`);


  svg.selectAll(".bar")
    .data(data)
    .enter()
    .append("rect")
    .attr("class", "bar")
    .attr("y", d => y(d.label))
    .attr("x", 0)
    .attr("height", y.bandwidth())
    .attr("fill", d => d.color)
    .attr("width", 0)
    .transition()
    .duration(1000)
    .attr("width", d => x(d.value));

  svg.selectAll(".bar-text")
    .data(data)
    .enter()
    .append("text")
    .attr("class", "bar-text")
    .attr("x", d => x(d.value) + 5)
    .attr("y", d => y(d.label) + y.bandwidth() / 2)
    .attr("dy", ".35em")
    .text(d => d.value.toFixed(1))
    .style("fill", "#2c3e50;")
    .style("font-size", "12px")
    .style("font-weight", "bold");

  svg.append("g").attr("class", "x-axis").call(d3.axisTop(x));
  svg.append("g").attr("class", "y-axis").call(d3.axisLeft(y));
};

const createMultiCountryCharts = (countries) => {
  d3.select("#INFORM-chart").selectAll("svg").remove();

  const keys = [
    { key: "inform_risk", label: "INFORM Risk" },
    { key: "vulnerability", label: "Vulnerability" },
    { key: "hazard_and_exposure", label: "Hazard & Exposure" },
    { key: "lack_of_coping_capacity", label: "Lack of Coping Capacity" }
  ];

  const margin = { top: 50, right: 30, bottom: 40, left: 195 };
  const width = 0.9 * (windowWidth.value / 2) - margin.left - margin.right;
  const barHeight = 30;
  const svgHeight = countries.length * barHeight + margin.top + margin.bottom;

  keys.forEach(({ key, label }) => {
    const data = countries.map((c) => ({
      country: c.SOVEREIGNT,
      value: +c[key],
      color: colorScale(c[key])
    }));

    const x = d3.scaleLinear().domain([0, 10]).range([0, width]).nice();
    const y = d3.scaleBand()
      .domain(data.map(d => d.country))
      .range([0, svgHeight - margin.top - margin.bottom])
      .padding(0.1);

    const svg = d3.select("#INFORM-chart")
      .append("svg")
      .attr("width", width + margin.left + margin.right)
      .attr("height", svgHeight)
      .append("g")
      .attr("transform", `translate(${margin.left},${margin.top})`);

    // Title
    svg.append("text")
      .attr("x", (width + margin.left + margin.right) / 2 - margin.left)
      .attr("y", -30)
      .attr("text-anchor", "middle")
      .text(label)
      .attr("font-size", "16px")
      .attr("fill", "#2c3e50")
      .attr("font-weight", "bold")

    // Bars
    svg.selectAll(".bar")
      .data(data)
      .enter()
      .append("rect")
      .attr("class", "bar")
      .attr("y", d => y(d.country))
      .attr("height", y.bandwidth())
      .attr("x", 0)
      .attr("fill", d => d.color)
      .attr("width", 0)
      .transition()
      .duration(800)
      .attr("width", d => x(d.value));

    // Labels
    svg.selectAll(".bar-text")
      .data(data)
      .enter()
      .append("text")
      .attr("class", "bar-text")
      .attr("x", d => x(d.value) + 5)
      .attr("y", d => y(d.country) + y.bandwidth() / 2)
      .attr("dy", ".35em")
      .text(d => d.value.toFixed(1))
      .style("fill", "black")
      .style("font-size", "12px")
      .style("font-weight", "bold");

    // Axes
    svg.append("g").attr("class", "x-axis").call(d3.axisTop(x));
    svg.append("g").attr("class", "y-axis").call(d3.axisLeft(y));
  });
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

watch(sortMethod, () => {
  createChart(chartData.value); // Recreate the chart with the new sort method
});

watch(chartData, (newData) => {
  if (newData.length > 0) {
    createChart(newData);
  }
});
</script>

<style scoped>
/* Layout */
.container {
  display: flex;
  flex-direction: column;
  height: 100%;
  /* Fill the height of parent instead of viewport */
  width: 100%;
  box-sizing: border-box;
  /* Respect parent padding if any */
  padding: 4rem 5rem 5rem 5rem;
}


.main-title {
  text-align: center;
  margin: 10px 0;
}

.content {
  flex: 1;
  /* Take up the remaining space */
  display: flex;
  justify-content: space-between;
  align-items: stretch;
  width: 100%;
  overflow: hidden;
  /* Prevent content from spilling */
}

/* Map container */
.map-container {
  width: 50%;
  height: 100%;
  position: relative;
}

l-map {
  width: 100%;
  height: 100%;
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

.map-container {
  position: relative;
}

.map-overlay-message {
  position: absolute;
  bottom: 25px;
  left: 10px;
  right: 10px;
  background-color: snow;
  padding: 12px 16px;
  border-radius: 4px;
  font-size: 14px;
  color: #2c3e50;
  z-index: 999;
}

.country-info {
  margin-top: 20px;
  padding: 10px;
  background-color: #f0f0f0;
  border-radius: 8px;
}

.country-info h4 {
  color: #2c3e50;
  margin-bottom: 10px;
}

.country-info p {
  margin: 5px 0;
}
</style>
