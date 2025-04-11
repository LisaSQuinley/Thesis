<template>
    <div class="temperature-wrapper">
        <!-- Historical Temperature Section -->
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

            <!-- Invisible dropdown spacer to align with projected section -->
            <div class="dropdown-spacer"></div>

            <svg ref="historicalSvg"></svg>
        </div>

        <!-- Projected Temperature Section -->
        <div class="mean-max-temperature projected-temperature">
            <h3>Projected Mean and Maximum Surface Air Temperature</h3>

            <!-- New radio buttons for selecting mean or max group -->
            <div>
                <label>
                    <input type="radio" v-model="selectedProjectedColumnGroup" value="mean" /> Mean
                </label>
                <label>
                    <input type="radio" v-model="selectedProjectedColumnGroup" value="max" /> Max
                </label>
            </div>

            <!-- Dropdown filters based on radio selection -->
            <select v-if="filteredProjectedColumnOptions.length > 0" v-model="selectedProjectedColumn">
                <option v-for="column in filteredProjectedColumnOptions" :key="column" :value="column">
                    {{ getDropdownTitle(column) }}
                </option>
            </select>

            <svg ref="projectedSvg"></svg>

            <!-- Dynamic message below projected heatmap -->
            <div class="projected-message" v-if="projectedMessage">
                {{ projectedMessage }}
            </div>
        </div>
    </div>
</template>




<script setup>
import * as d3 from "d3";
import { ref, onMounted, onBeforeUnmount, nextTick, watchEffect, computed, watch } from "vue";

// ==== Historical Data Refs ====
const historicalData = ref([]);
const historicalColumns = ref([]);
const selectedHistoricalColumn = ref('max');
const historicalSvg = ref(null);

// ==== Projected Data Refs ====
const projectedData = ref([]);
const projectedColumns = ref([]);
const selectedProjectedColumnGroup = ref("max"); // "mean" or "max"
const selectedProjectedColumn = ref("");
const projectedSvg = ref(null);

// Filter dropdown options based on the selected group (mean or max)
const filteredProjectedColumnOptions = computed(() => {
    return projectedColumns.value.filter(col =>
        selectedProjectedColumnGroup.value === "mean"
            ? col.toLowerCase().includes("mean")
            : col.toLowerCase().includes("max")
    );
});

// Watch to reset dropdown when group changes
watch(selectedProjectedColumnGroup, () => {
    const filtered = filteredProjectedColumnOptions.value;
    selectedProjectedColumn.value = filtered.length > 0 ? filtered[0] : "";
});

watch(selectedProjectedColumn, (newValue) => {
    // Log the selected column and the data related to it
    console.log("Selected Projected Column:", newValue);

    if (newValue) {
        const columnData = projectedData.value.map(d => d[newValue]);
        console.log("Data for selected column:", columnData);
    }
});

// ==== Historical Heatmap Rendering ====
function renderHistoricalHeatmap() {
    if (!historicalSvg.value) return;

    const container = historicalSvg.value.parentNode;
    const containerWidth = container.getBoundingClientRect().width;
    const containerHeight = containerWidth;

    const svg = d3.select(historicalSvg.value)
        .attr("width", containerWidth)
        .attr("height", containerHeight)
        .attr("viewBox", `0 0 ${containerWidth} ${containerHeight}`);

    svg.selectAll("*").remove();

    const groupedData = d3.group(historicalData.value, d => d.year, d => d.month);
    const heatmapData = [];

    groupedData.forEach((yearData, year) => {
        yearData.forEach((values, month) => {
            const firstEntry = values[0];
            const selectedColumn =
                selectedHistoricalColumn.value === "mean" ? "Mean Temp" : "Max Temp";
            const tempValue = parseFloat(firstEntry[selectedColumn]);
            if (!isNaN(tempValue)) {
                heatmapData.push({ year, month, value: tempValue });
            }
        });
    });

    const margin = { top: 10, right: 20, bottom: 50, left: 60 };
    const width = containerWidth - margin.left - margin.right;
    const height = containerHeight - margin.top - margin.bottom;

    const years = Array.from(groupedData.keys());
    const months = Array.from(new Set(historicalData.value.map(d => d.month))).reverse();

    const xScale = d3.scaleBand().domain(years).range([0, width]).padding(0.05);
    const yScale = d3.scaleBand().domain(months).range([height, 0]).padding(0.05);

    const colorScale = d3.scaleThreshold()
        .domain([15, 20, 25, 30, 35])
        .range(["#40E0D0", "#7FFF00", "#FFFF00", "#FF8000", "#FF0000", "#820747"]);


    const g = svg.append("g").attr("transform", `translate(${margin.left},${margin.top})`);

    const filteredYears = years.filter((year, idx, arr) => {
        return (
            idx === 0 || // First year
            idx === arr.length - 1 || // Last year
            year % 5 === 0 // Years ending in 0 or 5
        );
    });

    g.append("g")
        .selectAll(".x-axis")
        .data(filteredYears)
        .enter()
        .append("text")
        .attr("class", "x-axis")
        .attr("x", d => xScale(d) + xScale.bandwidth() / 1 + 30)
        .attr("y", height + 45)
        .attr("transform", d => `rotate(-90, ${xScale(d) + xScale.bandwidth() / 2}, ${height + 40})`)
        .style("text-anchor", "end")
        .text(d => d);

    g.append("g")
        .selectAll(".y-axis")
        .data(months)
        .enter()
        .append("text")
        .attr("class", "y-axis")
        .attr("x", -25)
        .attr("y", d => yScale(d) + yScale.bandwidth() / 1.5)
        .style("text-anchor", "middle")
        .text(d => d);

    const monthOrder = new Map(months.map((m, i) => [m, i]));
    const yearOrder = new Map(years.map((y, i) => [y, i]));
    const getDelay = d => {
        const yIndex = yearOrder.get(d.year);
        const mIndex = monthOrder.get(d.month);
        return (yIndex * months.length + mIndex) * 10;
    };

    const cells = g.selectAll(".heatmap-cell").data(heatmapData, d => `${d.year}-${d.month}`);

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

    cells.transition()
        .delay(d => getDelay(d))
        .duration(500)
        .style("fill", d => colorScale(d.value));

    cells.exit()
        .transition()
        .delay(d => getDelay(d))
        .duration(300)
        .style("opacity", 0)
        .remove();
}

// Render the projected heatmap
function renderProjectedHeatmap() {
    if (!projectedSvg.value || !selectedProjectedColumn.value) return;

    const container = projectedSvg.value.parentNode;
    const containerWidth = container.getBoundingClientRect().width;
    const containerHeight = containerWidth;

    const svg = d3.select(projectedSvg.value)
        .attr("width", containerWidth)
        .attr("height", containerHeight)
        .attr("viewBox", `0 0 ${containerWidth} ${containerHeight}`);

    svg.selectAll("*").remove();

    const groupedData = d3.group(
        projectedData.value,
        d => d.year,
        d => d.month
    );

    const heatmapData = [];
    groupedData.forEach((yearData, year) => {
        yearData.forEach((values, month) => {
            const firstEntry = values[0];
            const val = parseFloat(firstEntry[selectedProjectedColumn.value]);
            if (!isNaN(val)) {
                heatmapData.push({ year, month, value: val });
            }
        });
    });

    const margin = { top: 10, right: 20, bottom: 50, left: 60 };
    const width = containerWidth - margin.left - margin.right;
    const height = containerHeight - margin.top - margin.bottom;

    const years = Array.from(groupedData.keys());
    const months = Array.from(new Set(projectedData.value.map(d => d.month))).reverse();

    const xScale = d3.scaleBand().domain(years).range([0, width]).padding(0.05);
    const yScale = d3.scaleBand().domain(months).range([height, 0]).padding(0.05);

    const colorScale = d3.scaleThreshold()
        .domain([15, 20, 25, 30, 35])
        .range(["#40E0D0", "#7FFF00", "#FFFF00", "#FF8000", "#FF0000", "#820747"]);


    const g = svg.append("g").attr("transform", `translate(${margin.left},${margin.top})`);

    const filteredYears = years.filter((year, idx, arr) => {
        return (
            idx === 0 || // First year
            idx === arr.length - 1 || // Last year
            year % 5 === 0 // Years ending in 0 or 5
        );
    });

    g.append("g")
        .selectAll(".x-axis")
        .data(filteredYears)
        .enter()
        .append("text")
        .attr("class", "x-axis")
        .attr("x", d => xScale(d) + xScale.bandwidth() / 1 + 30)
        .attr("y", height + 45)
        .attr("transform", d => `rotate(-90, ${xScale(d) + xScale.bandwidth() / 2}, ${height + 40})`)
        .style("text-anchor", "end")
        .text(d => d);

    g.append("g")
        .selectAll(".y-axis")
        .data(months)
        .enter()
        .append("text")
        .attr("class", "y-axis")
        .attr("x", -25)
        .attr("y", d => yScale(d) + yScale.bandwidth() / 1.5)
        .style("text-anchor", "middle")
        .text(d => d);

    const monthOrder = new Map(months.map((m, i) => [m, i]));
    const yearOrder = new Map(years.map((y, i) => [y, i]));
    const getDelay = d => {
        const yIndex = yearOrder.get(d.year);
        const mIndex = monthOrder.get(d.month);
        return (yIndex * months.length + mIndex) * 10;
    };

    const cells = g.selectAll(".heatmap-cell").data(heatmapData, d => `${d.year}-${d.month}`);

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

    cells.transition()
        .delay(d => getDelay(d))
        .duration(500)
        .style("fill", d => colorScale(d.value));

    cells.exit()
        .transition()
        .delay(d => getDelay(d))
        .duration(300)
        .style("opacity", 0)
        .remove();
}

// ==== Lifecycle & Data Loading ====
onMounted(async () => {
    try {
        historicalData.value = await d3.csv("./data/MeanMaxSurfaceAirTempuratureAndPrecipitation-1951-2020-Month.csv");
        projectedData.value = await d3.csv("./data/AverageMeanAndMaxSurfaceAirTemperatureAndPrecipitation-2020-2099-Month-FourScenarios.csv");

        historicalColumns.value = Object.keys(historicalData.value[0] || {});
        projectedColumns.value = Object.keys(projectedData.value[0] || {});

        // Set the default selection for the projected data dropdown
        const initialFilteredColumn = filteredProjectedColumnOptions.value[0];
        selectedProjectedColumn.value = initialFilteredColumn || "";

        await nextTick();
        renderHistoricalHeatmap();
        renderProjectedHeatmap();

        window.addEventListener("resize", renderHistoricalHeatmap);
        window.addEventListener("resize", renderProjectedHeatmap);
    } catch (error) {
        console.error("Error loading data:", error);
    }
});


watchEffect(() => {
    renderHistoricalHeatmap();
    renderProjectedHeatmap();
});

onBeforeUnmount(() => {
    window.removeEventListener("resize", renderHistoricalHeatmap);
    window.removeEventListener("resize", renderProjectedHeatmap);
});



const projectedMessage = computed(() => {
    const val = selectedProjectedColumn.value;

    if (!val) return "";

    if (val.includes("SSP1-2.6")) {
        return "Aiming high on sustainability, this scenario sees the world making strong progress on cutting emissions, hitting net-zero after 2050. The result? Global warming is likely kept below 2°C by 2100. It’s the climate-friendly path.";
    } else if (val.includes("SSP2-4.5")) {
        return "This is the “do a little, but not too much” path. Emissions stay close to current levels until around mid-century, then slowly decline. Net-zero isn't reached until <i>after</i> 2100. It's a compromise route, with moderate climate action and moderate consequences.";
    } else if (val.includes("SSP3-7.0")) {
        return "In this future, the world is fragmented, with regional tensions and little cooperation. Emissions keep rising, nearly doubling by 2100. Climate action takes a back seat, leading to increasing global risks and instability.";
    } else if (val.includes("SSP5-8.5")) {
        return "Fueled by fossil energy and tech innovation, this path prioritizes economic growth over sustainability. Emissions soar, and radiative forcing reaches the highest levels— it’s the “worst-case” outlook. It's the high-speed lane to extreme climate change.";
    } else {
        return "Select a dataset to view its projection details.";
    }
});

const getDropdownTitle = (columnName) => {
    if (columnName.includes("SSP1-2.6")) {
        return "🌱 Green Leap (SSP1-2.6 – Best-Case Scenario)";
    } else if (columnName.includes("SSP2-4.5")) {
        return "🌍 Middle Ground (SSP2-4.5 – Business-as-Usual Scenario)";
    } else if (columnName.includes("SSP3-7.0")) {
        return "🔥 Divided Drift (SSP3-7.0 – Worsening Scenario)";
    } else if (columnName.includes("SSP5-8.5")) {
        return "💣 Turbocharged Trouble (SSP5-8.5 – Worst-Case Scenario)";
    }
    return columnName; // Default return if no match
};

</script>


<style scoped>
.temperature-wrapper {
    display: flex;
    justify-content: space-between;
    gap: 20px;
}

.mean-max-temperature {
    flex: 1;
    /* Let each column grow/shrink equally */
    max-width: 50%;
}

.historical-temperature,
.projected-temperature {
    width: 100%;
    /* Keep full width inside their containers */
}

/* Optional: Adjust SVG sizes responsively */
svg {
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

.dropdown-spacer {
    height: 24px;
    /* match the height of the dropdown */
    visibility: hidden;
    /* still takes up space */
}

.projected-message {
    margin-top: 10px;
    font-size: 14px;
    color: #444;
    background-color: snow;
    padding: 8px 12px;
    border-radius: 4px;
}

</style>
