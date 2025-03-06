<script setup>
import { onMounted } from 'vue';
import * as d3 from 'd3';

onMounted(() => {
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

  // Load the CSV file and normalize column names
  d3.csv('./data/INFORM_Risk_2025_v069_index.csv').then((data) => {
    // Normalize column names
    data = normalizeColumnNames(data);

    console.log("Normalized data:", data);  // Log normalized data

    const margin = { top: 20, right: 30, bottom: 40, left: 40 };
    const width = 500 - margin.left - margin.right;
    const height = 500 - margin.top - margin.bottom;

    // Create the SVG element for the bar chart
    const svg = d3.select("#chart")
      .attr("width", width + margin.left + margin.right)
      .attr("height", height + margin.top + margin.bottom)
      .append("g")
      .attr("transform", `translate(${margin.left},${margin.top})`);

    // Parse the 'inform_risk' and normalize it
    data.forEach(d => {
      d.inform_risk = +d.inform_risk;  // Parse as a number
      console.log(d.country, d.inform_risk);  // Log each data point
    });

    // Check if inform_risk max value is reasonable
    const maxRisk = d3.max(data, d => d.inform_risk);
    console.log("Max risk value:", maxRisk);  // Check max value for inform_risk

    const x = d3.scaleBand()
      .domain(data.map(d => d.country))  // Assuming 'COUNTRY' is now 'country'
      .range([0, width])
      .padding(0.1);

    const y = d3.scaleLinear()
      .domain([0, maxRisk])  // Use the max risk value for the y domain
      .nice()
      .range([height, 0]);

    // Create the bars (rectangles)
    svg.selectAll(".bar")
      .data(data)
      .enter().append("rect")
      .attr("class", "bar")
      .attr("x", d => {
        const xPosition = x(d.country);
        console.log("X Position: ", xPosition);
        return xPosition;  // Log X Position
      })
      .attr("y", d => {
        const yPosition = y(d.inform_risk);
        console.log("Y Position: ", yPosition);
        return yPosition;  // Log Y Position
      })
      .attr("width", x.bandwidth())
      .attr("height", d => {
        const barHeight = height - y(d.inform_risk);
        console.log("Bar Height: ", barHeight);
        return barHeight;  // Log Bar Height
      })
      .attr("fill", "steelblue")
      .attr("stroke", "black");  // Add a border to make the bars more visible

    // Add x-axis
    svg.append("g")
      .attr("class", "x-axis")
      .attr("transform", `translate(0,${height})`)
      .call(d3.axisBottom(x));

    // Add y-axis
    svg.append("g")
      .attr("class", "y-axis")
      .call(d3.axisLeft(y));
  }).catch((error) => {
    console.error("Error loading the CSV file:", error);
  });
});
</script>

<template>
  <div>
    <div>Hello from d3BarChart.vue, testing to see what's wrong with my chart.</div> <!-- Add this temporarily to check that things are populating -->
    <div id="chart"></div>
  </div>
</template>

<style scoped>
#chart {
  margin: auto;
  border: 1px solid #ccc;
  width: 100%;
  height: 500px;
  /* Add a fixed width if you have too many countries */
  max-width: 1200px;
}
.bar {
  transition: all 0.3s ease;
}
.bar:hover {
  fill: orange;
}
</style>
