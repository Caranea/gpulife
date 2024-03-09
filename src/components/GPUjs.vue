
<script setup lang="ts">
window.addEventListener('DOMContentLoaded', function () {
  const canvas = document.getElementById("pl");
  const context = (canvas as HTMLCanvasElement).getContext("2d");
  context!.canvas.width = window.innerWidth * 0.5;
  context!.canvas.height = window.innerWidth * 0.5;
  const n = 2000;
  const m = 4;
  const radius = 0.10;
  const fr = Math.pow(0.5, 20);

  const matrix = createRandomMatrix();
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
      //let-var: compiler smart enough for no performance diff
      context!.beginPath();
      const x = positionsX[i] * canvas!.clientWidth;
      const y = positionsY[i] * canvas!.clientHeight
      console.log(x, y)
      if (!(x < 1) && !(y < 1) && !(x > canvas!.clientWidth - 1) && !(y > canvas!.clientHeight - 1)) { //avoid blinking at edges
        context!.arc(x, y, 1.2, 0, 2 * Math.PI); //js compilers are smart enough for me not to have to move math.pi to var
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

  function force(d: number, f: number) {
    let repRadius = 0.2;
    if (d < repRadius) {
      return d / repRadius - 1
    } else if (repRadius < d) {
      return f * (1 - Math.abs(2 * d - 1 - repRadius) / 1 - repRadius)
    } else {
      return 0;
    }
  }

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
});

(async () => {

})()

</script>

<template>
  <canvas id="pl" width="1000px" height="1000px"></canvas>
</template>

<style scoped></style>
