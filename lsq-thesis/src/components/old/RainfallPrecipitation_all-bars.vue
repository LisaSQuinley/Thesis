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
    col.toLowerCase().includes("precipitation")
  )
);

function renderPrecipitationBarChart(combinedData, svgEl) {
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
  const y = d3.scaleLinear()
    .domain([0, 650])
    .nice()
    .range([innerHeight, 0]);

  const color = d3.scaleOrdinal()
    .domain(["historical", "projected"])
    .range(["#1f77b4", "#ff7f0e"]);

  g.selectAll("rect")
    .data(combinedData)
    .enter()
    .append("rect")
    .attr("x", d => x(d.year))
    .attr("y", d => y(d.value))
    .attr("width", x.bandwidth())
    .attr("height", d => innerHeight - y(d.value))
    .attr("fill", d => color(d.source));

  // Axes
  g.append("g")
    .attr("transform", `translate(0,${innerHeight})`)
    .call(d3.axisBottom(x).tickValues(years.filter(y => y % 10 === 0)));

  g.append("g").call(d3.axisLeft(y));

// Add value labels on top of summed bars
g.selectAll("text.bar-label")
  .data(combinedData)
  .enter()
  .append("text")
  .attr("class", "bar-label")
  .attr("x", d => x(d.year) + x.bandwidth() / 2)
  .attr("y", d => y(d.value) - 5)
  .attr("text-anchor", "middle")
  .attr("font-size", "10px")
  .attr("fill", "#333")
  .text(d => d.value.toFixed(1));


    // Filter only projected data
    const projectedOnly = combinedData.filter(d => d.source === "projected");

if (projectedOnly.length > 1) {
  const xVals = projectedOnly.map(d => d.year);
  const yVals = projectedOnly.map(d => d.value);

  // Calculate linear regression
  const n = xVals.length;
  const sumX = d3.sum(xVals);
  const sumY = d3.sum(yVals);
  const sumXY = d3.sum(xVals.map((x, i) => x * yVals[i]));
  const sumX2 = d3.sum(xVals.map(x => x * x));

  const slope = (n * sumXY - sumX * sumY) / (n * sumX2 - sumX * sumX);
  const intercept = (sumY - slope * sumX) / n;

  // Create line points from first to last year
  const trendLineData = [
    { year: d3.min(xVals), value: slope * d3.min(xVals) + intercept },
    { year: d3.max(xVals), value: slope * d3.max(xVals) + intercept }
  ];

  // Add trend line
  g.append("line")
    .attr("x1", x(trendLineData[0].year) + x.bandwidth() / 2)
    .attr("y1", y(trendLineData[0].value))
    .attr("x2", x(trendLineData[1].year) + x.bandwidth() / 2)
    .attr("y2", y(trendLineData[1].value))
    .attr("stroke", "red")
    .attr("stroke-width", 2)
    .attr("stroke-dasharray", "4 2");
}
  
}

function getDropdownTitle(col) {
  return col.replace(/_/g, " ");
}

watch([historicalData, projectedData], async () => {
  if (!historicalData.value.length || !projectedData.value.length) return;

  const histCol = "Precipitation";

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
  renderPrecipitationBarChart(combinedData, chartRefs[column]);
}


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

}, { immediate: true });

onMounted(async () => {
  try {
    const hist = await d3.csv("./data/MeanMaxSurfaceAirTempuratureAndPrecipitation-1951-2020-Month.csv");
    const proj = await d3.csv("./data/AverageMeanAndMaxSurfaceAirTemperatureAndPrecipitation-2020-2099-Month-FourScenarios.csv");

    historicalData.value = hist;
    projectedData.value = proj;
    projectedColumns.value = Object.keys(proj[0] || {});

    window.addEventListener("resize", () => {
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
        renderPrecipitationBarChart(combinedData, chartRefs[column]);
      }
    });
  } catch (err) {
    console.error("Error loading precipitation data:", err);
  }
});

</script>


<!-- Add "scoped" attribute to limit CSS to this component only -->
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
  height: 300px; /* set a fixed height or adjust dynamically */
}

svg {
  width: 100%;
  height: 100%;
}


h3 {
width: 100%;
}
</style>
