<template>
  <div class="map-wrapper" ref="mapWrapper">
    <div class="map-container">
      <h3 class="title">Disaster Time Machine</h3>
      <l-map ref="map" v-model:zoom="zoom" :center="[20, 0]" :zoom="zoom" :max-bounds="[[-60, -180], [85, 180]]">
        <l-geo-json v-if="geoJsonData" :geojson="geoJsonData" :key="year" :options-style="getCountryStyle" />
      </l-map>
    </div>
    <div class="color-legend" v-if="year">
      <div class="legend-label left">Few</div>
      <div class="legend-gradient"></div>
      <div class="legend-label right">Many</div>
    </div>
    <div class="custom-slider" v-if="years.length && isInView">
      <div class="slider-track"></div>
      <div class="slider-markers">
        <div v-for="(y, i) in years" :key="y" class="slider-marker" :class="{ active: year === y }"
          @click="selectedIndex = i; stopYearLoop();">

          <div class="circle"></div>
          <div class="label">{{ y }}</div>
        </div>
      </div>
    </div>
    <div class="event-summary-types" v-if="year">
      <div class="summary">
        <h4>{{ year }}</h4>
        <h5>{{ totalEventsForSelectedYear }} disasters</h5>
      </div>

      <div v-if="eventTypeBreakdownForSelectedYear.length">
        <ul>
          <li v-for="([type, count], i) in eventTypeBreakdownForSelectedYear" :key="i">
            {{ type }}: <strong>{{ count }}</strong>
          </li>
        </ul>
      </div>
    </div>
  </div>
</template>

<script>
import * as d3 from "d3";
import "leaflet/dist/leaflet.css";
import { LMap, LGeoJson } from "@vue-leaflet/vue-leaflet";

export default {
  components: {
    LMap,
    LGeoJson,
  },

  data() {
    return {
      looping: true,
      yearLoopInterval: null,
      zoom: 3,
      geoJsonData: null,
      disasterCounts: [],
      year: null,
      years: [],
      colorScale: () => "transparent", // default fallback
      isInView: false,
    };
  },

  computed: {
    selectedIndex: {
      get() {
        return this.years.indexOf(this.year);
      },
      set(val) {
        this.year = this.years[val];
      },
    },
    totalEventsForSelectedYear() {
      const year = this.year;
      return this.disasterCounts
        .filter((d) => d.year === year)
        .reduce((sum, d) => sum + d.totalDisasters, 0);
    },
    eventTypeBreakdownForSelectedYear() {
      const year = this.year;
      const typeTotals = {};

      this.disasterCounts
        .filter((d) => d.year === year)
        .forEach((d) => {
          Object.entries(d).forEach(([key, value]) => {
            if (
              key !== "year" &&
              key !== "country" &&
              key !== "iso" &&
              key !== "totalDisasters" &&
              value > 0
            ) {
              typeTotals[key] = (typeTotals[key] || 0) + value;
            }
          });
        });

      // Sort alphabetically by disaster type
      return Object.entries(typeTotals).sort((a, b) => a[0].localeCompare(b[0]));
    },

  },

  watch: {
    year(newYear) {
      const values = this.disasterCounts
        .filter((d) => d.year === newYear)
        .map((d) => d.totalDisasters)
        .filter((v) => v > 0); // ignore zeros for max

      const max = d3.max(values) || 1;

      const colorInterpolator = d3.interpolateRgb("#E9D1D5", "#821749");

      const scale = d3.scaleLinear()
        .domain([1, max])
        .interpolate(() => colorInterpolator);

      this.colorScale = (value) => {
        if (!value || value === 0) return "transparent";
        return scale(value);
      };
    },
  },

  methods: {
    getCountryStyle(feature) {
      const countryName = feature.properties.SOVEREIGNT;
      const dataForYear = this.disasterCounts.find(
        (d) => d.year === this.year && d.country === countryName
      );
      const value = dataForYear ? dataForYear.totalDisasters : 0;

      return {
        fillColor: this.colorScale(value),
        weight: 2,
        opacity: 1,
        color: "#089c9d",
        fillOpacity: 1,
      };
    },
    startYearLoop() {
    if (this.yearLoopInterval || !this.years.length || !this.isInView) return;

    this.yearLoopInterval = setInterval(() => {
      let nextIndex = (this.selectedIndex + 1) % this.years.length;
      this.selectedIndex = nextIndex;
    }, 3000); // 3 seconds
  },

  stopYearLoop() {
    clearInterval(this.yearLoopInterval);
    this.yearLoopInterval = null;
  },

  // ✅ New method to get total events by year
  getTotalEventsByYear() {
    const totals = {};

    this.disasterCounts.forEach((d) => {
      if (!totals[d.year]) {
        totals[d.year] = 0;
      }
      totals[d.year] += d.totalDisasters;
    });

    return totals;
  },
},

  mounted() {
    // Load GeoJSON
    fetch("./data/country-boundaries.geojson")
      .then((response) => response.json())
      .then((data) => {
        const filtered = {
          ...data,
          features: data.features.filter(
            (f) => f.properties.SOVEREIGNT !== "Antarctica"
          ),
        };
        this.geoJsonData = filtered;
      })
      .catch((error) => {
        console.error("Error loading GeoJSON file:", error);
      });

    // Load and process CSV
    fetch("./data/EM-DAT-data-ALL.csv")
      .then((response) => response.text())
      .then((data) => {
        const parsedData = d3.csvParse(data);

        const filtered = parsedData
          .filter(
            (d) =>
              d["Disaster Subgroup"] !== "Geophysical" &&
              d["Disaster Subgroup"] !== "Biological"
          )
          .map((d) => ({
            year: d["DisNo."].substring(0, 4),
            iso: d["ISO"],
            country: d["Country"],
            type: d["Disaster Type"],
          }));

        const grouped = d3.rollup(
          filtered,
          (entries) => {
            const typeCounts = d3.rollup(
              entries,
              (v) => v.length,
              (d) => d.type
            );
            const totalDisasters = Array.from(typeCounts.values()).reduce(
              (sum, count) => sum + count,
              0
            );
            return {
              iso: entries[0].iso,
              totalDisasters,
              typeBreakdown: Object.fromEntries(typeCounts),
            };
          },
          (d) => d.year,
          (d) => d.country
        );

        const flatData = [];
        grouped.forEach((countries, year) => {
          countries.forEach((value, country) => {
            flatData.push({
              year,
              country,
              iso: value.iso,
              totalDisasters: value.totalDisasters,
              ...value.typeBreakdown,
            });
          });
        });

        console.log("Flat Data:", flatData);

        this.disasterCounts = flatData;

        const uniqueYears = Array.from(new Set(flatData.map((d) => d.year))).sort();
        this.years = uniqueYears;
        this.year = uniqueYears[0]; // triggers initial color scale

        // ✅ Call the new method and log result
        const yearlyTotals = this.getTotalEventsByYear();
        console.log("Total events by year:", yearlyTotals);
        this.startYearLoop();
      })
      .catch((error) => {
        console.error("Error loading CSV file:", error);
      });

      const observer = new IntersectionObserver(
    ([entry]) => {
      this.isInView = entry.isIntersecting;
      
      // Start the loop when the component comes into view
      if (this.isInView) {
        this.startYearLoop();
      } else {
        this.stopYearLoop();
      }
    },
    {
      threshold: 0.1,
    }
  );

  if (this.$refs.mapWrapper) {
    observer.observe(this.$refs.mapWrapper);
  }
  },
  
};
</script>


<style scoped>
html,
body {
  height: 100%;
  margin: 0;
}

.leaflet-container {
  background-color: #40E0D0;
}

.map-container {
  display: flex;
  justify-content: center;
  align-items: center;
  height: calc(100vh - 11rem);
  width: calc(100vw - 14rem);
}

.map-wrapper {
  position: relative;
}

.custom-slider {
  position: absolute;
  bottom: 2rem;
  left: 50%;
  transform: translateX(-50%);
  width: 95%;
  z-index: 1000;
  display: flex;
  flex-direction: column;
  align-items: center;
}

.slider-track {
  position: relative;
  width: 100%;
  height: 2px;
  background: #089c9d;
  margin-bottom: -10px;
}

.slider-markers {
  position: relative;
  display: flex;
  justify-content: space-between;
  width: 100%;
}

.slider-marker {
  display: flex;
  flex-direction: column;
  align-items: center;
  cursor: pointer;
}

.slider-marker .circle {
  width: 14px;
  height: 14px;
  border-radius: 50%;
  border: 2px solid #089c9d;
  background: #40E0D0;
  margin-bottom: 4px;
  transition: background 0.3s, border 0.3s;
}

.slider-marker.active .circle {
  background: white;
  border-color: white;
}

.slider-marker .label {
  font-size: 16px;
  font-weight: 500;
  color: #089c9d;
  transition: color 0.3s;
}

.slider-marker.active .label {
  font-weight: 800;
  color: white;
}

.event-summary-types {
  position: absolute;
  top: 32.5rem;
  left: 3rem;
  color: #2c3e50;
  text-align: left;
  font-size: 22px;
  font-weight: 400;
  max-height: 500px;
  overflow-y: auto;
  width: 500px;
  z-index: 1000;
}

.title {
  position: absolute;
  text-align: right;
  z-index: 2;
  bottom: 5.5rem;
  right: 2.5rem;
  font-size: 5em;
  font-weight: 800;
  color: #2c3e5054;
  text-transform: uppercase;
  line-height: 1;
}

.event-summary-types h4 {
  font-size: 60px;
  margin: 0;
  font-weight: 300;
  line-height: 1;
  color: #2c3e50;
}

h5 {
  padding-top: 1rem;
  text-align: left;
  font-size: 28px;
  font-weight: 300;
  margin: 0;
  line-height: 1.25;
  color: #2c3e50;
}

.event-summary-types ul {
  list-style: none;
  margin: 0;
  padding: 0;
}

.event-summary-types li {
  margin: 0.2rem 0;
}

.color-legend {
  position: absolute;
  bottom: 6.5rem;
  left: 3rem;
  width: 20%;
  display: flex;
  align-items: center;
  z-index: 1000;
  pointer-events: none;
}

.legend-gradient {
  flex: 1;
  height: 16px;
  background: linear-gradient(to right, #E9D1D5, #821749);
  border: 2px solid #089c9d;
  border-radius: 10px;
  margin: 0 0.5rem;
}

.legend-label {
  font-size: 16px;
  color: #089c9d;
  white-space: nowrap;
  font-weight: 500;
}

.legend-label.left {
  text-align: right;
}

.legend-label.right {
  text-align: left;
}

</style>
