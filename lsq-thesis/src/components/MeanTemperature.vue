<template>
    <div ref="wrapperRef" class="temperature-wrapper unified-heatmap">
        <h3>Burn, Baby, Burn.</h3>

        <div class="control-container">
            <!-- Left Side: Controls -->
            <div class="left-controls">
                <div class="radio-buttons">
                    <label>
                        <input type="radio" v-model="selectedColumnGroup" value="max" /> Max
                    </label>
                    <label>
                        <input type="radio" v-model="selectedColumnGroup" value="mean" /> Mean
                    </label>
                </div>

                <div class="overlay-toggle">
                    <label>
                        <input type="radio" :value="true" v-model="showOverlay" /> Show Covers
                    </label>
                    <label>
                        <input type="radio" :value="false" v-model="showOverlay" /> Hide Covers
                    </label>
                </div>
            </div>

            <!-- Right Side: Legends -->
            <div class="legend">
                <!-- Color Legend -->
                <div class="legend-items">
                    <div class="legend-item" v-for="(color, index) in legendColors" :key="index">
                        <div class="legend-color" :style="{ backgroundColor: color.color }"></div>
                        <span>{{ color.label }}</span>
                    </div>
                </div>

                <!-- Grayscale Legend (directly underneath color legend) -->
                <div class="legend-items grayscale-legend">
                    <div class="legend-item" v-for="(color, index) in grayscaleLegendColors" :key="index">
                        <div class="legend-color" :style="{ backgroundColor: color.color }"></div>
                        <span>{{ color.label }}</span>
                    </div>
                </div>
            </div>
        </div>

        <div class="heatmap-container">
            <div v-for="(column, index) in filteredProjectedColumnOptions" :key="column" class="heatmap-block"
                :class="{ visible: showBlocks[index] }" @mouseenter="hoveredBlock = column"
                @mouseleave="hoveredBlock = null">

                <div class="heatmap-visual">
                    <svg :ref="el => heatmapRefs[column] = el"
                        :data-last-heatmap="index === filteredProjectedColumnOptions.length - 1"></svg>
                    <div class="heatmap-cover" :class="{ hidden: hoveredBlock === column || !showOverlay }">
                        <div class="cover-message">
                            <div class="cover-headings">
                                <h4>{{ getScenarioInfo(column).title }}</h4>
                                <h5>{{ getScenarioInfo(column).subtitle }}</h5>
                                <h6>{{ getScenarioInfo(column).soundBite }}</h6>
                            </div>
                            <div class="cover-paragraph">
                                <p v-html="getScenarioInfo(column).message"></p>
                            </div>
                        </div>

                    </div>

                </div>
            </div>
        </div>
    </div>
</template>



<script setup>
import * as d3 from "d3";
import { ref, onMounted, onBeforeUnmount, nextTick, computed, watch } from "vue";

const heatmapRefs = {};
const hoveredBlock = ref(null);
const wrapperRef = ref(null);
const observer = ref(null);

const historicalData = ref([]);
const projectedData = ref([]);
const projectedColumns = ref([]);

const selectedColumnGroup = ref("max");
const selectedProjectedColumn = ref("");

const hasAnimated = ref(false);

const legendInFahrenheit = ref(false);


// NEW STATE: Show/hide overlays
const showOverlay = ref(true);

const legendColors = computed(() => {
    const celsiusBreaks = [15, 20, 25, 30, 35];
    const fahrenheitBreaks = celsiusBreaks.map(c => Math.round(c * 9 / 5 + 32));
    const labels = legendInFahrenheit.value
        ? [...fahrenheitBreaks.map(t => `${t}°F`), `${Math.round(40 * 9 / 5 + 32)}°F+`]
        : [...celsiusBreaks.map(t => `${t}°C`), "40°C+"];

    const colors = ["#40E0D0", "#7FFF00", "#FFFF00", "#FF8000", "#FF0000", "#820747"]; // colors are: pale turquoise, chartreuse, yellow, orange, red, dark red

    return colors.map((color, i) => ({
        color,
        label: labels[i]
    }));
});

// Grayscale version of the color scale (same range but grayscale)
const grayscaleLegendColors = computed(() => {
    const grayscaleColors = [
        "#D3D3D3", "#A9A9A9", "#808080", "#505050", "#303030", "#101010"
    ];
    const labels = legendColors.value.map(color => color.label); // Use the same labels

    return grayscaleColors.map((color, i) => ({
        color,
        label: labels[i]
    }));
});


const filteredProjectedColumnOptions = computed(() =>
    projectedColumns.value.filter(col =>
        selectedColumnGroup.value === "mean"
            ? col.toLowerCase().includes("mean")
            : col.toLowerCase().includes("max")
    ).reverse()
);

const getScenarioInfo = (columnName) => {
    if (columnName.includes("SSP1-2.6")) {
        return {
            title: "🌱 Green Leap",
            subtitle: "Best-Case Scenario",
            soundBite: "🌿 Too late for the leap — we missed this train.",
            message: "Aiming high on sustainability, this scenario sees the world making strong progress on cutting emissions, hitting net-zero after 2050. <br>The result? Global warming is likely kept below 2°C by 2100. <br>It’s the climate-friendly path."
        };
    } else if (columnName.includes("SSP2-4.5")) {
        return {
            title: "🌍 Middle Ground",
            subtitle: "Business-as-Usual Scenario",
            soundBite: "🛤️ Still on track — but drifting toward danger.",
            message: "This is the “do a little, but not too much” path. <br>Emissions stay close to current levels until around mid-century, then slowly decline. <br>Net-zero isn't reached until <i>after</i> 2100. <br>It's a compromise route, with moderate climate action and moderate consequences."
        };
    } else if (columnName.includes("SSP3-7.0")) {
        return {
            title: "🔥 Divided Drift",
            subtitle: "Worsening Scenario",
            soundBite: "⚠️ Cracks in the system — and no one’s patching them.",
            message: "In this future, the world is fragmented, with regional tensions and little cooperation. <br>Emissions keep rising, nearly doubling by 2100. <br>Climate action takes a back seat, leading to increasing global risks and instability."
        };
    } else if (columnName.includes("SSP5-8.5")) {
        return {
            title: "💣 Turbocharged Trouble",
            subtitle: "Worst-Case Scenario",
            soundBite: "🚨 Pedal to the metal — and no brakes in sight.",
            message: "Fueled by fossil energy and tech innovation, this path prioritizes economic growth over sustainability. <br>Emissions soar, and radiative forcing reaches the highest levels— it’s the “worst-case” outlook. <br>It's the high-speed lane to extreme climate change."
        };
    } else {
        return {
            title: columnName,
            subtitle: columnName,
            soundBite: "🤷‍♂️ Unknown path — map not available.",
            message: "Scenario description unavailable."
        };
    }
};



watch(selectedColumnGroup, () => {
    const filtered = filteredProjectedColumnOptions.value;
    selectedProjectedColumn.value = filtered.length > 0 ? filtered[0] : "";
});

watch([historicalData, projectedData, selectedColumnGroup], async () => {
    if (!historicalData.value.length || !projectedData.value.length) return;

    for (const [index, column] of filteredProjectedColumnOptions.value.entries()) {
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

        await nextTick();
        const isLastHeatmap = index === filteredProjectedColumnOptions.value.length - 1;
        renderUnifiedHeatmap(combinedData, heatmapRefs[column], isLastHeatmap);
    }
}, { immediate: true });

function renderUnifiedHeatmap(data, svgEl, isLastHeatmap) {
    if (!svgEl || !data.length) return;

    const container = svgEl.parentNode;
    const width = container.getBoundingClientRect().width;
    const height = container.getBoundingClientRect().height;

    const months = Array.from(new Set(data.map(d => d.month))).reverse();
    const years = Array.from(new Set(data.map(d => d.year)));

    const margin = { top: 10, right: 20, bottom: 30, left: 60 };
    const innerHeight = height - margin.top - margin.bottom;
    const innerWidth = width - margin.left - margin.right;

    const svg = d3.select(svgEl)
        .attr("width", "100%")
        .attr("height", "100%")
        .attr("viewBox", `0 0 ${width} ${height}`);

    svg.selectAll("*").remove();

    const xScale = d3.scaleBand().domain(years).range([0, innerWidth]).padding(0.05);
    const yScale = d3.scaleBand().domain(months).range([innerHeight, 0]).padding(0.05);

    const colorScale = d3.scaleThreshold()
        .domain([15, 20, 25, 30, 35])
        .range(["#40E0D0", "#7FFF00", "#FFFF00", "#FF8000", "#FF0000", "#820747"]);

    const g = svg.append("g").attr("transform", `translate(${margin.left},${margin.top})`);

    const xAxisTicks = years.filter((y, i) => i === 0 || i === years.length - 1 || y % 10 === 0);
    if (isLastHeatmap) {
        g.append("g")
            .selectAll("text.x-axis")
            .data(xAxisTicks)
            .enter()
            .append("text")
            .attr("class", "x-axis")
            .attr("x", d => xScale(d) + xScale.bandwidth() / 2)
            .attr("y", innerHeight + 25)
            .style("text-anchor", "middle")
            .text(d => d);
    }

    g.append("g")
        .selectAll("line.x-axis-tick")
        .data(xAxisTicks)
        .enter()
        .append("line")
        .attr("x1", d => xScale(d) + xScale.bandwidth() / 2)
        .attr("x2", d => xScale(d) + xScale.bandwidth() / 2)
        .attr("y1", innerHeight)
        .attr("y2", innerHeight + 6)
        .style("stroke", "black");

    g.append("g")
        .selectAll("text.y-axis")
        .data(months)
        .enter()
        .append("text")
        .attr("class", "y-axis")
        .attr("x", -25)
        .attr("y", d => yScale(d) + yScale.bandwidth() / 1.25)
        .style("text-anchor", "middle")
        .text(d => d);

    // Function to convert color to grayscale
    function toGrayscale(color) {
        const rgb = d3.rgb(color);
        const grayscale = 0.2126 * rgb.r + 0.7152 * rgb.g + 0.0722 * rgb.b;
        return d3.rgb(grayscale, grayscale, grayscale);
    }

    g.selectAll(".heatmap-cell")
        .data(data)
        .enter()
        .append("rect")
        .attr("class", "heatmap-cell")
        .attr("x", d => xScale(d.year))
        .attr("y", d => yScale(d.month))
        .attr("width", xScale.bandwidth())
        .attr("height", yScale.bandwidth())
        .style("fill", d => {
            const color = colorScale(d.value);
            // Convert the color to grayscale for years 2020 or earlier
            if (d.year <= 2020) {
                return toGrayscale(color); // Convert color to grayscale
            }
            return color; // Keep the original color for later years
        })
        .style("opacity", 0)
        .transition()
        .duration(500)
        .style("opacity", 1);
}

const showBlocks = ref([]);

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

        // Initial state for animation
        showBlocks.value = Array(filteredProjectedColumnOptions.value.length).fill(false);

        for (let i = 0; i < filteredProjectedColumnOptions.value.length; i++) {
            const column = filteredProjectedColumnOptions.value[i];
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

            renderUnifiedHeatmap(combinedData, heatmapRefs[column], column === filteredProjectedColumnOptions.value.at(-1));


        }

        // IntersectionObserver setup
        observer.value = new IntersectionObserver((entries) => {
            const entry = entries[0];
            if (entry.isIntersecting && !hasAnimated.value) {
                hasAnimated.value = true; // prevent future triggers

                // Start animation only once
                showBlocks.value = Array(filteredProjectedColumnOptions.value.length).fill(false);

                for (let i = 0; i < filteredProjectedColumnOptions.value.length; i++) {
                    const column = filteredProjectedColumnOptions.value[i];
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

                    renderUnifiedHeatmap(combinedData, heatmapRefs[column], column === filteredProjectedColumnOptions.value.at(-1));

                    setTimeout(() => {
                        showBlocks.value[i] = true;
                    }, i * 1000);
                }
            }
        }, { threshold: 0.3 });

        let toggleInterval = setInterval(() => {
            legendInFahrenheit.value = !legendInFahrenheit.value;
        }, 10000); // every 10 seconds

        onBeforeUnmount(() => {
            clearInterval(toggleInterval);
        });


        let resizeTimeout;
        const handleResize = () => {
            clearTimeout(resizeTimeout);
            resizeTimeout = setTimeout(() => {
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
                    renderUnifiedHeatmap(combinedData, heatmapRefs[column], column === filteredProjectedColumnOptions.value.at(-1));
                }
            }, 1000);
        };

        window.addEventListener("resize", handleResize);
        onBeforeUnmount(() => {
            window.removeEventListener("resize", handleResize);
        });

    } catch (err) {
        console.error("Error loading CSV data:", err);
    }

    if (wrapperRef.value && observer.value) {
        observer.value.observe(wrapperRef.value);
    }

});


onBeforeUnmount(() => {
    if (observer.value && wrapperRef.value) {
        observer.value.unobserve(wrapperRef.value);
    }
    window.removeEventListener("resize", renderUnifiedHeatmap);
});

</script>

<style scoped>
.temperature-wrapper {
    height: 100vh;
    width: 100vw;
    display: flex;
    flex-direction: column;
    padding: 4rem 5rem 8rem 5rem;
    box-sizing: border-box;
    overflow: hidden;
}

.control-container {
    display: flex;
    justify-content: space-between;
    /* Push children to edges */
    align-items: flex-start;
    margin-bottom: 0.5rem;
    flex-shrink: 0;
    gap: 2rem;
}

.left-controls {
    display: flex;
    flex-direction: column;
}

.radio-buttons,
.overlay-toggle {
    display: flex;
    gap: 10px;
}

.legend {
    display: flex;
    flex-direction: column; /* Stack legends vertically */
    align-items: flex-start;
}

.legend-items {
    display: flex;
    gap: 15px;
    flex-wrap: wrap;
}

.legend-item {
    display: flex;
    align-items: center;
    gap: 5px;
}

.legend-color {
    width: 15px;
    height: 15px;
}



.heatmap-container {
    display: flex;
    flex-direction: column;
    gap: 0;
    flex: 1;
    height: 100%;
    margin-top: 0;
    /* Ensure no margin at the top */
}

.heatmap-block {
    flex: 1;
    height: 25%;
    /* Ensure equal distribution of space */
    width: 100%;
    display: flex;
    align-items: flex-start;
    /* Align heatmaps at the top */
    justify-content: center;
    margin-top: 0;
    /* Remove any margin pushing the blocks down */
    opacity: 0;
    transform: translateY(20px);
    transition: opacity 1s ease, transform 1s ease;
}

.heatmap-block.visible {
    opacity: 1;
    transform: translateY(0);
}

.heatmap-visual {
    position: relative;
    width: 100%;
    height: 100%;
    display: flex;
    justify-content: flex-start;
    /* Align heatmap content to the top */
}

.heatmap-cover {
    position: absolute;
    top: 0;
    /* Align the cover to the top */
    left: 0;
    width: 100%;
    height: 100%;
    color: white;
    display: flex;
    justify-content: center;
    align-items: center;
    z-index: 2;
    transition: opacity 0.3s ease;
    pointer-events: auto;
    font-weight: bold;
    font-size: 1.1rem;
    cursor: pointer;
}

.heatmap-block .heatmap-cover {
    height: calc(100% - 30px);
}

.heatmap-cover.hidden {
    opacity: 0;
    pointer-events: none;
}

.cover-message {
    display: flex;
    width: 100%;
    flex-direction: row;
    gap: 1rem;
    align-items: center;
}

.cover-headings,
.cover-paragraph {
    display: flex;
    flex-direction: column;
    width: 50%;
    padding: 0 10rem;
}

.x-axis {
    font-size: 0.75em;
}

.y-axis {
    font-size: 0.75em;
}

.heatmap-block:nth-child(2),
.heatmap-block:nth-child(3),
.heatmap-block:nth-child(4) {
    margin-top: -25px;
}

.heatmap-block:nth-child(4) .heatmap-cover {
    background-color: rgb(102, 0, 0);
}

.heatmap-block:nth-child(3) .heatmap-cover {
    background-color: rgb(153, 0, 0);
}

.heatmap-block:nth-child(2) .heatmap-cover {
    background-color: rgb(204, 0, 0);
}

.heatmap-block:nth-child(1) .heatmap-cover {
    background-color: rgb(255, 0, 0);
}

h4 {
    text-align: left;
    font-size: 1em;
    margin: 0;
}

h5 {
    text-align: left;
    font-size: 0.8em;
    margin: 0;
}

h6 {
    text-align: left;
    font-size: 0.6em;
    margin: 0;
}

p {
    font-size: 1em;
    margin: 0;
    text-align: left;
    font-weight: 400;
}
</style>