<script>
  import mapboxgl from "mapbox-gl";
  import "../../node_modules/mapbox-gl/dist/mapbox-gl.css";
  import { onMount } from "svelte";
  import * as d3 from "d3";
  //   import { MAPBOX_ACCESS_TOKEN } from "../../config.js";

  let stations = [];
  let trips = [];
  let arrivals;
  let filteredArrivals = new Map();
  let filteredDepartures = new Map();
  let filteredStations = [];
  let departures;
  let map;
  let radiusScale;
  let mapViewChanged = 0;
  let timeFilter = -1;
  let timeFilterLabel;
  let departuresByMinute = Array.from({ length: 1440 }, () => []);
  let arrivalsByMinute = Array.from({ length: 1440 }, () => []);

  $: timeFilterLabel = new Date(0, 0, 0, 0, timeFilter).toLocaleString("en", {
    timeStyle: "short",
  });

  $: map?.on("move", (evt) => mapViewChanged++);

  let stationFlow = d3.scaleQuantize().domain([0, 1]).range([0, 0.5, 1]);
  $: radiusScale = d3
    .scaleSqrt()
    .domain([0, d3.max(stations, (d) => d.totalTraffic)])
    .range([0, 25]);

  $: {
    filteredDepartures = d3.rollup(
      filterByMinute(departuresByMinute, timeFilter),
      (v) => v.length,
      (d) => d.start_station_id
    );

    filteredArrivals = d3.rollup(
      filterByMinute(arrivalsByMinute, timeFilter),
      (v) => v.length,
      (d) => d.end_station_id
    );

    filteredStations = {
      ...stations.map((station) => {
        let id = station.Number;
        station.arrivals = filteredArrivals.get(id) ?? 0;
        station.departures = filteredDepartures.get(id) ?? 0;
        station.totalTraffic = station.arrivals + station.departures;
        return station;
      }),
    };
  }

  mapboxgl.accessToken =
    "pk.eyJ1IjoiZWZhaXRoMSIsImEiOiJjbHVwM3hqbngxejEwMmlxcHZoMnd4NzVoIn0.aImOljzGu-9EUSa9aFcQzw";

  onMount(async () => {
    stations = await d3.csv(
      "https://vis-society.github.io/labs/8/data/bluebikes-stations.csv"
    );

    trips = await d3
      .csv(
        "https://vis-society.github.io/labs/8/data/bluebikes-traffic-2024-03.csv"
      )
      .then((trips) => {
        for (let trip of trips) {
          let temp_start = trip.started_at;
          let [datePart, timePart] = temp_start.split(" ");
          let [year, month, day] = datePart.split("-");
          let [hour, minute, second] = timePart.split(":");
          trip.started_at = new Date(
            year,
            month - 1,
            day,
            hour,
            minute,
            second
          );

          let temp_end = trip.ended_at;
          let [endDatePart, endTimePart] = temp_end.split(" ");
          let [y, m, d] = endDatePart.split("-");
          let [h, min, sec] = endTimePart.split(":");
          trip.ended_at = new Date(y, m - 1, d, h, min, sec);

          let startedMinutes = minutesSinceMidnight(trip.started_at);
          departuresByMinute[startedMinutes].push(trip);

          let endedMinutes = minutesSinceMidnight(trip.ended_at);
          arrivalsByMinute[endedMinutes].push(trip);
        }
        return trips;
      });

    departures = d3.rollup(
      trips,
      (v) => v.length,
      (d) => d.start_station_id
    );

    arrivals = d3.rollup(
      trips,
      (v) => v.length,
      (d) => d.end_station_id
    );

    stations = stations.map((station) => {
      let id = station.Number;
      station.arrivals = arrivals.get(id) ?? 0;
      station.departures = departures.get(id) ?? 0;
      station.totalTraffic = station.arrivals + station.departures;
      return station;
    });

    const container = "map";

    const style = "mapbox://styles/mapbox/streets-v12";

    const center = [-71.0921, 42.3601];
    const zoom = 12;
    const minZoom = 1;
    const maxZoom = 20;

    map = new mapboxgl.Map({
      container: container,
      style: style,
      center: center,
      zoom: zoom,
      minZoom: minZoom,
      maxZoom: maxZoom,
    });

    await new Promise((resolve) => map.on("load", resolve));

    map.addSource("boston_route", {
      type: "geojson",
      data: "https://bostonopendata-boston.opendata.arcgis.com/datasets/boston::existing-bike-network-2022.geojson?outSR=%7B%22latestWkid%22%3A3857%2C%22wkid%22%3A102100%7D",
    });
    map.addSource("cambridge_bike_lanes", {
      type: "geojson",
      data: "https://raw.githubusercontent.com/cambridgegis/cambridgegis_data/main/Recreation/Bike_Facilities/RECREATION_BikeFacilities.geojson",
    });

    map.addLayer({
      id: "boston_route_layer",
      type: "line",
      source: "boston_route",
      paint: {
        "line-color": "rgba(0, 128, 0, 0.4)",
        "line-width": 3,
      },
    });

    map.addLayer({
      id: "cambridge_bike_lanes_layer",
      type: "line",
      source: "cambridge_bike_lanes",
      paint: {
        "line-color": "rgba(0, 128, 0, 0.4)",
        "line-width": 3,
      },
    });
  });

  function getCoords(station) {
    let point = new mapboxgl.LngLat(+station.Long, +station.Lat);
    let { x, y } = map.project(point);
    return { cx: x, cy: y };
  }

  function minutesSinceMidnight(date) {
    return date.getHours() * 60 + date.getMinutes();
  }

  function filterByMinute(tripsByMinute, minute) {
    let minMinute = (minute - 60 + 1440) % 1440;
    let maxMinute = minute + (60 % 1440);

    if (minMinute > maxMinute) {
      let beforeMidnight = tripsByMinute.slice(minMinute);
      let afterMidnight = tripsByMinute.slice(0, maxMinute);
      return beforeMidnight.concat(afterMidnight).flat();
    } else {
      return tripsByMinute.slice(minMinute, maxMinute).flat();
    }
  }
</script>

<h1>Bluebikes 🚴🏽‍♀️</h1>

<header style="display: flex; gap: 1em; align-items: baseline;">
  <div
    style=" margin-left: auto; display: flex; flex-direction: column; align-items: center;"
  >
    <label
      >Filter by time:
      <input
        type="range"
        id="time-slider"
        min="-1"
        max="1440"
        bind:value={timeFilter}
      />
    </label>

    {#if timeFilter === -1}
      <em style="color: #999; font-style: italic;">(any time)</em>
    {:else}
      <time id="selected-time">({timeFilterLabel})</time>
    {/if}
  </div>
</header>

<div id="map">
  <svg>
    {#each stations as station}
      {#key mapViewChanged}
        <circle
          style="--departure-ratio: {stationFlow(
            station.departures / station.totalTraffic
          )}"
          {...getCoords(station)}
          r={radiusScale(station.totalTraffic)}
        >
          <title
            >{station.totalTraffic} trips ({station.departures} departures, {station.arrivals}
            arrivals)</title
          >
        </circle>
      {/key}
    {/each}
  </svg>
</div>

<div class="legend">
  <div style="--departure-ratio: 1">More departures</div>
  <div style="--departure-ratio: 0.5">Balanced</div>
  <div style="--departure-ratio: 0">More arrivals</div>
</div>

<style>
  @import url("$lib/global.css");

  circle,
  .legend > div {
    --color-departures: steelblue;
    --color-arrivals: darkorange;
    --color: color-mix(
      in oklch,
      var(--color-departures) calc(100% * var(--departure-ratio)),
      var(--color-arrivals)
    );
    fill: var(--color);
    background-color: var(--color);
  }

  #map {
    flex: 1;
  }
  #map svg {
    position: absolute;
    z-index: 1;
    width: 100%;
    height: 100%;
    pointer-events: none;
  }

  #map svg circle {
    fill-opacity: 0.6;
    stroke: white;
    stroke-width: 1;
  }

  circle {
    pointer-events: auto;
  }
  header label {
    display: block;
  }

  #any-time {
    color: #999;
    font-style: italic;
  }

  .legend {
    display: flex;
    justify-content: space-between;
    padding: 10px 20px;
    gap: 1px;
  }
  .legend > div {
    flex: 1;
    text-align: center;
    padding: 8px 0;
  }
</style>
