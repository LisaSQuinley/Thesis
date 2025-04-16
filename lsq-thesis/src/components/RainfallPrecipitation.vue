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
    .attr("width", width)
    .attr("height", height)
    .attr("viewBox", `0 0 ${width} ${height}`);

  svg.selectAll("*").remove();

  const margin = { top: 20, right: 20, bottom: 60, left: 60 };
  const innerWidth = width - margin.left - margin.right;
  const innerHeight = height - margin.top - margin.bottom;

  const g = svg.append("g").attr("transform", `translate(${margin.left},${margin.top})`);

  const years = Array.from(new Set(combinedData.map(d => d.year))).sort((a, b) => a - b);
  const x = d3.scaleBand().domain(years).range([0, innerWidth]).padding(0.1);

  const ellipseMinHeight = 14;
  const maxEllipsesVisible = Math.floor(innerHeight / ellipseMinHeight);
  const maxValue = d3.max(combinedData, d => d.value);

  const unitPerEllipse = maxValue / maxEllipsesVisible;

  const y = d3.scaleLinear()
    .domain([0, maxEllipsesVisible])
    .range([innerHeight, 0]);

  const yValue = d3.scaleLinear()
    .domain([0, maxValue])
    .range([innerHeight, 0]);

  const color = d3.scaleOrdinal()
    .domain(["historical", "projected"])
    .range(["#089c9d", "#40E0D0"]);

  combinedData.forEach(d => {
    const totalCircles = Math.floor(d.value / unitPerEllipse); // This rounds down to ensure it's a whole number
    const barCenter = x(d.year) + x.bandwidth() / 2;
    const ellipseRx = Math.min(x.bandwidth() / 3, 12);
    const ellipseRy = ellipseRx;

    for (let i = 0; i < totalCircles; i++) {
      const randomYOffset = Math.random() * 100 + 20;
      const randomDelay = Math.random() * 800;

      g.append("ellipse")
        .attr("cx", barCenter)
        .attr("cy", -randomYOffset)
        .attr("rx", ellipseRx)
        .attr("ry", ellipseRy)
        .attr("fill", color(d.source))
        .transition()
        .delay(randomDelay)
        .duration(600)
        .ease(d3.easeBounceOut)
        .attr("cy", y(i + 1));
    }
  });

  g.append("g")
    .attr("transform", `translate(0,${innerHeight})`)
    .call(d3.axisBottom(x).tickValues(years.filter(y => y % 10 === 0)))
    .call(g => g.select("path").remove());

  g.append("g")
    .call(d3.axisLeft(yValue).ticks(5).tickFormat(d => d.toFixed(0)))
    .call(g => {
      g.select("path").remove();
      g.selectAll(".tick").filter((d, i) => i === 0).remove();
    });

  const hoverBar = g.append("rect")
    .attr("class", "hover-bar")
    .attr("width", x.bandwidth())
    .attr("fill-opacity", 0.7)
    .style("display", "none")
    .attr("y", innerHeight)
    .attr("height", 0);

  g.selectAll(".hover-zone")
    .data(combinedData)
    .enter()
    .append("rect")
    .attr("class", "hover-zone")
    .attr("x", d => x(d.year))
    .attr("y", 0)
    .attr("width", x.bandwidth())
    .attr("height", innerHeight)
    .attr("fill", "transparent")
    .on("mouseover", function (event, d) {
      hoverBar.interrupt();

      const hoverColor = color(d.source);

      hoverBar
        .style("display", "block")
        .transition()
        .duration(500)
        .ease(d3.easeCubicOut)
        .attr("x", x(d.year))
        .attr("y", yValue(d.value))
        .attr("height", innerHeight - yValue(d.value))
        .attr("fill", hoverColor);

      // Remove existing labels
      g.selectAll(".bar-label").remove();

      // Add new label
      g.append("text")
        .attr("class", "bar-label")
        .attr("x", x(d.year) + x.bandwidth() / 2)
        .attr("y", yValue(d.value) - 10)
        .attr("text-anchor", "middle")
        .attr("fill", "#333")
        .attr("opacity", 0)
        .text(d.value.toFixed(1))
        .transition()
        .duration(500)
        .attr("opacity", 1);
    })
    .on("mouseout", function () {
      hoverBar.interrupt();

      hoverBar
        .transition()
        .duration(500)
        .ease(d3.easeCubicOut)
        .attr("y", innerHeight)
        .attr("height", 0)
        .style("display", "none");

      g.selectAll(".bar-label")
        .transition()
        .duration(300)
        .attr("opacity", 0)
        .remove();
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
html, body, #app {
  height: 100%;
  margin: 0;
  padding: 0;
}

.rainfaill-precipitation {
  display: flex;
  justify-content: center;
  align-items: center;
  text-align: center;
  width: 100%;
  height: 100%;
}

.precip-wrapper {
  display: flex;
  flex-direction: column;
  height: 100vh; /* Full viewport height */
  box-sizing: border-box;
  padding: 1rem;
}

.bar-chart {
  flex: 1;
  display: flex;
  flex-direction: column;
}


.bar-chart-block {
  width: 100%;
  height: 100%;
}

.bar-chart-content {
  width: 100%;
  height: 100%; /* or use a percentage based on parent */
}


svg {
  width: 100%;
  height: 100%;
}

h3 {
  width: 100%;
}

</style>
