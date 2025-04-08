<template>
    <div class="temperature-wrapper">
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
        <svg ref="historicalSvg"></svg>
      </div>
      <div class="mean-max-temperature projected-temperature">
        <h3>Projected Mean and Maximum Surface Air Temperature</h3>
        <select v-if="projectedColumns.length > 0" v-model="selectedProjectedColumn">
          <option v-for="column in projectedColumns" :key="column" :value="column">{{ column }}</option>
        </select>
        <svg ref="projectedSvg"></svg>
      </div>
    </div>
  </template>
  

  <script setup>
  import * as d3 from "d3";
  import { ref, onMounted, onBeforeUnmount, nextTick, watchEffect } from "vue";
  
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
  
  // Function to render the historical heatmap
  function renderHistoricalHeatmap() {
    if (!historicalSvg.value) return; // Ensure the SVG reference is not null
  
    // Determine dynamic sizing from the container
    const container = historicalSvg.value.parentNode;
    const containerWidth = container.getBoundingClientRect().width;
    // You can choose a fixed height or derive it similarly
    const containerHeight = containerWidth * 0.5; // Example: 50% of container's width
  
    // Select the SVG element and update its dimensions
    const svg = d3.select(historicalSvg.value)
      .attr("width", containerWidth)
      .attr("height", containerHeight)
      .attr("viewBox", `0 0 ${containerWidth} ${containerHeight}`); // ensures scaling
  
    svg.selectAll("*").remove(); // Clear the SVG before rendering the new heatmap
  
    // Prepare the data for the heatmap
    const groupedData = d3.group(
      historicalData.value, 
      d => d.year,   // Group by year
      d => d.month   // Group by month
    );
  
    console.log("groupedData", groupedData); // Log the grouped data once
  
    // Convert grouped data into a format that can be used in the heatmap
    const heatmapData = [];
    groupedData.forEach((yearData, year) => {
      yearData.forEach((values, month) => {
        const firstEntry = values[0]; // Get the first entry for that year-month
        const selectedColumn =
          selectedHistoricalColumn.value === "mean" ? "Mean Temp" : "Max Temp";
        const tempValue = parseFloat(firstEntry[selectedColumn]);
        if (!isNaN(tempValue)) {
          heatmapData.push({ year, month, value: tempValue });
        }
      });
    });
  
    console.log("heatmapData", heatmapData); // Log the final heatmap data
  
    // Define the margins and dimensions for the heatmap
    const margin = { top: 10, right: 20, bottom: 50, left: 60 };
    const width = containerWidth - margin.left - margin.right;
    const height = containerHeight - margin.top - margin.bottom;
  
    const years = Array.from(groupedData.keys()); // List of unique years
    const months = Array.from(new Set(historicalData.value.map(d => d.month))).reverse();
  
    // Set the scales for the axes
    const xScale = d3.scaleBand()
      .domain(years)
      .range([0, width])
      .padding(0.05);
  
    const yScale = d3.scaleBand()
      .domain(months)
      .range([height, 0])
      .padding(0.05);
  
    // Define the color scale for the heatmap
    const colorScale = d3.scaleQuantize()
      .domain([d3.min(heatmapData, d => d.value), d3.max(heatmapData, d => d.value)])
      .range(["#40E0D0", "#7FFF00", "#FFFF00", "#FF8000", "#FF0000", "#820747"]);
  
    // Append a group element for the heatmap
    const g = svg.append("g")
      .attr("transform", `translate(${margin.left},${margin.top})`);
  
    // Add X axis labels
    g.append("g")
  .selectAll(".x-axis")
  .data(years)
  .enter()
  .append("text")
  .attr("class", "x-axis")
  .attr("x", d => xScale(d) + xScale.bandwidth() / 1 + 30)
  .attr("y", height + 47)
  .attr("transform", d => `rotate(-90, ${xScale(d) + xScale.bandwidth() / 2}, ${height + 40})`)
  .style("text-anchor", "end")
  .text(d => d);
  
    // Add Y axis labels
    g.append("g")
      .selectAll(".y-axis")
      .data(months)
      .enter()
      .append("text")
      .attr("class", "y-axis")
      .attr("x", -25)
      .attr("y", d => yScale(d) + yScale.bandwidth() / 1.25)
      .style("text-anchor", "middle")
      .text(d => d);
  
// Helper maps for calculating delays
const monthOrder = new Map(months.map((m, i) => [m, i]));
const yearOrder = new Map(years.map((y, i) => [y, i]));

const getDelay = d => {
  const yIndex = yearOrder.get(d.year);
  const mIndex = monthOrder.get(d.month);
  return (yIndex * months.length + mIndex) * 10; // You can adjust `10` for faster/slower stagger
};

// Bind data with a key function
const cells = g.selectAll(".heatmap-cell")
  .data(heatmapData, d => `${d.year}-${d.month}`);

// ENTER phase
cells.enter()
  .append("rect")
  .attr("class", "heatmap-cell")
  .attr("x", d => xScale(d.year))
  .attr("y", d => yScale(d.month))
  .attr("width", xScale.bandwidth())
  .attr("height", yScale.bandwidth())
  .style("fill", d => colorScale(d.value))
  .style("opacity", 0)
  .transition()
  .delay(d => getDelay(d))
  .duration(500)
  .style("opacity", 1);

// UPDATE phase
cells.transition()
  .delay(d => getDelay(d))
  .duration(500)
  .style("fill", d => colorScale(d.value));

// EXIT phase
cells.exit()
  .transition()
  .delay(d => getDelay(d))
  .duration(300)
  .style("opacity", 0)
  .remove();


  }
  
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
      await nextTick();
      renderHistoricalHeatmap();
  
      // Set up the window resize listener to redraw the chart dynamically
      window.addEventListener("resize", renderHistoricalHeatmap);
    } catch (error) {
      console.error("Error loading data:", error);
    }
  });
  
  // Watch for changes in selectedHistoricalColumn to update the heatmap
  watchEffect(() => {
    renderHistoricalHeatmap();
  });
  
  // Clean up the resize listener when the component is unmounted
  onBeforeUnmount(() => {
    window.removeEventListener("resize", renderHistoricalHeatmap);
  });
  </script>
  
<style scoped>
.temperature-wrapper {
  display: flex;
  justify-content: space-between;
  gap: 20px;
}

.mean-max-temperature {
  flex: 1; /* Let each column grow/shrink equally */
  max-width: 50%;
}

.historical-temperature,
.projected-temperature {
  width: 100%; /* Keep full width inside their containers */
}

/* Optional: Adjust SVG sizes responsively */
svg {
  width: 100%;
  height: auto;
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
