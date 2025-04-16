<template>
  <div class="precip-wrapper bar-chart" ref="precipWrapperRef">
    <h3>Historical + Projected Precipitation</h3>
    <div v-for="column in filteredPrecipitationColumns" :key="column" class="bar-chart-block">
      <div class="bar-chart-content">
        <h4>{{ getDropdownTitle(column) }}</h4>
        <div class="legend">
          <svg :ref="el => legendRefs[column] = el" height="40"></svg>
        </div>
        <svg :ref="el => chartRefs[column] = el"></svg>
      </div>
    </div>
  </div>
</template>

<script setup>
import * as d3 from "d3";
import { ref, onMounted, nextTick, computed, watch } from "vue";

const precipWrapperRef = ref(null);
const chartRefs = {};
const legendRefs = {};
const historicalData = ref([]);
const projectedData = ref([]);
const projectedColumns = ref([]);

const filteredPrecipitationColumns = computed(() =>
  projectedColumns.value.filter(col =>
    col.toLowerCase().includes("precipitation") && col.includes("8.5")
  )
);

function getDropdownTitle(col) {
  return col.replace(/_/g, " ");
}

onMounted(() => {
  if (precipWrapperRef.value) {
    observer.observe(precipWrapperRef.value);
  }
});

// Intersection Observer setup to trigger animation
const observer = new IntersectionObserver(
  (entries) => {
    entries.forEach((entry) => {
      if (entry.isIntersecting) {
        // Trigger animation when the component is in view
        entry.target.classList.add("in-view");

        // Trigger the animation of the precipitation circles
        triggerPrecipitationAnimation();
      } else {
        // Optionally, remove animation class if it's out of view
        entry.target.classList.remove("in-view");
      }
    });
  },
  { threshold: 0.5 } // Trigger when 50% of the component is in view
);

function triggerPrecipitationAnimation() {
  filteredPrecipitationColumns.value.forEach(column => {
    // Retrieve the data for this column
    const historical = historicalData.value.map(d => ({
      year: +d.year,
      value: Math.round(parseFloat(d["Precipitation"])),
      source: "historical"
    })).filter(d => !isNaN(d.value));

    const projected = projectedData.value
      .filter(d => +d.year !== 2020)
      .map(d => ({
        year: +d.year,
        value: Math.round(parseFloat(d[column])),
        source: "projected"
      })).filter(d => !isNaN(d.value));

    const combinedRaw = [...historical, ...projected];
    const combinedData = getYearlySums(combinedRaw);

    // Calculate unit per ellipse
    const maxValue = d3.max(combinedData, d => d.value);
    const ellipseMinHeight = 14;
    const chartHeight = chartRefs[column]?.parentNode?.getBoundingClientRect().height || 400;
    const maxEllipsesVisible = Math.floor((chartHeight - 80) / ellipseMinHeight);
    const unitPerEllipse = maxValue / maxEllipsesVisible;

    // Render the precipitation circles
    renderPrecipitationCircles(combinedData, chartRefs[column], column, unitPerEllipse);
  });
}


function renderLegend(unitPerEllipse, svgEl) {
  const svg = d3.select(svgEl);
  svg.selectAll("*").remove();

  const dropSize = 10;
  const dropWidth = dropSize;
  const dropHeight = dropSize * 1.3;
  const centerX = 20;
  const centerY = 25;

  const path = d3.path();
  path.moveTo(0, -dropHeight / 2);
  path.bezierCurveTo(dropWidth / 2, -dropHeight / 2, dropWidth / 2, dropHeight / 4, 0, dropHeight / 2);
  path.bezierCurveTo(-dropWidth / 2, dropHeight / 4, -dropWidth / 2, -dropHeight / 2, 0, -dropHeight / 2);

  svg.append("path")
    .attr("d", path.toString())
    .attr("transform", `translate(${centerX}, ${centerY}) scale(1, -1)`)
    .attr("fill", "#089c9d");

  svg.append("text")
    .attr("x", centerX + 15)
    .attr("y", centerY + 5)
    .attr("fill", "#333")
    .attr("font-size", "12px")
    .text(`1 raindrop = ${Math.round(unitPerEllipse)} mm`);
}


function renderPrecipitationCircles(combinedData, svgEl, column, unitPerEllipse) {
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
  unitPerEllipse = maxValue / maxEllipsesVisible;

  const y = d3.scaleLinear().domain([0, maxEllipsesVisible]).range([innerHeight, 0]);
  const yValue = d3.scaleLinear().domain([0, maxValue]).range([innerHeight, 0]);

  const color = d3.scaleOrdinal().domain(["historical", "projected"]).range(["#089c9d", "#40E0D0"]);

  if (column === filteredPrecipitationColumns.value[0]) {
    renderLegend(unitPerEllipse, legendRefs[column]);
  }

  combinedData.forEach(d => {
    const totalCircles = Math.floor(d.value / unitPerEllipse);
    const barCenter = x(d.year) + x.bandwidth() / 2;
    const ellipseRx = Math.min(x.bandwidth() / 3, 12);

    for (let i = 0; i < totalCircles; i++) {
      const randomYOffset = Math.random() * 100 + 20;
      const randomDelay = Math.random() * 800;
      const dropHeight = ellipseRx * 2.5;
      const dropWidth = ellipseRx * 2;

      const dropPath = d3.path();
      dropPath.moveTo(0, -dropHeight / 2);
      dropPath.bezierCurveTo(dropWidth / 2, -dropHeight / 2, dropWidth / 2, dropHeight / 4, 0, dropHeight / 2);
      dropPath.bezierCurveTo(-dropWidth / 2, dropHeight / 4, -dropWidth / 2, -dropHeight / 2, 0, -dropHeight / 2);

      g.append("path")
        .attr("d", dropPath.toString())
        .attr("transform", `translate(${barCenter}, ${-randomYOffset}) scale(1, -1)`)
        .attr("fill", color(d.source))
        .transition()
        .delay(randomDelay)
        .duration(600)
        .ease(d3.easeBounceOut)
        .attr("transform", `translate(${barCenter}, ${y(i + 1)}) scale(1, -1)`);
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
      hoverBar.style("display", "block")
        .transition()
        .duration(500)
        .ease(d3.easeCubicOut)
        .attr("x", x(d.year))
        .attr("y", yValue(d.value))
        .attr("height", innerHeight - yValue(d.value))
        .attr("fill", hoverColor);

      g.selectAll(".bar-label").remove();
      g.append("text")
        .attr("class", "bar-label")
        .attr("x", x(d.year) + x.bandwidth() / 2)
        .attr("y", yValue(d.value) - 10)
        .attr("text-anchor", "middle")
        .attr("fill", "#333")
        .attr("opacity", 0)
        .text(`${Math.round(d.value)} mm`)
        .transition()
        .duration(500)
        .attr("opacity", 1);
    })
    .on("mouseout", function () {
      hoverBar.interrupt();
      hoverBar.transition()
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

watch([historicalData, projectedData], async () => {
  if (!historicalData.value.length || !projectedData.value.length) return;

  const histCol = "Precipitation";

  for (const column of filteredPrecipitationColumns.value) {
    const historical = historicalData.value.map(d => ({
      year: +d.year,
      value: Math.round(parseFloat(d[histCol])),
      source: "historical"
    })).filter(d => !isNaN(d.value));

    const projected = projectedData.value
      .filter(d => +d.year !== 2020)
      .map(d => ({
        year: +d.year,
        value: Math.round(parseFloat(d[column])),
        source: "projected"
      })).filter(d => !isNaN(d.value));

    const combinedRaw = [...historical, ...projected];
    const combinedData = getYearlySums(combinedRaw);

    await nextTick();

    const maxValue = d3.max(combinedData, d => d.value);
    const ellipseMinHeight = 14;
    const chartHeight = chartRefs[column]?.parentNode?.getBoundingClientRect().height || 400;
    const maxEllipsesVisible = Math.floor((chartHeight - 80) / ellipseMinHeight);
    const unitPerEllipse = maxValue / maxEllipsesVisible;

    renderPrecipitationCircles(combinedData, chartRefs[column], column, unitPerEllipse);
  }
});

onMounted(async () => {
  try {
    const hist = await d3.csv("./data/MeanMaxSurfaceAirTempuratureAndPrecipitation-1951-2020-Month.csv");
    const proj = await d3.csv("./data/AverageMeanAndMaxSurfaceAirTemperatureAndPrecipitation-2020-2099-Month-FourScenarios.csv");

    historicalData.value = hist;
    projectedData.value = proj;
    projectedColumns.value = Object.keys(proj[0] || {});
  } catch (err) {
    console.error("Error loading CSV data:", err);
  }
});
</script>

<style scoped>
html,
body,
#app {
  height: 100%;
  margin: 0;
  padding: 0;
}

.precip-wrapper {
  display: flex;
  flex-direction: column;
  height: 100vh;
  box-sizing: border-box;
  padding: 4rem 5rem 5rem 5rem;
  overflow: auto;  /* Allow scrolling */
  opacity: 0; /* Initially hidden */
  transition: opacity 1s ease-in-out;
}

.precip-wrapper.in-view {
  opacity: 1; /* Fade in when in view */
}

.bar-chart {
  flex: 1;
  display: flex;
  flex-direction: column;
}

.bar-chart-block {
  width: 100%;
  height: 100%;
  flex: 1;
  overflow: hidden; /* Prevent overflow of chart content */
}

.bar-chart-content {
  width: 100%;
  height: 100%;
  position: relative;
}

.legend {
  position: absolute;
  top: 2.5rem; /* Adjust as needed */
  left: 4.5rem;
  padding-left: 1rem;
  z-index: 2;
  pointer-events: none;
}

svg {
  overflow: visible;
}
</style>
