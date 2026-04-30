<html lang="en">
<head>
  <meta charset="utf-8" />
  <title>Sensor Time Series</title>

  <!-- Vega stack -->
  <script src="https://cdn.jsdelivr.net/npm/vega@5"></script>
  <script src="https://cdn.jsdelivr.net/npm/vega-lite@5"></script>
  <script src="https://cdn.jsdelivr.net/npm/vega-embed@6"></script>

  <style>
    body {
      margin: 0;
      padding: 12px;
      font-family: Arial, sans-serif;
    }

    .controls {
      display: flex;
      flex-wrap: wrap;
      gap: 12px;
      margin-bottom: 10px;
    }

    label {
      font-size: 0.85rem;
    }

    select, input {
      font-size: 0.85rem;
      padding: 3px;
    }
  </style>
</head>

<body>

<div class="controls">
  <label>
    Location:
    <select id="location"></select>
  </label>

  <label>
    Sensor:
    <select id="sensor"></select>
  </label>

  <label>
    Start date:
    <input type="date" id="startDate">
  </label>

  <label>
    End date:
    <input type="date" id="endDate">
  </label>
</div>

<div id="chart"></div>

<script>
// ===============================
// CONFIG
// ===============================
const FEATURE_LAYER_URL = "https://utility.arcgis.com/usrsvcs/servers/e1f8cd3ce3e644968019ccc951d5b678/rest/services/R1Removals/Riverside_PCB_Viper/FeatureServer/6?f=json";

// ===============================
// UTILITIES
// ===============================
async function fetchDistinct(field) {
  const params = new URLSearchParams({
    where: "1=1",
    outFields: field,
    returnDistinctValues: true,
    f: "json"
  });

  const res = await fetch(`${FEATURE_LAYER_URL}/query?${params}`);
  const json = await res.json();
  return json.features.map(f => f.attributes[field]).sort();
}

// ===============================
// INITIALIZE CONTROLS
// ===============================
async function initControls() {
  const locations = await fetchDistinct("RUNID_DM");
  const sensors   = await fetchDistinct("SensorName");

  const locSel = document.getElementById("location");
  const senSel = document.getElementById("sensor");

  locations.forEach(v => locSel.add(new Option(v, v)));
  sensors.forEach(v => senSel.add(new Option(v, v)));

  locSel.onchange =
  senSel.onchange =
  startDate.onchange =
  endDate.onchange = drawChart;

  drawChart();
}

// ===============================
// BUILD VEGA-LITE SPEC
// ===============================
function buildSpec() {
  const location = locationSelect.value || location.value;
  const sensor   = sensorSelect.value   || sensor.value;

  const start = startDate.value
    ? `ReceivedDateTime >= DATE '${startDate.value}'`
    : "1=1";

  const end = endDate.value
    ? `ReceivedDateTime <= DATE '${endDate.value}'`
    : "1=1";

  return {
    $schema: "https://vega.github.io/schema/vega-lite/v5.json",
    width: "container",
    height: 350,

    data: {
      url: `${FEATURE_LAYER_URL}/query`,
      format: { type: "json", property: "features" },
      params: {
        where: `
          RUNID_DM = '${location}'
          AND SensorName = '${sensor}'
          AND ${start}
          AND ${end}
        `,
        outFields: "ReceivedDateTime,SensorReading,SensorUnits",
        orderByFields: "ReceivedDateTime",
        f: "json"
      }
    },

    transform: [
      {
        calculate: "toDate(datum.attributes.ReceivedDateTime)",
        as: "ReceivedDateTime"
      },
      {
        calculate: "datum.attributes.SensorReading",
        as: "SensorReading"
      },
      {
        calculate: "datum.attributes.SensorUnits",
        as: "SensorUnits"
      },
      {
        timeUnit: "yearmonthdate",
        field: "ReceivedDateTime",
        as: "date"
      },
      {
        aggregate: [
          { op: "mean", field: "SensorReading", as: "value" }
        ],
        groupby: ["date", "SensorUnits"]
      }
    ],

    mark: {
      type: "line",
      point: { filled: true, size: 45 }
    },

    encoding: {
      x: {
        field: "date",
        type: "temporal",
        title: "Date"
      },
      y: {
        field: "value",
        type: "quantitative",
        title: "Mean Sensor Reading",
        axis: {
          format: ".2f",
          tickCount: 6
        }
      },
      tooltip: [
        { field: "date", type: "temporal", title: "Date" },
        { field: "value", type: "quantitative", format: ".2f", title: "Mean" },
        { field: "SensorUnits", type: "nominal", title: "Units" }
      ]
    }
  };
}

// ===============================
// RENDER
// ===============================
async function drawChart() {
  const spec = buildSpec();
  vegaEmbed("#chart", spec, { actions: false });
}

// ===============================
initControls();
</script>

</body>
</html>
