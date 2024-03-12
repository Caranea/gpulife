<script setup lang="ts">
import { defaultConfig } from '../shared/defaultConfig';
import { createRandomMatrix } from '../shared/functions';
import { onMounted } from 'vue';

onMounted(async () => {
  const canvas = document.getElementById("pl") as HTMLCanvasElement; //Grab a reference to Canvas element and cast its type as HTMLCanvasElement to have access to all its properties and methods
  const context = (canvas).getContext("2d"); //Create 2D rendering context for our canvas
  //Make our canvas full-screen
  canvas!.width = document.documentElement.clientWidth
  canvas!.height = document.documentElement.clientHeight
  //Since our x,y coordinates have values between (0,1), we need screenRatio when drawing particles, so that we don't render stretched/distorted animation.
  const screenRatio = canvas!.width / canvas!.height
  //Let's store our systems parameters in localStorage, so that they persist between user's session. If no 'parameters' object is in localStorage, fallback to default config. 
  const parameters = window.localStorage.getItem('parameters') ? JSON.parse(window.localStorage.getItem('parameters')!) : defaultConfig
  const m = parameters.colorsNumber;
  const n = parameters.particlesCountJS;
  const radius = parameters.interactionRadiusJS;
  const fr = Math.pow(0.5, 20);

  const matrix = createRandomMatrix(m);

  //using typed arrays would give us no performance advantage
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
  console.log(screenRatio)

    //Clear previous positions on every frame
    context!.fillStyle = "black";
    context!.fillRect(0, 0, canvas!.clientWidth, canvas!.clientHeight);

    for (let i = 0; i < n; i++) {
      //beginPath() method is called before drawing every particle, so that they may be filled with different colors.
      context!.beginPath();
      let x = positionsX[i] * canvas!.clientWidth;
      let y = positionsY[i] * canvas!.clientHeight
      //Adjusting position:
      y = screenRatio > 1 ? y * screenRatio : y;
      if (!(x < 1) && !(y < 1) && !(x > canvas!.clientWidth - 1) && !(y > canvas!.clientHeight - 1)) { //avoid blinking at edges due to how space wrapping is done
        //Drawing the actual particle
        context!.arc(x, y, 1, 0, 2 * Math.PI); //js compilers are smart enough for me not to have to move math.pi to var
        context!.fillStyle = `hsl(${360 * (colors[i] / m)},100%,50%)`;
        context!.fill();
      }
    }

    updateParticles();
    // Native window method to request animation frame. Note: The frequency of calls aims to match the display refresh rate.
    // This method passes DOMHighResTimeStamp as an argument to callback, so that we can calculate how much our animation should progress in one frame
    // Otherwise there will be different speed for animation on screens with different refresh rates
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
        let d = Math.sqrt(x * x + y * y); // No performance advantage in removing math.sqrt. 10x faster than math.hypot.
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

        // Prevent particless running away from the viewport. 
        // If x or y is > 1, substract 1 from it, so that it moves to other part of the screen
        // This only handles particle position - it's not interacting with 
        // particles from the other side of the screen. In other words, particle at position (0,0) with radius of 0.1
        // will not interact with particle at position (1.0,1.0). This gives us interesting visual effect (see for GPU.js examples screenshots)
        // but also causes some issues, like particles aggregating at the edges

        x = x > 1 ? x - 1 : x;
        y = y > 1 ? y - 1 : y;
        x = x < 0 ? x + 1 : x;
        y = y < 0 ? y + 1 : y;

        positionsX[i] = x;
        positionsY[i] = y;
      }
    }
  }

  //Instead of using one force function allow user to choose from several options. 
  //Graphs of every function (assuming the force value between two particles to be 1.0 and a repulsive radius of 1/4 of the max radius)
  //available on live site of the project

  function force(d: number, f: number): number {
    let repRadius = parameters.repulsiveRadius;
    switch (parameters.forceFunction) {
      case '1':
        return forceVariantI(d, f, repRadius);
      case '2':
        return forceVariantII(d, f, repRadius);
      case '3':
        return forceVariantIII(d, f, repRadius);
      default:
        return defaultForce(d, f, repRadius)
    }
  }

  function forceVariantI(d: number, f: number, repRadius: number): number {
    if (d <= repRadius) {
          return d / repRadius - 1
        } else {
          return f * (1 - Math.abs(2 * d - 1 - repRadius) / 1 - repRadius);
        }
  }

  function forceVariantII(d: number, f: number, repRadius: number): number {
    if (d <= repRadius) {
          return (d / repRadius - 1) + (f * (1 - d))
        } else {
          return f * (1 - d)
        }
  }

  function forceVariantIII(d: number, f: number, repRadius: number): number {
   return defaultForce(d,f,repRadius)
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
