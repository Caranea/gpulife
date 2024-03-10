<script setup lang="ts">
import { defaultConfig } from '../shared/defaultConfig';
import { createRandomMatrix } from '../shared/functions';
import { onMounted } from 'vue';

onMounted(async () => {
  const canvas: HTMLCanvasElement = document.getElementById("pl") as HTMLCanvasElement;
  const context = (canvas).getContext("2d");
  canvas!.width = document.documentElement.clientWidth
  canvas!.height = document.documentElement.clientHeight
  const screenRatio = canvas!.width / canvas!.height

  const parameters = window.localStorage.getItem('parameters') ? JSON.parse(window.localStorage.getItem('parameters')!) : defaultConfig
  const m = parameters.colorsNumber;
  const n = parameters.particlesCountJS;
  const radius = parameters.interactionRadiusJS;
  const fr = Math.pow(0.5, 20);

  const matrix = createRandomMatrix(m);
  let colors: number[] = [];
  let positionsX: number[] = [];
  let positionsY: number[] = [];
  let velocitiesX: any[] = [];
  let velocitiesY: any[] = [];

  for (let i = 0; i < n; i++) {
    colors[i] = Math.floor(Math.random() * m);
    positionsX[i] = Math.random();
    positionsY[i] = Math.random();
    velocitiesX[i] = 0;
    velocitiesY[i] = 0;
  }

  renderFrame();

  function renderFrame() {
    context!.fillStyle = "black";
    context!.fillRect(0, 0, canvas!.clientWidth, canvas!.clientHeight);

    for (let i = 0; i < n; i++) {
      context!.beginPath();
      let x = positionsX[i] * canvas!.clientWidth;
      let y = positionsY[i] * canvas!.clientHeight
      x = screenRatio < 1 ? x * (1 / screenRatio) : x;
      y = screenRatio > 1 ? y * screenRatio : y;
      if (!(x < 1) && !(y < 1) && !(x > canvas!.clientWidth - 1) && !(y > canvas!.clientHeight - 1)) { //avoid blinking at edges
        context!.arc(x, y, 1, 0, 2 * Math.PI); //js compilers are smart enough for me not to have to move math.pi to var
        context!.fillStyle = `hsl(${360 * (colors[i] / m)},100%,50%)`;
        context!.fill();
      }
    }

    updateParticles();
    requestAnimationFrame(renderFrame);
  }

  function updateParticles() {
    for (let i = 0; i < n; i++) {
      let totalForceX = 0;
      let totalForceY = 0;
      for (let j = 0; j < n; j++) {
        if (i === j) continue;
        let x = positionsX[j] - positionsX[i];
        let y = positionsY[j] - positionsY[i];
        let d = Math.sqrt(x * x + y * y); // no advantage in removing math.sqrt. 10x faster than math.hypot.
        if (d < radius) {
          let f = force(
            d / radius,
            matrix[colors[i]][colors[j]])
          totalForceX += (x / d) * f;
          totalForceY += (y / d) * f;
        }
        velocitiesX[i] += totalForceX;
        velocitiesY[i] += totalForceY;

        velocitiesX[i] *= fr;
        velocitiesY[i] *= fr;

        positionsX[i] += velocitiesX[i];
        positionsY[i] += velocitiesY[i];

        x = positionsX[i];
        y = positionsY[i];

        x = x > 1 ? x - 1 : x;
        y = y > 1 ? y - 1 : y;
        x = x < 0 ? x + 1 : x;
        y = y < 0 ? y + 1 : y;

        positionsX[i] = x;
        positionsY[i] = y;
      }
    }
  }

  function force(d: number, f: number): number {
    let repRadius = parameters.repulsiveRadius;
    switch (parameters.forceFunction) {
      case '1':
        if (d <= repRadius) {
          return d / repRadius - 1
        } else {
          return f * (1 - Math.abs(2 * d - 1 - repRadius) / 1 - repRadius);
        }
      case '2':
        if (d <= repRadius) {
          return (d / repRadius - 1) + (f * (1 - d))
        } else {
          return f * (1 - d)
        }
      case '3':
        return defaultForce(d, f, repRadius);
      default:
        return defaultForce(d, f, repRadius)
    }
  }

  function defaultForce(d: number, f: number, repRadius: number): number {
    if (d <= repRadius) {
      return d / repRadius - 1
    } else {
      return f * (1 - d)
    }
  }
})

</script>

<template>
  <canvas id="pl" class="fixed top-0 left-0 "></canvas>
</template>

<style scoped></style>../shared/defaultConfig
