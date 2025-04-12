<template>
    <div class="temperature-wrapper unified-heatmap">
      <h3>Historical + Projected Surface Air Temperature</h3>
  
      <!-- Radio buttons for mean/max -->
      <div>
        <label>
          <input type="radio" v-model="selectedColumnGroup" value="mean" /> Mean
        </label>
        <label>
          <input type="radio" v-model="selectedColumnGroup" value="max" /> Max
        </label>
      </div>
  
      <!-- Heatmap blocks per scenario -->
      <div v-for="column in filteredProjectedColumnOptions" :key="column" class="heatmap-block">
  <div class="heatmap-content">
    <div class="heatmap-description">
      <h4>{{ getDropdownTitle(column) }}</h4>
      <h5>{{ getDropdownSubtitle(column) }}</h5>
      <p v-html="getProjectedMessage(column)"></p>
    </div>
    <svg :ref="el => heatmapRefs[column] = el"></svg>
  </div>
</div>
</div>
  </template>
  
  


  <script setup>
  import * as d3 from "d3";
  import { ref, onMounted, onBeforeUnmount, nextTick, computed, watch } from "vue";
  
  const heatmapRefs = {}; // dynamic refs by column
  
  const historicalData = ref([]);
  const projectedData = ref([]);
  const projectedColumns = ref([]);
  
  const selectedColumnGroup = ref("max");
  const selectedProjectedColumn = ref("");
  
  const filteredProjectedColumnOptions = computed(() =>
    projectedColumns.value.filter(col =>
      selectedColumnGroup.value === "mean"
        ? col.toLowerCase().includes("mean")
        : col.toLowerCase().includes("max")
    )
  );
  
  watch(selectedColumnGroup, () => {
    const filtered = filteredProjectedColumnOptions.value;
    selectedProjectedColumn.value = filtered.length > 0 ? filtered[0] : "";
  });
  
  //const unifiedData = ref([]);
  
  watch(
  [historicalData, projectedData, selectedColumnGroup],
  async () => {
    if (!historicalData.value.length || !projectedData.value.length) return;

    for (const column of filteredProjectedColumnOptions.value) {
      const histCol = selectedColumnGroup.value === "mean" ? "Mean Temp" : "Max Temp";

      const combinedData = [
        ...historicalData.value.map(d => ({
          year: +d.year,
          month: d.month,
          value: parseFloat(d[histCol]),
          source: "historical"
        })),
        ...projectedData.value.map(d => ({
          year: +d.year,
          month: d.month,
          value: parseFloat(d[column]),
          source: "projected"
        }))
      ];

      await nextTick(); // wait for DOM

      renderUnifiedHeatmap(combinedData, heatmapRefs[column]);
    }
  },
  { immediate: true }
);

  
function renderUnifiedHeatmap(data, svgEl) {
  if (!svgEl || !data.length) return;

  const container = svgEl.parentNode;
  const width = container.getBoundingClientRect().width;
  const height = 300; // fixed height for each heatmap

  const svg = d3.select(svgEl)
    .attr("width", width)
    .attr("height", height)
    .attr("viewBox", `0 0 ${width} ${height}`);

  svg.selectAll("*").remove();

  const margin = { top: 10, right: 20, bottom: 50, left: 60 };
  const innerWidth = width - margin.left - margin.right;
  const innerHeight = height - margin.top - margin.bottom;

  const years = Array.from(new Set(data.map(d => d.year)));
  const months = Array.from(new Set(data.map(d => d.month))).reverse();

  const xScale = d3.scaleBand().domain(years).range([0, innerWidth]).padding(0.05);
  const yScale = d3.scaleBand().domain(months).range([innerHeight, 0]).padding(0.05);

  const colorScale = d3.scaleThreshold()
    .domain([15, 20, 25, 30, 35])
    .range(["#40E0D0", "#7FFF00", "#FFFF00", "#FF8000", "#FF0000", "#820747"]);

  const g = svg.append("g").attr("transform", `translate(${margin.left},${margin.top})`);

  g.append("g")
    .selectAll("text.x-axis")
    .data(years.filter((y, i, arr) => i === 0 || i === arr.length - 1 || y % 10 === 0))
    .enter()
    .append("text")
    .attr("class", "x-axis")
    .attr("x", d => xScale(d) + xScale.bandwidth() / 2)
    .attr("y", innerHeight + 20)
//    .attr("transform", d => `rotate(-90, ${xScale(d) + xScale.bandwidth() / 2}, ${innerHeight + 25})`)
    .style("text-anchor", "middle")
    .text(d => d);

  g.append("g")
    .selectAll("text.y-axis")
    .data(months)
    .enter()
    .append("text")
    .attr("class", "y-axis")
    .attr("x", -25)
    .attr("y", d => yScale(d) + yScale.bandwidth() / 1.5)
    .style("text-anchor", "middle")
    .text(d => d);

  g.selectAll(".heatmap-cell")
    .data(data)
    .enter()
    .append("rect")
    .attr("class", "heatmap-cell")
    .attr("x", d => xScale(d.year))
    .attr("y", d => yScale(d.month))
    .attr("width", xScale.bandwidth())
    .attr("height", yScale.bandwidth())
    .style("fill", d => colorScale(d.value))
    .style("opacity", 0)
    .transition()
    .duration(500)
    .style("opacity", 1);
}

  
  onMounted(async () => {
    try {
      const hist = await d3.csv("./data/MeanMaxSurfaceAirTempuratureAndPrecipitation-1951-2020-Month.csv");
      const proj = await d3.csv("./data/AverageMeanAndMaxSurfaceAirTemperatureAndPrecipitation-2020-2099-Month-FourScenarios.csv");
  
      historicalData.value = hist;
      projectedData.value = proj;
      projectedColumns.value = Object.keys(proj[0] || {});
  
      const defaultGroup = selectedColumnGroup.value;
      const defaultOptions = projectedColumns.value.filter(col =>
        defaultGroup === "mean" ? col.toLowerCase().includes("mean") : col.toLowerCase().includes("max")
      );
      selectedProjectedColumn.value = defaultOptions[0];
  
      window.addEventListener("resize", renderUnifiedHeatmap);
    } catch (err) {
      console.error("Error loading CSV data:", err);
    }
  });
  
  onBeforeUnmount(() => {
    window.removeEventListener("resize", renderUnifiedHeatmap);
  });

  const getProjectedMessage = (columnName) => {
  if (columnName.includes("SSP1-2.6")) {
    return "Aiming high on sustainability, this scenario sees the world making strong progress on cutting emissions, hitting net-zero after 2050. The result? Global warming is likely kept below 2°C by 2100. It’s the climate-friendly path.";
  } else if (columnName.includes("SSP2-4.5")) {
    return "This is the “do a little, but not too much” path. Emissions stay close to current levels until around mid-century, then slowly decline. Net-zero isn't reached until <i>after</i> 2100. It's a compromise route, with moderate climate action and moderate consequences.";
  } else if (columnName.includes("SSP3-7.0")) {
    return "In this future, the world is fragmented, with regional tensions and little cooperation. Emissions keep rising, nearly doubling by 2100. Climate action takes a back seat, leading to increasing global risks and instability.";
  } else if (columnName.includes("SSP5-8.5")) {
    return "Fueled by fossil energy and tech innovation, this path prioritizes economic growth over sustainability. Emissions soar, and radiative forcing reaches the highest levels— it’s the “worst-case” outlook. It's the high-speed lane to extreme climate change.";
  } else {
    return "Scenario description unavailable.";
  }
};

const getDropdownSubtitle = (columnName) => {
    if (columnName.includes("SSP1-2.6")) {
        return "SSP1-2.6: Best-Case Scenario";
    } else if (columnName.includes("SSP2-4.5")) {
        return "SSP2-4.5: Business-as-Usual Scenario";
    } else if (columnName.includes("SSP3-7.0")) {
        return "SSP3-7.0: Worsening Scenario";
    } else if (columnName.includes("SSP5-8.5")) {
        return "SSP5-8.5: Worst-Case Scenario";
    }
    return columnName;
  };
  
  const getDropdownTitle = (columnName) => {
    if (columnName.includes("SSP1-2.6")) {
        return "🌱 Green Leap";
    } else if (columnName.includes("SSP2-4.5")) {
        return "🌍 Middle Ground";
    } else if (columnName.includes("SSP3-7.0")) {
        return "🔥 Divided Drift";
    } else if (columnName.includes("SSP5-8.5")) {
        return "💣 Turbocharged Trouble";
    }
    return columnName;
  };
  </script>



<style scoped>
.temperature-wrapper {
    width: 100%;
    display: flex;
    flex-direction: column;
    gap: 20px;
}

.mean-max-temperature {
    width: 100%;
}

.historical-temperature,
.projected-temperature {
    width: 100%;
    /* Keep full width inside their containers */
}

.heatmap-block {
  width: 100%;
}

.heatmap-content {
  display: flex;
  gap: 20px;
  align-items: flex-start;
}

.heatmap-description {
  flex: 0 0 250px;
  max-width: 250px;
}

svg {
  flex: 1;
  width: 100%;
  height: auto;
}


select {
    font-family: "Parkinsans", sans-serif;
}

option {
    font-family: "Parkinsans", sans-serif;
}

.heatmap-cell {
    transition: fill 0.3s ease;
}

.x-axis {
    font-size: 0.75em;
    text-align: right;
}

.y-axis {
    font-size: 12px;
}

p {
  font-size: 0.9em;
  margin: 0;
  text-align: left;
}

</style>
