<template>
  <div class="mean-max-temperature historical-temperature">
    <h3>Historical Mean and Maximum Surface Air Temperature</h3>
    <!-- Radio buttons for selecting mean or max -->
    <div>
      <label>
        <input type="radio" v-model="selectedHistoricalColumn" value="mean" /> Mean
      </label>
      <label>
        <input type="radio" v-model="selectedHistoricalColumn" value="max" /> Max
      </label>
    </div>
    <!-- SVG container for the heatmap -->
    <svg ref="historicalSvg" width="800" height="500"></svg>
  </div>
  <div class="mean-max-temperature projected-temperature">
    <h3>Projected Mean and Maximum Surface Air Temperature</h3>
    <!-- Dropdown for selecting projected column -->
    <select v-if="projectedColumns.length > 0" v-model="selectedProjectedColumn">
      <option v-for="column in projectedColumns" :key="column" :value="column">{{ column }}</option>
    </select>
    <svg ref="projectedSvg"></svg>
  </div>
</template>

<script setup>
import * as d3 from "d3";
import { ref, onMounted, nextTick, watchEffect } from "vue";

// Reactive data references
const historicalData = ref([]);
const projectedData = ref([]);
const historicalColumns = ref([]);
const projectedColumns = ref([]);

// Default values for selected columns
const selectedHistoricalColumn = ref('mean');
const selectedProjectedColumn = ref('');

// References to the SVG elements
const historicalSvg = ref(null);

// Load data asynchronously
onMounted(async () => {
  try {
    // Load historical and projected data
    historicalData.value = await d3.csv("./data/MeanMaxSurfaceAirTempuratureAndPrecipitation-1951-2020-Month.csv");
    projectedData.value = await d3.csv("./data/AverageMeanAndMaxSurfaceAirTemperatureAndPrecipitation-2020-2099-Month-FourScenarios.csv");

    // Get the column names from the first row of each CSV file
    historicalColumns.value = Object.keys(historicalData.value[0] || {});
    projectedColumns.value = Object.keys(projectedData.value[0] || {});

    // After data load, ensure DOM is updated before rendering the heatmap
    nextTick(() => {
      renderHistoricalHeatmap();
    });

  } catch (error) {
    console.error("Error loading data:", error);
  }
});

// Watch for changes in selectedHistoricalColumn to update the heatmap
watchEffect(() => {
  renderHistoricalHeatmap();
});

// Function to render the historical heatmap
function renderHistoricalHeatmap() {
  if (!historicalSvg.value) return; // Ensure the SVG reference is not null

  // Select the SVG element
  const svg = d3.select(historicalSvg.value);
  svg.selectAll("*").remove(); // Clear the SVG before rendering the new heatmap

  // Prepare the data for the heatmap
  //const selectedColumn = selectedHistoricalColumn.value;
  
// Group the historical data by year and month
const groupedData = d3.group(
  historicalData.value, 
  d => d.year,   // Group by year
  d => d.month   // Group by month
);

 console.log("groupedData", groupedData); // Log the grouped data once

// Convert grouped data into a format that can be used in the heatmap
const heatmapData = [];
groupedData.forEach((yearData, year) => {
  yearData.forEach((value, month) => {
    heatmapData.push({ year, month, value });
  });
});

 console.log("heatmapData", heatmapData); // Log the final heatmap data

  // Define the margins and dimensions for the heatmap
  const margin = { top: 40, right: 20, bottom: 50, left: 60 };
  const width = +svg.attr("width") - margin.left - margin.right;
  const height = +svg.attr("height") - margin.top - margin.bottom;

  const years = Array.from(groupedData.keys()); // List of unique years
  const months = Array.from(new Set(historicalData.value.map(d => d.month))); // List of unique months

  // Set the scales for the axes
  const xScale = d3.scaleBand()
    .domain(years)
    .range([0, width])
    .padding(0.05);

  const yScale = d3.scaleBand()
    .domain(months)
    .range([height, 0])
    .padding(0.05);

  // Now that heatmapData is populated, define the color scale
  const colorScale = d3.scaleQuantize()
    .domain([d3.min(heatmapData, d => d.value), d3.max(heatmapData, d => d.value)])
    .range(["blue", "green", "yellow", "orange", "red", "purple"]);

  // Append a group element for the heatmap
  const g = svg.append("g")
    .attr("transform", `translate(${margin.left},${margin.top})`);

  // Add X axis
  g.append("g")
    .selectAll(".x-axis")
    .data(years)
    .enter()
    .append("text")
    .attr("class", "x-axis")
    .attr("x", d => xScale(d) + xScale.bandwidth() / 2)
    .attr("y", height + 40)
    .style("text-anchor", "middle")
    .text(d => d);

  // Add Y axis
  g.append("g")
    .selectAll(".y-axis")
    .data(months)
    .enter()
    .append("text")
    .attr("class", "y-axis")
    .attr("x", -50)
    .attr("y", d => yScale(d) + yScale.bandwidth() / 2)
    .style("text-anchor", "middle")
    .text(d => d);

  // Create the heatmap cells
  g.selectAll(".heatmap-cell")
    .data(heatmapData)
    .enter()
    .append("rect")
    .attr("class", "heatmap-cell")
    .attr("x", d => xScale(d.year)) // X position is determined by year
    .attr("y", d => yScale(d.month)) // Y position is determined by month
    .attr("width", xScale.bandwidth())
    .attr("height", yScale.bandwidth())
    .style("fill", d => colorScale(d.value)) // Use the color scale;
}


</script>

<style scoped>
.mean-max-temperature div {
  margin-bottom: 10px;
}

.mean-max-temperature label {
  margin-right: 15px;
}

.heatmap-cell {
  transition: fill 0.3s ease;
}

.x-axis {
  font-size: 12px;
}

.y-axis {
  font-size: 12px;
}
</style>
