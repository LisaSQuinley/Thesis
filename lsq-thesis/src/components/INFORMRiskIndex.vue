<script setup>
import { onMounted, ref, computed, watch } from 'vue';
import * as d3 from 'd3';

// Reactive state for sorting
const chartData = ref([]);
const sortOption = ref('descending'); // Default sort: Decreasing risk index

// Function to sort data based on selected option
const sortData = (data, option) => {
  if (option === 'alphabetical') {
    return [...data].sort((a, b) => a.country.localeCompare(b.country));
  } else if (option === 'ascending') {
    return [...data].sort((a, b) => a.inform_risk - b.inform_risk);
  } else { // descending
    return [...data].sort((a, b) => b.inform_risk - a.inform_risk);
  }
};

// Computed property for sorted data
const sortedData = computed(() => {
  if (chartData.value.length === 0) return [];
  return sortData(chartData.value, sortOption.value);
});

// Normalize all keys (column names) in the CSV data
const normalizeColumnNames = (data) => {
  return data.map(row => {
    const normalizedRow = {};
    for (const [key, value] of Object.entries(row)) {
      // Convert column names to lowercase and remove spaces
      const normalizedKey = key.toLowerCase().replace(/\s+/g, '_');
      normalizedRow[normalizedKey] = value;
    }
    return normalizedRow;
  });
};

const createChart = (data) => {
  // Clear existing chart
  d3.select("#chart").select("svg").remove();
  
  const margin = { top: 40, right: 30, bottom: 40, left: 150 };
  const width = 500 - margin.left - margin.right;
  
  // Calculate height based on number of countries
  const barHeight = 25; // Height per country
  const svgHeight = data.length * barHeight + margin.top + margin.bottom;

  // Create the SVG element for the bar chart
  const svg = d3.select("#chart")
    .append("svg")
    .attr("width", width + margin.left + margin.right)
    .attr("height", svgHeight)
    .append("g")
    .attr("transform", `translate(${margin.left},${margin.top})`);

  // Check if inform_risk max value is reasonable
  const maxRisk = d3.max(data, d => d.inform_risk);
  console.log("Max risk value:", maxRisk);  // Check max value for inform_risk

  // Define a color scale for risk classes
  const colorScale = d3.scaleOrdinal()
    .domain(['Very Low', 'Low', 'Medium', 'High', 'Very High'])
    .range(['#00FF00', '#7FFF00', '#FFFF00', '#FF8000', '#FF0000']); // Green, Yellow-Green, Yellow, Orange, Red

  // Horizontal bar chart scales (swapped from vertical)
  const x = d3.scaleLinear()
    .domain([0, maxRisk])
    .nice()
    .range([0, width]);

  const y = d3.scaleBand()
    .domain(data.map(d => d.country))
    .range([0, svgHeight - margin.top - margin.bottom])
    .padding(0.1);

  // Create the bars (rectangles) - horizontal orientation
  svg.selectAll(".bar")
    .data(data)
    .enter().append("rect")
    .attr("class", "bar")
    .attr("x", 0)
    .attr("y", d => y(d.country))
    .attr("width", d => x(d.inform_risk))
    .attr("height", y.bandwidth())
    .attr("fill", d => colorScale(d.risk_class)); // Apply color based on risk class

  // Add x-axis (values)
  svg.append("g")
    .attr("class", "x-axis")
    .call(d3.axisTop(x));

  // Add y-axis (countries)
  svg.append("g")
    .attr("class", "y-axis")
    .call(d3.axisLeft(y));

  // Add chart title
  svg.append("text")
    .attr("x", width / 2)
    .attr("y", -margin.top / 2)
    .attr("text-anchor", "middle")
    .attr("font-weight", "bold")
    .text("INFORM Risk Index by Country");
};

onMounted(() => {
  // Load the CSV file and normalize column names
  d3.csv('./data/INFORM_Risk_2025_v069_index.csv').then((data) => {
    // Normalize column names
    data = normalizeColumnNames(data);
    console.log("Normalized data:", data);  // Log normalized data

    // Parse the 'inform_risk' and normalize it
    data.forEach(d => {
      d.inform_risk = +d.inform_risk;  // Parse as a number
      console.log(d.country, d.inform_risk);  // Log each data point
    });

    // Store the processed data in reactive state
    chartData.value = data;
  }).catch((error) => {
    console.error("Error loading the CSV file:", error);
  });
});

// Watch for changes to sorted data to update the chart
watch(sortedData, (newData) => {
  if (newData.length > 0) {
    createChart(newData);
  }
}, { immediate: true });
</script>

<template>
  <div>
    <!-- Filter controls -->
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
    
    <div id="chart"></div>
  </div>
</template>

<style scoped>
#chart {
  margin: auto;
  width: 100%;
  height: 650px;
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
