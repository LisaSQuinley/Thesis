<template>
  <div class="precip-wrapper bar-chart">
    <h3>Historical + Projected Precipitation</h3>

    <div v-for="column in filteredPrecipitationColumns" :key="column" class="bar-chart-block">
      <div class="bar-chart-content">
        <h4>{{ getDropdownTitle(column) }}</h4>
        <svg :ref="el => chartRefs[column] = el"></svg>
      </div>
    </div>
  </div>
</template>

<script setup>
import * as d3 from "d3";
import { ref, onMounted, nextTick, computed, watch } from "vue";

const chartRefs = {};
const historicalData = ref([]);
const projectedData = ref([]);
const projectedColumns = ref([]);

const filteredPrecipitationColumns = computed(() =>
  projectedColumns.value.filter(col =>
    col.toLowerCase().includes("precipitation") && col.includes("8.5") // Only show 8.5 scenario columns
  )
);

function renderPrecipitationCircles(combinedData, svgEl) {
  if (!svgEl || !combinedData.length) return;

  const container = svgEl.parentNode;
  const width = container.getBoundingClientRect().width;
  const height = container.getBoundingClientRect().height;

  const svg = d3.select(svgEl)
    .attr("width", width)  // Update width based on container size
    .attr("height", height) // Update height based on container size
    .attr("viewBox", `0 0 ${width} ${height}`);
  
  svg.selectAll("*").remove(); // Clear previous chart

  const margin = { top: 20, right: 20, bottom: 60, left: 60 };
  const innerWidth = width - margin.left - margin.right;
  const innerHeight = height - margin.top - margin.bottom;

  const g = svg.append("g").attr("transform", `translate(${margin.left},${margin.top})`);

  const years = Array.from(new Set(combinedData.map(d => d.year))).sort((a, b) => a - b);
  const x = d3.scaleBand().domain(years).range([0, innerWidth]).padding(0.1);

  const maxValue = d3.max(combinedData, d => d.value);
  const maxCircles = Math.ceil(maxValue / 20);

  const y = d3.scaleLinear()
    .domain([0, maxCircles])
    .range([innerHeight, 0]);

    const yValue = d3.scaleLinear()
  .domain([0, maxValue])
  .range([innerHeight, 0]);


  const color = d3.scaleOrdinal()
    .domain(["historical", "projected"])
    .range(["#089c9d", "#40E0D0"]);

  combinedData.forEach(d => {
    const totalCircles = Math.floor(d.value / 20);
    const barCenter = x(d.year) + x.bandwidth() / 2;
    const ellipseRx = Math.min(x.bandwidth() / 3, 12);
    const ellipseRy = ellipseRx;

    for (let i = 0; i < totalCircles; i++) {
      g.append("ellipse")
        .attr("cx", barCenter)
        .attr("cy", y(i + 1))
        .attr("rx", ellipseRx)
        .attr("ry", ellipseRy)
        .attr("fill", color(d.source));
    }
  });

  g.append("g")
  .attr("transform", `translate(0,${innerHeight})`)
  .call(d3.axisBottom(x).tickValues(years.filter(y => y % 10 === 0)))
  .call(g => g.select("path").remove()); // Remove the horizontal axis line

  g.append("g")
  .call(d3.axisLeft(yValue).ticks(5).tickFormat(d => d.toFixed(0)))
  .call(g => {
    g.select("path").remove(); // remove axis line
    g.selectAll(".tick").filter((d, i) => i === 0).remove(); // remove first tick
  });


// Add value labels and hide them initially
g.selectAll(".bar-label")
  .data(combinedData)
  .enter()
  .append("text")
  .attr("class", "bar-label")
  .attr("x", d => x(d.year) + x.bandwidth() / 2)  // Center the label on the bar
  .attr("y", d => y(Math.floor(d.value / 20)) - 10)  // Position the label above the ellipses
  .attr("text-anchor", "middle")
  .attr("font-size", "10px")
  .attr("fill", "#333")
  .attr("opacity", 0) // Ensure labels are initially hidden
  .text(d => d.value.toFixed(1));

// Create the hover bar (start it off as hidden and position it dynamically later)
const hoverBar = g.append("rect")
  .attr("class", "hover-bar")
  .attr("width", x.bandwidth())
  .attr("fill-opacity", 0.7)
  .style("display", "none")  // Initially hidden
  .attr("y", innerHeight)   // Start at the bottom
  .attr("height", 0);       // Start with no height

// Add hover effect with animation
g.selectAll(".hover-zone")
  .data(combinedData)
  .enter()
  .append("rect")
  .attr("class", "hover-zone")
  .attr("x", d => x(d.year))  // Correct x-position based on the year
  .attr("y", 0)
  .attr("width", x.bandwidth())
  .attr("height", innerHeight)
  .attr("fill", "transparent")
  .on("mouseover", function (event, d) {
    // Cancel any ongoing transitions before starting a new one
    hoverBar.interrupt();

    // Determine the color based on the data source
    const hoverColor = color(d.source); // "historical" or "projected"

    // Show and animate the hover bar
    hoverBar
      .style("display", "block") // Make the bar visible
      .transition()  // Start a transition
      .duration(500) // Duration of the animation (500ms)
      .ease(d3.easeCubicOut) // Ease function for smoothness
      .attr("x", x(d.year)) // Ensure hover bar is at the correct x (year position)
      .attr("y", yValue(d.value)) // Animate to the top position based on value
      .attr("height", innerHeight - yValue(d.value)) // Animate to the correct height
      .attr("fill", hoverColor); // Set the hover bar's color to match ellipses

    // Show the value label for the hovered bar
    g.selectAll(".bar-label")
      .filter(label => label.year === d.year)  // Find the label for the current year
      .transition()
      .duration(500)
      .attr("opacity", 1); // Make the label visible
  })
  .on("mouseout", function () {
    // Cancel any ongoing transitions before starting a new one
    hoverBar.interrupt();

    // Hide and reset the hover bar
    hoverBar
      .transition() // Start a transition
      .duration(500) // Duration of the animation (500ms)
      .ease(d3.easeCubicOut) // Ease function for smoothness
      .attr("y", innerHeight) // Move the bar back to the bottom
      .attr("height", 0) // Shrink the bar to zero height
      .style("display", "none"); // Hide the bar after the transition

    // Hide the value label for the bar after hover
    g.selectAll(".bar-label")
      .transition()
      .duration(500)
      .attr("opacity", 0); // Make the label invisible
  });


}



function getDropdownTitle(col) {
  return col.replace(/_/g, " ");
}

watch([historicalData, projectedData], async () => {
  if (!historicalData.value.length || !projectedData.value.length) return;

  const histCol = "Precipitation";

  // Loop through the filtered columns (only the 8.5 scenario)
  for (const column of filteredPrecipitationColumns.value) {
    const historical = historicalData.value.map(d => ({
      year: +d.year,
      value: parseFloat(d[histCol]),
      source: "historical"
    })).filter(d => !isNaN(d.value));

    const projected = projectedData.value
      .filter(d => +d.year !== 2020)
      .map(d => ({
        year: +d.year,
        value: parseFloat(d[column]),
        source: "projected"
      })).filter(d => !isNaN(d.value));

    const combinedRaw = [...historical, ...projected];
    const combinedData = getYearlySums(combinedRaw);

    await nextTick();
    renderPrecipitationCircles(combinedData, chartRefs[column]);
  }
});

function getYearlySums(data) {
  const yearlySums = d3.rollups(
    data,
    values => d3.sum(values, d => d.value),
    d => d.year
  ).map(([year, sum]) => ({
    year,
    value: sum,
    source: data.find(d => d.year === year).source
  }));

  return yearlySums;
}

onMounted(async () => {
  try {
    const hist = await d3.csv("./data/MeanMaxSurfaceAirTempuratureAndPrecipitation-1951-2020-Month.csv");
    const proj = await d3.csv("./data/AverageMeanAndMaxSurfaceAirTemperatureAndPrecipitation-2020-2099-Month-FourScenarios.csv");

    historicalData.value = hist;
    projectedData.value = proj;
    projectedColumns.value = Object.keys(proj[0] || {});

    let resizeTimeout;
window.addEventListener("resize", () => {
  clearTimeout(resizeTimeout);
  resizeTimeout = setTimeout(() => {
    for (const column of filteredPrecipitationColumns.value) {
      const combinedData = [
        ...historicalData.value.map(d => ({
          year: +d.year,
          value: parseFloat(d["Precipitation"]),
          source: "historical"
        })).filter(d => !isNaN(d.value)),

        ...projectedData.value.map(d => ({
          year: +d.year,
          value: parseFloat(d[column]),
          source: "projected"
        })).filter(d => !isNaN(d.value))
      ];
      renderPrecipitationCircles(combinedData, chartRefs[column]);
    }
  }, 200); // Adjust delay for performance (e.g., 200ms)
});
  } catch (err) {
    console.error("Error loading CSV data:", err);
  }
});
</script>

<style scoped>
.rainfaill-precipitation {
  display: flex;
  justify-content: center;
  align-items: center;
  text-align: center;
  width: 100%;
}

.bar-chart {
  display: flex;
  flex-direction: column;
  width: 100%;
}

.bar-chart-block {
  width: 100%;
}

.bar-chart-content {
  width: 100%;
  height: 300px;
  /* set a fixed height or adjust dynamically */
}

svg {
  width: 100%;
  height: 100%;
}

h3 {
  width: 100%;
}
</style>
