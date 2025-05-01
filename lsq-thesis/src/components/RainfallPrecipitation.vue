<template>
    <div class="precip-wrapper bar-chart" ref="precipWrapperRef">
      <h3>Raincheck<br>Forever</h3>
    <div v-for="column in filteredPrecipitationColumns" :key="column" class="bar-chart-block">
      <div class="bar-chart-content">
        <!--         <h4>{{ getDropdownTitle(column) }}</h4> -->
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

/* 
function getDropdownTitle(col) {
  return col.replace(/_/g, " ");
}
 */

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
        entry.target.classList.add("in-view");
        triggerPrecipitationAnimation();
      } else {
        entry.target.classList.remove("in-view");
      }
    });
  },
  { threshold: 0.5 }
);

function getUnitPerEllipse() {
  const { innerWidth: width, innerHeight: height } = window;
  return height > width ? 5 : 10;
}


function triggerPrecipitationAnimation() {
  const unitPerEllipse = getUnitPerEllipse(); // 👈 New line

  filteredPrecipitationColumns.value.forEach(column => {
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

    renderPrecipitationCircles(combinedData, chartRefs[column], column, unitPerEllipse);
  });
}

function renderLegend(unitPerEllipse, svgEl) {
  const svg = d3.select(svgEl);
  svg.selectAll("*").remove();

  const dropSize = 10;
  const dropWidth = dropSize;
  const dropHeight = dropSize * 1.3;

  const colorMap = {
    historical: "#089c9d",
    projected: "#40E0D0",
    regression: "url(#regressionGradient)"
  };

  const dropPath = d3.path();
  dropPath.moveTo(0, -dropHeight / 2);
  dropPath.bezierCurveTo(dropWidth / 2, -dropHeight / 2, dropWidth / 2, dropHeight / 4, 0, dropHeight / 2);
  dropPath.bezierCurveTo(-dropWidth / 2, dropHeight / 4, -dropWidth / 2, -dropHeight / 2, 0, -dropHeight / 2);

  const legendItems = ["historical", "projected", "regression"];

  legendItems.forEach((type, i) => {
    const centerX = 20;
    const centerY = 20 + i * 25;

    if (type === "regression") {
      // Add regression line
      svg.append("line")
        .attr("x1", centerX - 5)
        .attr("x2", centerX + 15)
        .attr("y1", centerY)
        .attr("y2", centerY)
        .attr("stroke", "#5C4033")
        .attr("stroke-width", 2); 

      svg.append("text")
        .attr("x", centerX + 20)
        .attr("y", centerY + 5)
        .attr("fill", "#2c3e50")
        .attr("font-size", "1em")
        .text("Downward Trend");
    } else {
      // Add raindrop legend
      svg.append("path")
        .attr("d", dropPath.toString())
        .attr("transform", `translate(${centerX}, ${centerY}) scale(1, -1)`)
        .attr("fill", colorMap[type]);

      svg.append("text")
        .attr("x", centerX + 15)
        .attr("y", centerY + 5)
        .attr("fill", "#2c3e50")
        .attr("font-size", "1em")
        .text(`${type === "historical" ? "Past" : "Future"}: 1 raindrop = ${Math.round(unitPerEllipse)} mm`);
    }
  });

  svg.attr("width", 250).attr("height", 90);
}

function linearRegression(x, y) {
  const xMean = d3.mean(x);
  const yMean = d3.mean(y);
  const slope = d3.sum(x.map((d, i) => (d - xMean) * (y[i] - yMean))) / d3.sum(x.map(d => (d - xMean) ** 2));
  const intercept = yMean - slope * xMean;

  return {
    slope,
    intercept,
    predict: x => intercept + slope * x
  };
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
  const xScale = d3.scaleBand().domain(years).range([0, innerWidth]).padding(0.1);
  const maxValue = d3.max(combinedData, d => d.value);
  const yScale = d3.scaleLinear().domain([0, maxValue]).range([innerHeight, 0]);
  const color = d3.scaleOrdinal().domain(["historical", "projected"]).range(["#089c9d", "#40E0D0"]);

  // Compute linear regression
  const regression = linearRegression(combinedData.map(d => d.year), combinedData.map(d => d.value));

  // Create a line path
  const line = d3.line()
    .x(d => xScale(d.year) + xScale.bandwidth() / 2)
    .y(d => yScale(regression.predict(d.year)));

  const regressionData = [
    { year: d3.min(combinedData, d => d.year) },
    { year: d3.max(combinedData, d => d.year) }
  ];

  // 1. Append a gradient to the SVG
  const defs = svg.append("defs");

  const gradient = defs.append("linearGradient")
    .attr("id", "regressionGradient")
    .attr("x1", "0%")
    .attr("x2", "100%")
    .attr("y1", "0%")
    .attr("y2", "0%");

  gradient.append("stop")
    .attr("offset", "0%")
    .attr("stop-color", "#DEB887"); // light brown

  gradient.append("stop")
    .attr("offset", "100%")
    .attr("stop-color", "#5C4033"); // dark brown

  // 2. Draw the regression line using the gradient stroke
  const regressionPath = g.append("path")
    .datum(regressionData)
    .attr("fill", "none")
    .attr("stroke", "url(#regressionGradient)") // Use the gradient
    .attr("stroke-width", 2)
    .attr("d", line);

  // 3. Animate the stroke drawing
  const totalLength = regressionPath.node().getTotalLength();

  regressionPath
    .attr("stroke-dasharray", `${totalLength} ${totalLength}`)
    .attr("stroke-dashoffset", totalLength)
    .transition()
    .duration(3000)
    .ease(d3.easeCubicOut)
    .attr("stroke-dashoffset", 0);


  if (column === filteredPrecipitationColumns.value[0]) {
    renderLegend(unitPerEllipse, legendRefs[column]);
  }

  combinedData.forEach(d => {
    const totalCircles = Math.floor(d.value / unitPerEllipse);
    const barCenter = xScale(d.year) + xScale.bandwidth() / 2;
    const ellipseRx = Math.min(xScale.bandwidth() / 3, 12);

    for (let i = 0; i < totalCircles; i++) {
      const dropYmm = unitPerEllipse * (i + 1);
      const dropY = yScale(dropYmm);

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
        .duration(2000)
        .ease(d3.easeBounceOut)
        .attr("transform", `translate(${barCenter}, ${dropY}) scale(1, -1)`);
    }
  });

  const xTickYears = years.filter((y, i, arr) => i === 0 || i === arr.length - 1 || y % 10 === 0);

  // X Axis Tick Labels
  g.append("g")
    .selectAll("text.x-axis")
    .data(xTickYears)
    .enter()
    .append("text")
    .attr("class", "x-axis")
    .attr("x", d => xScale(d) + xScale.bandwidth() / 2)
    .attr("y", innerHeight + 25)
    .style("text-anchor", "middle")
    .text(d => d);

  // X Axis Tick Lines
  g.append("g")
    .selectAll("line.x-axis-tick")
    .data(xTickYears)
    .enter()
    .append("line")
    .attr("class", "x-axis-tick")
    .attr("x1", d => xScale(d) + xScale.bandwidth() / 2)
    .attr("x2", d => xScale(d) + xScale.bandwidth() / 2)
    .attr("y1", innerHeight)
    .attr("y2", innerHeight + 6)
    .style("stroke", "black")
    .style("stroke-width", 1);


  // Y-axis Labels
  g.append("g")
    .selectAll("text.y-axis")
    .data(yScale.ticks(5).filter(d => d !== 0))
    .enter()
    .append("text")
    .attr("class", "y-axis")
    .attr("x", -40)
    .attr("y", d => (yScale(d) + 5))
    .style("text-anchor", "middle")
    .text(d => d.toFixed(0));  // Display the values rounded to integers

  // Y-axis Tick Lines
  g.append("g")
    .selectAll("line.y-axis-tick")
    .data(yScale.ticks(5).filter(d => d !== 0))
    .enter()
    .append("line")
    .attr("class", "y-axis-tick")
    .attr("x1", -20)
    .attr("x2", -10)
    .attr("y1", d => yScale(d))
    .attr("y2", d => yScale(d))
    .style("stroke", "black")
    .style("stroke-width", 1);


  const hoverBar = g.append("rect")
    .attr("class", "hover-bar")
    .attr("width", xScale.bandwidth())
    .attr("fill-opacity", 0.7)
    .style("display", "none")
    .attr("y", innerHeight)
    .attr("height", 0);

  g.selectAll(".hover-zone")
    .data(combinedData)
    .enter()
    .append("rect")
    .attr("class", "hover-zone")
    .attr("x", d => xScale(d.year))
    .attr("y", 0)
    .attr("width", xScale.bandwidth())
    .attr("height", innerHeight)
    .attr("fill", "transparent")
    .on("mouseover", function (event, d) {
      hoverBar.interrupt();
      const hoverColor = color(d.source);
      hoverBar.style("display", "block")
        .transition()
        .duration(500)
        .ease(d3.easeCubicOut)
        .attr("x", xScale(d.year))
        .attr("y", yScale(d.value))
        .attr("height", innerHeight - yScale(d.value))
        .attr("fill", hoverColor);

      g.selectAll(".bar-label").remove();
      g.append("text")
        .attr("class", "bar-label")
        .attr("x", xScale(d.year) + xScale.bandwidth() / 2)
        .attr("y", yScale(d.value) - 10)
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

  const unitPerEllipse = getUnitPerEllipse(); // 👈 New line
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
  position: relative;
  display: flex;
  flex-direction: column;
  height: 100vh;
  box-sizing: border-box;
  padding: 4rem 5rem 5rem 5rem;
  overflow: auto;
  opacity: 0;
  transition: opacity 1s ease-in-out;
}

.precip-wrapper.in-view {
  opacity: 1;
  /* Fade in when in view */
}

.precip-wrapper h3 {
  color: #40E0D04f;
  text-transform: uppercase;
  line-height: 1;
  font-size: 5em;
  position: absolute;
  top: 4rem;
  left: 9.5rem;
  text-align: left;
  margin: 0;
  font-weight: 800;
  z-index: 0;
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
  overflow: hidden;
  /* Prevent overflow of chart content */
}

.bar-chart-content {
  width: 100%;
  height: 100%;
  position: relative;
}

.legend {
  position: absolute;
  top: 2.5rem;
  right: 5%;
  padding-left: 1rem;
  z-index: 2;
  pointer-events: none;
}

svg {
  overflow: visible;
}

.x-axis {
  font-size: 0.75em;
}

.y-axis {
  font-size: 0.75em;
}

</style>
