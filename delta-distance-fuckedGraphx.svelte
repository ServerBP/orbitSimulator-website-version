<script>
  import { onMount } from 'svelte';
  import Chart from 'chart.js/auto';

  let orbitCanvas;
  let distanceCanvas;
  let deltaDistanceCanvas;
  let currentDay = 0;
  let charts = [];
  let semiMajorAxis = 1.000001018; // in AU
  let semiMinorAxis = 0.999985; // in AU
  let unit = 'AU'; // Default unit

  $: updateOrbit(semiMajorAxis, semiMinorAxis);
  $: updatePosition(currentDay);
  $: updateUnits();

  function updatePosition(day) {
    if (charts.length === 3) {
      const orbitChart = charts[0];
      const distanceChart = charts[1];
      const deltaDistanceChart = charts[2];

      const index = Math.floor(day / 365 * orbitPoints.length);
      orbitChart.data.datasets[2].data = [orbitPoints[index]];
      orbitChart.update();

      distanceChart.data.datasets[2].data = [{x: day, y: distances[index]}];
      distanceChart.update();

      deltaDistanceChart.data.datasets[0].data = distances.map((d, i) => ({
        x: i * 365 / 999,
        y: unit === 'AU' ? d - distances[0] : (d - distances[0]) * conversionFactor
      }));
      deltaDistanceChart.data.datasets[1].data = [{x: day, y: unit === 'AU' ? distances[index] - distances[0] : (distances[index] - distances[0]) * conversionFactor}];
      deltaDistanceChart.update();
    }
    updateCurrentDate(day);
  }

  let orbitPoints = [];
  let distances = [];
  let fittedDistances = [];
  let equationText = '';
  let currentDate = '';

  function updateOrbit(a, b) {
    const e = Math.sqrt(1 - (b/a)**2);
    const year_in_seconds = 365.25 * 24 * 3600;
    const t_seconds = Array.from({length: 1000}, (_, i) => i * year_in_seconds / 999);
    const t_days = t_seconds.map(t => t / (24 * 3600));

    orbitPoints = t_seconds.map(t => {
      const M = 2 * Math.PI * t / year_in_seconds;
      let E = M;
      for (let i = 0; i < 10; i++) {
        E = M + e * Math.sin(E);
      }
      const nu = 2 * Math.atan2(Math.sqrt(1 + e) * Math.sin(E / 2), Math.sqrt(1 - e) * Math.cos(E / 2));
      const r = a * (1 - e**2) / (1 + e * Math.cos(nu));
      return {
        x: r * Math.cos(nu),
        y: r * Math.sin(nu)
      };
    });

    distances = orbitPoints.map(point => Math.sqrt(point.x**2 + point.y**2));

    const A = (Math.max(...distances) - Math.min(...distances)) / 2;
    const D = (Math.max(...distances) + Math.min(...distances)) / 2;
    const B = 2 * Math.PI / 365.25;
    const C = Math.PI / 2;

    fittedDistances = t_days.map(t => -A * Math.sin(B * t + C) + D);

    equationText = `y = ${(-A).toFixed(6)}sin(${B.toFixed(6)}x + ${C.toFixed(6)}) + ${D.toFixed(6)}`;

    if (charts.length === 3) {
      charts[0].data.datasets[0].data = orbitPoints;
      charts[0].update();

      charts[1].data.datasets[0].data = distances.map((d, i) => ({x: t_days[i], y: d}));
      charts[1].data.datasets[1].data = fittedDistances.map((d, i) => ({x: t_days[i], y: d}));
      charts[1].update();

      charts[2].data.datasets[0].data = distances.map((d, i) => ({
        x: t_days[i],
        y: unit === 'AU' ? d - distances[0] : (d - distances[0]) * conversionFactor
      }));
      charts[2].update();
    }
  }

  function updateCurrentDate(day) {
    const date = new Date(2024, 0, 1); // Start from January 1, 2024
    date.setDate(date.getDate() + day);
    currentDate = date.toLocaleDateString('en-US', { month: 'long', day: 'numeric' });
  }

  const conversionFactor = 149597.8707; // 1 AU to 1000s of km

  function updateUnits() {
    if (charts.length < 3) return; // Ensure charts are initialized

    const orbitChart = charts[0];
    const distanceChart = charts[1];
    const deltaDistanceChart = charts[2];

    if (unit === 'AU') {
      semiMajorAxis /= conversionFactor;
      semiMinorAxis /= conversionFactor;
      orbitChart.options.scales.x.title.text = 'X (AU)';
      orbitChart.options.scales.y.title.text = 'Y (AU)';
      distanceChart.options.scales.y.title.text = 'Distance (AU)';
      deltaDistanceChart.options.scales.y.title.text = '∆Distance (AU)';
    } else {
      semiMajorAxis *= conversionFactor;
      semiMinorAxis *= conversionFactor;
      orbitChart.options.scales.x.title.text = 'X (1000s of km)';
      orbitChart.options.scales.y.title.text = 'Y (1000s of km)';
      distanceChart.options.scales.y.title.text = 'Distance (1000s of km)';
      deltaDistanceChart.options.scales.y.title.text = '∆Distance (1000s of km)';
    }

    orbitChart.data.datasets[0].data = orbitPoints.map(point => ({
      x: unit === 'AU' ? point.x : point.x * conversionFactor,
      y: unit === 'AU' ? point.y : point.y * conversionFactor
    }));

    distanceChart.data.datasets[0].data = distances.map((d, i) => ({
      x: i * 365 / 999,
      y: unit === 'AU' ? d : d * conversionFactor
    }));

    distanceChart.data.datasets[1].data = fittedDistances.map((d, i) => ({
      x: i * 365 / 999,
      y: unit === 'AU' ? d : d * conversionFactor
    }));

    deltaDistanceChart.data.datasets[0].data = distances.map((d, i) => ({
      x: i * 365 / 999,
      y: unit === 'AU' ? d - distances[0] : (d - distances[0]) * conversionFactor
    }));

    updatePosition(currentDay);

    orbitChart.update();
    distanceChart.update();
    deltaDistanceChart.update();

    // Update the equationText when the unit changes
    const A = (Math.max(...distances) - Math.min(...distances)) / 2;
    const D = (Math.max(...distances) + Math.min(...distances)) / 2;
    const B = 2 * Math.PI / 365.25;
    const C = Math.PI / 2;
    equationText = `y = ${(-A).toFixed(6)}sin(${B.toFixed(6)}x + ${C.toFixed(6)}) + ${D.toFixed(6)}`;
  }

  onMount(() => {
    updateOrbit(semiMajorAxis, semiMinorAxis);
    updateCurrentDate(currentDay);

    charts[0] = new Chart(orbitCanvas, {
      type: 'scatter',
      data: {
        datasets: [{
          label: 'Earth\'s Orbit',
          data: orbitPoints,
          backgroundColor: '#4CAF50'
        }, {
          label: 'Sun',
          data: [{x: 0, y: 0}],
          backgroundColor: '#FFC107',
          pointRadius: 5
        }, {
          label: 'Earth',
          data: [orbitPoints[0]],
          backgroundColor: '#2196F3',
          pointRadius: 5
        }]
      },
      options: {
        aspectRatio: 1,
        scales: {
          x: {
            title: { display: true, text: 'X (AU)' },
            ticks: { color: '#FFFFFF' }
          },
          y: {
            title: { display: true, text: 'Y (AU)' },
            ticks: { color: '#FFFFFF' }
          }
        },
        plugins: {
          legend: { labels: { color: '#FFFFFF' } },
          title: {
            display: true,
            text: '2D Orbit of Earth around the Sun',
            color: '#FFFFFF'
          }
        }
      }
    });

    charts[1] = new Chart(distanceCanvas, {
      type: 'line',
      data: {
        datasets: [
          {
            label: 'Actual Distance',
            data: distances.map((d, i) => ({x: i * 365 / 999, y: d})),
            borderColor: '#2196F3',
            backgroundColor: 'rgba(33, 150, 243, 0.5)',
            fill: true
          },
          {
            label: 'Fitted Distance',
            data: fittedDistances.map((d, i) => ({x: i * 365 / 999, y: d})),
            borderColor: '#FF5722',
            backgroundColor: 'rgba(255, 87, 34, 0.5)',
            fill: false
          },
          {
            label: 'Current Distance',
            data: [{x: 0, y: distances[0]}],
            borderColor: '#4CAF50',
            backgroundColor: '#4CAF50',
            pointRadius: 5
          }
        ]
      },
      options: {
        scales: {
          x: {
            title: { display: true, text: 'Day of the Year' },
            ticks: { color: '#FFFFFF' }
          },
          y: {
            title: { display: true, text: 'Distance (AU)' },
            ticks: { color: '#FFFFFF' }
          }
        },
        plugins: {
          legend: { labels: { color: '#FFFFFF' } },
          title: {
            display: true,
            text: 'Earth-Sun Distance over a Year',
            color: '#FFFFFF'
          }
        }
      }
    });

    charts[2] = new Chart(deltaDistanceCanvas, {
      type: 'line',
      data: {
        datasets: [
          {
            label: '∆Distance from Jan 1',
            data: distances.map((d, i) => ({
              x: i * 365 / 999,
              y: unit === 'AU' ? d - distances[0] : (d - distances[0]) * conversionFactor
            })),
            borderColor: '#FFEB3B',
            backgroundColor: 'rgba(255, 235, 59, 0.5)',
            fill: true
          },
          {
            label: 'Current ∆Distance',
            data: [{x: 0, y: 0}],
            borderColor: '#FF9800',
            backgroundColor: '#FF9800',
            pointRadius: 5
          }
        ]
      },
      options: {
        scales: {
          x: {
            title: { display: true, text: 'Day of the Year' },
            ticks: { color: '#FFFFFF' }
          },
          y: {
            title: { display: true, text: '∆Distance (AU)' },
            ticks: { color: '#FFFFFF' }
          }
        },
        plugins: {
          legend: { labels: { color: '#FFFFFF' } },
          title: {
            display: true,
            text: 'Change in Distance from the Sun since Jan 1',
            color: '#FFFFFF'
          }
        }
      }
    });
  });
</script>

<main>
  <h1>Orbital Simulation</h1>
  <div class="chart-container">
    <div class="chart-box">
      <canvas bind:this={orbitCanvas}></canvas>
    </div>
    <div class="chart-box">
      <canvas bind:this={distanceCanvas}></canvas>
      <canvas bind:this={deltaDistanceCanvas}></canvas>
    </div>
  </div>
  <div class="slider-container">
    <input type="range" min="0" max="365" step="1" bind:value={currentDay} on:input={() => updatePosition(currentDay)} />
    <span>Day: {currentDay}</span>
  </div>
  <div class="input-container">
    <div class="input-box">
      <label for="semiMajorAxis">Semi-Major Axis:</label>
      <input type="number" id="semiMajorAxis" bind:value={semiMajorAxis} step="0.000000001" min="0.5" max="2">
    </div>
    <div class="input-box">
      <label for="semiMinorAxis">Semi-Minor Axis:</label>
      <input type="number" id="semiMinorAxis" bind:value={semiMinorAxis} step="0.000000001" min="0.5" max="2">
    </div>
    <div class="input-box">
      <label for="unit">Select Unit:</label>
      <select id="unit" bind:value={unit} on:change={updateUnits}>
        <option value="AU">Astronomical Units (AU)</option>
        <option value="km">1000s of Kilometers</option>
      </select>
    </div>
  </div>
  <div class="data-box">
    <h2>Raw Data</h2>
    <p>Current Date: {currentDate}</p>
    <p>Day: {currentDay}</p>
    <p>Semi-Major Axis: {unit === 'AU' ? semiMajorAxis.toFixed(9) + ' AU' : (semiMajorAxis * conversionFactor).toFixed(3) + ' x 1000 km'}</p>
    <p>Semi-Minor Axis: {unit === 'AU' ? semiMinorAxis.toFixed(9) + ' AU' : (semiMinorAxis * conversionFactor).toFixed(3) + ' x 1000 km'}</p>
    <p>X position: {orbitPoints[currentDay] ? (unit === 'AU' ? orbitPoints[currentDay].x.toFixed(6) + ' AU' : (orbitPoints[currentDay].x * conversionFactor).toFixed(3) + ' x 1000 km') : 'N/A'}</p>
    <p>Y position: {orbitPoints[currentDay] ? (unit === 'AU' ? orbitPoints[currentDay].y.toFixed(6) + ' AU' : (orbitPoints[currentDay].y * conversionFactor).toFixed(3) + ' x 1000 km') : 'N/A'}</p>
    <p>Actual distance: {distances[currentDay] ? (unit === 'AU' ? distances[currentDay].toFixed(6) + ' AU' : (distances[currentDay] * conversionFactor).toFixed(3) + ' x 1000 km') : 'N/A'}</p>
    <p>Fitted distance: {fittedDistances[currentDay] ? (unit === 'AU' ? fittedDistances[currentDay].toFixed(6) + ' AU' : (fittedDistances[currentDay] * conversionFactor).toFixed(3) + ' x 1000 km') : 'N/A'}</p>
    <p>Equation: {equationText}</p>
  </div>
</main>

<style>
  :global(body) {
    background-color: #121212;
    color: #FFFFFF;
    font-family: Arial, sans-serif;
  }

  main {
    max-width: 1200px;
    margin: 0 auto;
    padding: 20px;
  }

  h1, h2 {
    text-align: center;
    margin-bottom: 30px;
  }

  .chart-container {
    display: flex;
    justify-content: space-between;
    gap: 20px; /* Add spacing between the chart boxes */
    margin-bottom: 30px;
  }

  .chart-box, .data-box, .equation-box, .input-box, .slider-container {
    background-color: #1E1E1E;
    border-radius: 10px;
    padding: 20px;
    box-shadow: 0 4px 6px rgba(0, 0, 0, 0.1);
    margin-bottom: 20px;
  }

  .chart-box {
    width: 48%;
  }

  .slider-container, .input-container {
    display: flex;
    align-items: center;
    justify-content: center;
  }

  .input-container {
    justify-content: space-between;
    gap: 20px; /* Add spacing between the input boxes */
  }

  .input-box {
    width: 30%;
  }

  input[type="range"] {
    width: 80%;
    margin-right: 10px;
  }

  input[type="number"], select {
    width: 100%;
    padding: 5px;
    margin-top: 5px;
    background-color: #333;
    color: #FFF;
    border: 1px solid #555;
    border-radius: 5px;
  }

  label {
    display: block;
    margin-bottom: 5px;
  }
</style>
