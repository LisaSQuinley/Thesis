<template>
  <div class="container" ref="componentRoot">
    <h3 class="main-title" v-if="isVisible">Zooming Into Risk</h3>
    <div class="content">

      <!-- Leaflet Map -->
      <div class="map-container">
        <l-map ref="map" v-model:zoom="zoom" :center="[20, 0]" :zoom-control="false">
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

      <!-- Radio Filters -->
      <div class="filter-controls" v-if="isVisible">
        <div class="radio-group">
          <label>
            <input type="radio" v-model="riskColumn" value="inform_risk" />
            INFORM Risk Index
          </label>
          <div class="descriptor">A score that measures how likely a country is to face crises and how much help it
            might
            need, based on three factors: Hazards & Exposure, Vulnerability, and Lack of Coping Capacity.</div>
        </div>

        <div class="radio-group">
          <label>
            <input type="radio" v-model="riskColumn" value="hazard_and_exposure" />
            Hazard & Exposure
          </label>
          <div class="descriptor">How much danger a country faces from storms, floods, earthquakes, and other events.
          </div>
        </div>

        <div class="radio-group">
          <label>
            <input type="radio" v-model="riskColumn" value="vulnerability" />
            Vulnerability
          </label>
          <div class="descriptor">How badly people and places could be hurt when disasters happen.</div>
        </div>

        <div class="radio-group">
          <label>
            <input type="radio" v-model="riskColumn" value="lack_of_coping_capacity" />
            Lack of Coping Capacity
          </label>
          <div class="descriptor">How ready a country is to handle emergencies and help its people.</div>
        </div>
      </div>


      <!-- INFORM Risk Index Chart -->
      <div class="chart-container" v-if="isVisible" v-show="selectedCountries.length > 0">
        <button class="reset-button" @click="deselectAllCountries">X</button>
        <div class="column-info"></div>
        <div id="INFORM-chart" class="multi-chart-container"></div>
      </div>
    </div>
  </div>
</template>

<script setup>
import { onMounted, ref, watch, onBeforeUnmount } from "vue";
import * as d3 from "d3";
import "leaflet/dist/leaflet.css";
import { LMap, LGeoJson, LPopup } from "@vue-leaflet/vue-leaflet";

const zoom = ref(3);
const geoJsonData = ref(null);
const chartData = ref([]);
const sortMethod = ref("alphabetical");
const riskColumn = ref("inform_risk");
const windowWidth = ref(window.innerWidth);
const selectedCountries = ref([]);
const ultimateBarHeight = 40;

let riskData = [];

const geoJsonOptions = {
  onEachFeature: (feature, layer) => {
    layer.on({
      click: (e) => {
        handleCountryClick(e, feature);
        layer.bringToFront(); // 👈 This brings the clicked layer to the front
      }
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

  if (selectedCountries.value.length === 1) {
    createCountryCharts(selectedCountries.value.map(c => c.properties));
  } else if (selectedCountries.value.length > 1) {
    createMultiCountryCharts(selectedCountries.value.map(c => c.properties));
  }
};

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
    weight: isSelected ? 5 : 2,
    opacity: 1,
    color: isSelected ? "#089c9d" : "#089c9d",
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

  const margin = { top: 40, right: 10, bottom: 10, left: 205 };
  const width = 0.8 * (windowWidth.value / 2) - margin.left - margin.right;
  const barHeight = ultimateBarHeight;
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
    .duration(1000)
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
    .style("fill", "#2c3e50");

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
  svg.append("g").attr("class", "x-axis").call(d3.axisTop(x)).selectAll("text").style("font-size", "1rem").style("fill", "#2c3e50");
  svg.append("g").attr("class", "y-axis").call(d3.axisLeft(y)).selectAll("text").style("font-size", "1rem").style("fill", "#2c3e50");
};

const createCountryChart = (country) => {
  const chartContainer = d3.select("#INFORM-chart")
    .append("div")
    .attr("class", "country-chart");

  const margin = { top: 40, right: 10, bottom: 10, left: 205 };
  const width = 0.8 * (windowWidth.value / 2) - margin.left - margin.right;
  const barHeight = ultimateBarHeight;
  const keys = [
    { key: "inform_risk", label: "INFORM Risk" },
    { key: "vulnerability", label: "Vulnerability" },
    { key: "hazard_and_exposure", label: "Hazard & Exposure" },
    { key: "lack_of_coping_capacity", label: "Lack of\nCoping Capacity" }
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
    .delay(500)
    .duration(500)
    .attr("width", d => x(d.value));

  svg.append("text")
    .attr("x", (width + margin.left + margin.right) / 2 - margin.left)
    .attr("y", -10)
    .attr("text-anchor", "middle")
    .text(country.SOVEREIGNT)
    .attr("font-size", "16px")
    .attr("fill", "#2c3e50")
    .attr("font-weight", "bold")

  svg.selectAll(".bar-text")
    .data(data)
    .enter()
    .append("text")
    .attr("class", "bar-text")
    .attr("x", d => x(d.value) + 5)
    .attr("y", d => y(d.label) + y.bandwidth() / 2)
    .attr("dy", ".35em")
    .text(d => d.value.toFixed(1));

  const xAxis = d3.axisTop(x);

  const xAxisGroup = svg.append("g")
    .attr("class", "x-axis")
    .call(xAxis);

  // Style all tick labels
  xAxisGroup.selectAll("text")
    .style("font-size", "1rem")
    .style("fill", "#2c3e50");

  // Remove all ticks except 10
  xAxisGroup.selectAll(".tick")
    .each(function (d) {
      if (d !== 10) {
        d3.select(this).select("text").remove();
        d3.select(this).select("line").remove();
      }
    });

  svg.append("g")
    .attr("class", "y-axis")
    .call(d3.axisLeft(y))
    .selectAll(".tick text")
    .each(function () {
      const self = d3.select(this);
      const fullText = self.text();

      // Only split into tspans if it's a multi-line label
      if (fullText.includes("\n")) {
        const lines = fullText.split("\n");
        self.text(null); // Clear existing text

        lines.forEach((line, i) => {
          self.append("tspan")
            .text(line)
            .attr("x", -10)
            .attr("dy", i === 0 ? "-0.25em" : "1.1em"); // First line up, rest normal
        });
      }
    })
    .style("font-size", "1rem")
    .style("fill", "#2c3e50");
};

const insertLineBreakBeforeOf = (label) => {
  return label.replace(/\b(\w+)\s(?=of\b)/gi, '$1\n');
};

const createMultiCountryCharts = (countries) => {
  d3.select("#INFORM-chart").selectAll("svg").remove();

  const keys = [
    { key: "inform_risk", label: "INFORM Risk" },
    { key: "vulnerability", label: "Vulnerability" },
    { key: "hazard_and_exposure", label: "Hazard & Exposure" },
    { key: "lack_of_coping_capacity", label: "Lack of Coping Capacity" }
  ];

  const margin = { top: 40, right: 10, bottom: 10, left: 205 };
  const width = 0.8 * (windowWidth.value / 2) - margin.left - margin.right;
  const barHeight = ultimateBarHeight;
  const svgHeight = countries.length * barHeight + margin.top + margin.bottom;

  keys.forEach(({ key, label }) => {
    const data = countries.map((c) => ({
      country: c.SOVEREIGNT,
      formattedCountry: insertLineBreakBeforeOf(c.SOVEREIGNT),
      value: +c[key],
      color: colorScale(c[key])
    }));

    const x = d3.scaleLinear().domain([0, 10]).range([0, width]).nice();
    const y = d3.scaleBand()
      .domain(data.map(d => insertLineBreakBeforeOf(d.country)))
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
      .attr("y", -10)
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
      .attr("y", d => y(d.formattedCountry))
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
      .attr("y", d => y(d.formattedCountry) + y.bandwidth() / 2)
      .attr("dy", ".35em")
      .text(d => d.value.toFixed(1));

    // Axes
    const xAxis = d3.axisTop(x);

    const xAxisGroup = svg.append("g")
      .attr("class", "x-axis")
      .call(xAxis);

    // Style all tick labels
    xAxisGroup.selectAll("text")
      .style("font-size", "1rem")
      .style("fill", "#2c3e50");

    // Remove all ticks except 10
    xAxisGroup.selectAll(".tick")
      .each(function (d) {
        if (d !== 10) {
          d3.select(this).select("text").remove();
          d3.select(this).select("line").remove();
        }
      });

    svg.append("g")
      .attr("class", "y-axis")
      .call(d3.axisLeft(y))
      .selectAll(".tick text")
      .each(function () {
        const self = d3.select(this);
        const fullText = self.text();

        // Only split into tspans if it's a multi-line label
        if (fullText.includes("\n")) {
          const lines = fullText.split("\n");
          self.text(null); // Clear existing text

          lines.forEach((line, i) => {
            self.append("tspan")
              .text(line)
              .attr("x", -10)
              .attr("dy", i === 0 ? "-0.25em" : "1.1em"); // First line up, rest normal
          });
        }
      })
      .style("font-size", "1rem")
      .style("fill", "#2c3e50");
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

const componentRoot = ref(null);
const isVisible = ref(false);

let observer;

onMounted(() => {
  window.addEventListener("resize", handleResize);
  observer = new IntersectionObserver(
    ([entry]) => {
      isVisible.value = entry.isIntersecting;

      if (!entry.isIntersecting) {
        // Deselect all countries and remove chart
        selectedCountries.value = [];
        d3.select("#INFORM-chart").selectAll("*").remove();
      }

    },
    {
      root: null, // viewport
      threshold: 0.1,
    },
    (entries) => {
      entries.forEach((entry) => {
        if (!entry.isIntersecting) {
          // The component has left the viewport, trigger deselect action
          deselectAllCountries();
        }
      });
    },
    {
      threshold: 0, // Trigger when the element is fully out of the viewport
    }
  );

  if (componentRoot.value) {
    observer.observe(componentRoot.value);
  }

  window.addEventListener("resize", handleResize);
  loadGeoJsonMap();
});

onBeforeUnmount(() => {
  if (componentRoot.value) {
    observer.unobserve(componentRoot.value);
  }
  window.removeEventListener("resize", handleResize);

});

const deselectAllCountries = () => {
  // Clear selected countries
  selectedCountries.value = [];

  geoJsonData.value.features.forEach(feature => {
    feature.properties.fillColor = "#D3D3D3"; // Reset to default color
    feature.properties.weight = 1; // Reset border width to 1
  });

  geoJsonData.value = { ...geoJsonData.value }; // Trigger reactivity

  // Optionally, reset the chart data if required
  if (chartData.value.length > 0) {
    createChart(chartData.value);
  }
};



</script>

<style scoped>
/* Layout */
.container {
  display: flex;
  flex-direction: column;
  position: relative;
  height: 100%;
  width: 100%;
  box-sizing: border-box;
  padding: 4rem 5rem 5rem 5rem;
}

.content {
  flex: 1;
  display: flex;
  justify-content: space-between;
  align-items: stretch;
  width: 100%;
  overflow: hidden;
}

.map-container {
  width: 100%;
  height: 100%;
  position: relative;
}

l-map {
  width: 100%;
  height: 100%;
}

.main-title {
  position: absolute;
  z-index: 1000;
  padding-top: 2rem;
  padding-left: 2rem;
}

@keyframes slideInFromRight {
  from {
    transform: translateX(100%);
    opacity: 0;
  }

  to {
    transform: translateX(0);
    opacity: 1;
  }
}

.chart-container {
  position: absolute;
  top: 5.5rem;
  right: 6.5rem;
  width: calc(50% - 7.5rem);
  max-height: calc(100% - 12rem);
  z-index: 1000;
  background: white;
  border: 1px solid #ccc;
  box-shadow: 0 0 10px rgba(0, 0, 0, 0.1);
  overflow: auto;

  /* animation settings */
  animation: slideInFromRight 0.5s ease-out forwards;
}

#INFORM-chart {
  width: 100%;
  overflow-y: auto;
}

h3 {
  font-size: 5em;
  font-weight: 800;
  color: #089d9d4f;
  text-transform: uppercase;
  position: absolute;
  bottom: 5rem;
  right: 7rem;
  margin: 0 0 10px 0;
  z-index: 1000;
}

.map-container {
  position: relative;
  width: 100%;
  height: 100%;
}

h4 {
  padding-top: 20px;
}

.country-info h4 {
  color: #2c3e50;
}

.country-info p {
  margin: 5px 0;
}

.leaflet-container {
  background-color: #40E0D0;
}

.filter-controls {
  position: absolute;
  bottom: 5rem;
  left: 5.25rem;
  z-index: 1000;
  padding: 10px;
  text-align: left;
  width: 30%;
}

.radio-group {
  margin-bottom: .25rem;
  position: relative;
}

.radio-group:hover .descriptor {
  max-height: 500px; /* adjust as needed */
  opacity: 1;
}

/* treat like an h5 */
.filter-controls label {
  display: inline-flex;
  align-items: center;
  gap: 0.25rem;
  font-size: 28px;
  font-weight: 300;
}

.radio-group label:hover + .descriptor,
.radio-group input:hover + .descriptor {
  max-height: 500px; /* large enough for your longest descriptor */
  opacity: 1;
}

/* treat like a global p */
.descriptor {
  padding-left: 5px;
  max-height: 0;
  opacity: 0;
  overflow: hidden;
  transition: max-height 0.3s ease, opacity 0.3s ease;
  margin-left: 1.75rem;
  font-weight: 400;
  margin-bottom: 0.5em;
  font-size: 22px;
  color: #2c3e50;
}

input[type="radio"] {
  appearance: none;
  -webkit-appearance: none;
  background-color: #fff;
  border: 2px solid #089d9d82;
  border-radius: 50%;
  width: 20px;
  height: 20px;
  cursor: pointer;
  position: relative;
}

input[type="radio"]:checked {
  border-color: #089c9d;
}

input[type="radio"]:checked::before {
  content: "";
  position: absolute;
  top: 3px;
  left: 3px;
  width: 10px;
  height: 10px;
  border-radius: 50%;
  background: #089c9d;
}

.reset-button {
  position: absolute;
  font-weight: bold;
  top: 10px;
  left: 10px;
  background-color: #40E0D0;
  color: white;
  border: none;
  padding: 3px 6px;
  font-size: 16px;
  cursor: pointer;
}

.reset-button:hover {
  background-color: #089d9d;
}
</style>

<style>
.leaflet-control-zoom {
  display: none;
}
</style>