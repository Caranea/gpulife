<script setup lang="ts">
declare var Rectangle: any, QuadTree: any, Point: any;

import { defaultConfig } from '../shared/defaultConfig';
import { createRandomMatrix } from '../shared/functions';
import { onMounted } from 'vue';

onMounted(async () => {
  const canvas = document.getElementById("pl") as HTMLCanvasElement;
  const context = (canvas)!.getContext("2d");
  canvas!.width = document.documentElement.clientWidth
  canvas!.height = document.documentElement.clientHeight
  const screenRatio = canvas!.width / canvas!.height
  let r1 = new Rectangle(0.5, 0.5, 1, 1);

  const parameters = window.localStorage.getItem('parameters') ? JSON.parse(window.localStorage.getItem('parameters')!) : defaultConfig
  const n = parameters.particlesCountQTJS;
  const speed = 0.2;
  const fr = Math.pow(0.5, 1);
  const maxRadius = parameters.interactionRadiusQTJS;
  const m = parameters.colorsNumber;

  const matrix = createRandomMatrix(m);

  //no performance benefit from using typed arrays
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

  requestAnimationFrame(renderFrame);

  async function renderFrame() {
    context!.fillStyle = "black";
    context!.fillRect(0, 0, canvas!.clientWidth, canvas!.clientHeight);

    for (let i = 0; i < n; i++) {
      context!.beginPath();
      const x = positionsX[i] * canvas!.clientWidth;
      const y = positionsY[i] * canvas!.clientHeight;

      context!.arc(x, y * screenRatio, .5, 0, 2 * Math.PI);
      context!.fillStyle = `hsl(${360 * (colors[i] / m)},100%,50%)`;
      context!.fill();
    }
    updateParticles();
    requestAnimationFrame(renderFrame);
  }

  function force(d: number, f: number) {
    let repRadius = 0.1;
    if (d < repRadius) {
      return d / repRadius - 1;
    } else if (repRadius < d) {
      return f * (1 - Math.abs(2 * d - 1 - repRadius) / 1 - repRadius);
    } else {
      return 0;
    }
  }

  let qt = new QuadTree(r1);

  for (let i = 0; i < n; i++) {
    qt.insert(new Point(positionsX[i], positionsY[i]));
  }

  function updateParticles() {
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
  }
})
</script>

<template>
  <canvas id="pl" class="absolute top-0 left-0 "></canvas>
</template>

<style scoped></style>../shared/defaultConfig
