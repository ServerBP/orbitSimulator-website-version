<script>
    import { onMount } from 'svelte';
    import Chart from 'chart.js/auto';
    
    let orbitCanvas;
    let distanceCanvas;
    let currentDay = 0;
    let charts = [];
    let semiMajorAxis = 1.000001018; // in AU
    let semiMinorAxis = 0.999985; // in AU
    
    $: updateOrbit(semiMajorAxis, semiMinorAxis);
    $: updatePosition(currentDay);
    
    function updatePosition(day) {
      if (charts.length === 2) {
        const orbitChart = charts[0];
        const distanceChart = charts[1];
    
        const index = Math.floor(day / 365 * orbitPoints.length);
        orbitChart.data.datasets[2].data = [orbitPoints[index]];
        orbitChart.update();
    
        distanceChart.data.datasets[2].data = [{x: day, y: distances[index]}];
        distanceChart.update();
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
    
      // Flip the fitted distances on the y-axis
      fittedDistances = t_days.map(t => -A * Math.sin(B * t + C) + D);
    
      equationText = `y = ${(-A).toFixed(6)}sin(${B.toFixed(6)}x + ${C.toFixed(6)}) + ${D.toFixed(6)}`;
    
      if (charts.length === 2) {
        charts[0].data.datasets[0].data = orbitPoints;
        charts[0].update();
    
        charts[1].data.datasets[0].data = distances.map((d, i) => ({x: t_days[i], y: d}));
        charts[1].data.datasets[1].data = fittedDistances.map((d, i) => ({x: t_days[i], y: d}));
        charts[1].update();
      }
    }
  
    function updateCurrentDate(day) {
      const date = new Date(2024, 0, 1); // Start from January 1, 2024
      date.setDate(date.getDate() + day);
      currentDate = date.toLocaleDateString('en-US', { month: 'long', day: 'numeric' });
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
              fill: false
            },
            {
              label: 'Fitted Distance',
              data: fittedDistances.map((d, i) => ({x: i * 365 / 999, y: d})),
              borderColor: '#FF5722',
              fill: false,
              borderDash: [5, 5]
            },
            {
              label: 'Current Position',
              data: [{x: 0, y: distances[0]}],
              backgroundColor: '#FFFFFF',
              pointRadius: 5
            }
          ]
        },
        options: {
          scales: {
            x: { 
              type: 'linear',
              title: { display: true, text: 'Time (days)' },
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
              text: 'Distance between Earth and Sun with Sinusoidal Fit',
              color: '#FFFFFF'
            }
          },
          onClick: (event, elements) => {
            if (elements.length > 0) {
              const clickedElement = elements[0];
              const clickedDay = Math.round(clickedElement.element.x);
              currentDay = clickedDay;
              updatePosition(clickedDay);
            }
          }
        }
      });
    });
  </script>
  
  <main>
    <h1>Earth-Sun Orbit Model</h1>
    <div class="chart-container">
      <div class="chart-box">
        <canvas bind:this={orbitCanvas}></canvas>
      </div>
      <div class="chart-box">
        <canvas bind:this={distanceCanvas}></canvas>
      </div>
    </div>
    <div class="equation-box">
      <p>Fitted equation: {equationText}</p>
    </div>
    <div class="slider-container">
      <input type="range" min="0" max="365" bind:value={currentDay} on:input={() => updatePosition(currentDay)} />
      <span>Day: {currentDay}</span>
    </div>
    <div class="input-container">
      <div class="input-box">
        <label for="semiMajorAxis">Semi-Major Axis (AU):</label>
        <input type="number" id="semiMajorAxis" bind:value={semiMajorAxis} step="0.000000001" min="0.5" max="2">
      </div>
      <div class="input-box">
        <label for="semiMinorAxis">Semi-Minor Axis (AU):</label>
        <input type="number" id="semiMinorAxis" bind:value={semiMinorAxis} step="0.000000001" min="0.5" max="2">
      </div>
    </div>
    <div class="data-box">
      <h2>Raw Data</h2>
      <p>Current Date: {currentDate}</p>
      <p>Day: {currentDay}</p>
      <p>Semi-Major Axis: {semiMajorAxis.toFixed(9)} AU</p>
      <p>Semi-Minor Axis: {semiMinorAxis.toFixed(9)} AU</p>
      <p>X position: {orbitPoints[currentDay] ? orbitPoints[currentDay].x.toFixed(6) : 'N/A'} AU</p>
      <p>Y position: {orbitPoints[currentDay] ? orbitPoints[currentDay].y.toFixed(6) : 'N/A'} AU</p>
      <p>Actual distance: {distances[currentDay] ? distances[currentDay].toFixed(6) : 'N/A'} AU</p>
      <p>Fitted distance: {fittedDistances[currentDay] ? fittedDistances[currentDay].toFixed(6) : 'N/A'} AU</p>
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
    }
  
    .input-box {
      width: 48%;
    }
  
    input[type="range"] {
      width: 80%;
      margin-right: 10px;
    }
  
    input[type="number"] {
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