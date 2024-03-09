<script setup lang="ts">
declare var Rectangle: any, QuadTree: any, Point: any;

window.addEventListener('DOMContentLoaded', function () {
  const canvas = document.getElementById("pl");
  const context = (canvas as HTMLCanvasElement)!.getContext("2d");
  context!.canvas!.width = window.innerWidth * 0.5;
  context!.canvas!.height = window.innerWidth * 0.5;
  let r1 = new Rectangle(0.5, 0.5, 1, 1);

  //system variables

  const n = 50000;
  const speed = 0.2;
  const fr = Math.pow(0.5, 1);
  const maxRadius = 0.01
  const m = 3;

  // const n = 20000;
  // const speed = 0.2;
  // const fr = Math.pow(0.5, 1);
  // const maxRadius = 0.01;
  // const m = 4;

  // const n = 10000;
  // const speed = 0.2;
  // const fr = Math.pow(0.5, 1);
  // const maxRadius = 0.02;
  // const m = 4;

  const matrix = createRandomMatrix();

  function createRandomMatrix() {
    const rows = [];
    for (let i = 0; i < m; i++) {
      const row = [];
      for (let j = 0; j < m; j++) {
        row.push(Math.random() * 2 - 1);
      }
      rows.push(row);
    }
    return rows;
  }

  console.log(matrix)

  //no benefit from using typed arrays

  // let colors = new Int32Array(n);
  // let positionsX = new Float32Array(n);
  // let positionsY = new Float32Array(n);
  // let velocitiesX = new Float32Array(n);
  // let velocitiesY = new Float32Array(n);

  let colors: number[] = [];
  let positionsX: any[] = [];
  let positionsY: any[] = [];
  let velocitiesX: number[] = [];
  let velocitiesY: number[] = [];

  for (let i = 0; i < n; i++) {
    colors[i] = Math.floor(Math.random() * m);
    positionsX[i] = Math.random();
    positionsY[i] = Math.random();
    velocitiesX[i] = 0;
    velocitiesY[i] = 0;
  }

  console.log(colors)

  requestAnimationFrame(renderFrame);

  async function renderFrame() {
    context!.fillStyle = "black";
    context!.fillRect(0, 0, canvas!.clientWidth, canvas!.clientHeight);

    for (let i = 0; i < n; i++) {
      context!.beginPath();
      const x = positionsX[i] * canvas!.clientWidth;
      const y = positionsY[i] * canvas!.clientHeight;

      context!.arc(x, y, 1, 0, 2 * Math.PI);
      context!.fillStyle = `hsl(${360 * (colors[i] / m)},100%,50%)`;
      context!.fill();
    }
    updateParticles();
    requestAnimationFrame(renderFrame);
  }

  function force(d: number, f: number) {
    let repRadius = 0.2;
    if (d < repRadius) {
      return d / repRadius - 1;
    } else if (repRadius < d) {
      return f * (1 - Math.abs(2 * d - 1 - repRadius) / 1 - repRadius);
    } else {
      return 0;
    }
  }

  let qt = new QuadTree(r1);
  let iteration = 0;

  for (let i = 0; i < n; i++) {
    qt.insert(new Point(positionsX[i], positionsY[i]));
  }

  console.log(qt);

  function updateParticles() {
    let t = Date.now();

    iteration++;
    // if (iteration % 4 === 0) {
    iteration = 0;
    qt = new QuadTree(r1);

    for (let i = 0; i < n; i++) {
      qt.insert(new Point(positionsX[i], positionsY[i]));
    }

    const padding = maxRadius * 150

    for (let i = 0; i < n; i++) {
      let totalForceX = 0;
      let totalForceY = 0;
      let r2 = new Rectangle(
        positionsX[i],
        positionsY[i],
        maxRadius * padding,
        maxRadius * padding,
      );
      let psInRange = qt.query(r2);

      for (let j = 0; j < psInRange.length; j++) {
        const rx = psInRange[j].x - positionsX[i];
        const ry = psInRange[j].y - positionsY[i];
        const r = Math.sqrt(rx * rx + ry * ry);

        if (r > 0 && r < maxRadius) {
          const f = force(r / maxRadius, matrix[colors[i]][colors[j]]);
          totalForceX += (rx / r) * f;
          totalForceY += (ry / r) * f;
        }
      }

      totalForceX *= maxRadius;
      totalForceY *= maxRadius;

      velocitiesX[i] *= fr;
      velocitiesY[i] *= fr;
      velocitiesX[i] += totalForceX * speed;
      velocitiesY[i] += totalForceY * speed;
    }

    for (let i = 0; i < n; i++) {
      let x = positionsX[i] + velocitiesX[i] * speed;
      let y = positionsY[i] + velocitiesY[i] * speed;


      x = x > 1 ? x - 1 : x;
      y = y > 1 ? y - 1 : y;
      x = x < 0 ? x + 1 : x;
      y = y < 0 ? y + 1 : y;

      positionsX[i] = x;
      positionsY[i] = y;
    }
    console.log(Date.now() - t);
  }
})
</script>

<template>
  <canvas id="pl" width="1000px" height="1000px"></canvas>
</template>

<style scoped></style>
