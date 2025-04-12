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
    .domain([0, d3.max(combinedData, d => d.value)])
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
}

function getDropdownTitle(col) {
  return col.replace(/_/g, " ");
}

watch([historicalData, projectedData], async () => {
  if (!historicalData.value.length || !projectedData.value.length) return;

  const histCol = "Precipitation"; // Adjust based on actual column name in historical dataset

  for (const column of filteredPrecipitationColumns.value) {
    const combinedData = [
      ...historicalData.value.map(d => ({
        year: +d.year,
        value: parseFloat(d[histCol]),
        source: "historical"
      })).filter(d => !isNaN(d.value)),

      ...projectedData.value.map(d => ({
        year: +d.year,
        value: parseFloat(d[column]),
        source: "projected"
      })).filter(d => !isNaN(d.value))
    ];

    await nextTick();
    renderPrecipitationBarChart(combinedData, chartRefs[column]);
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
  margin-bottom: 2rem;
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
