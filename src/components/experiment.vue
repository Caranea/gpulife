<script setup lang="ts">
declare var GPUBufferUsage: any;

(async () => {

  /*
  
  
  system's parameters


  */

  const m = 5;
  const workGroupSize = 1;
  const workGroupsNum = 32;
  const particleCount = workGroupSize * workGroupsNum;
  const particleSize = 1;
  const interactionRadius = 1.0;
  const axisDivisionCount = (Math.floor(2 / interactionRadius))
  const axisDivisionLength = 2 / axisDivisionCount;
  const sectorsNumber = Math.pow(axisDivisionCount, 3)

  console.log(sectorsNumber)
  console.log(axisDivisionCount, axisDivisionLength)

  /*
  
  
  setup


  */

  const adapter = await (navigator as any).gpu.requestAdapter({
    powerPreference: "high-performance"
  });
  const presentationFormat = (navigator as any).gpu.getPreferredCanvasFormat(adapter);

  const device = await adapter.requestDevice();
  const canvas = document.getElementById("webgpu-canvas") as HTMLCanvasElement;

  canvas.width = document.documentElement.clientWidth
  canvas.height = document.documentElement.clientHeight

  const screenRatio = canvas!.width / canvas!.height;

  const context = canvas!.getContext("webgpu");

  (context as any).configure({
    device,
    format: presentationFormat
  });

  /*
   
   
   sectors buffers
 
 
   */

  let positionBufferData = new Float32Array(particleCount * 4);


  let particles = new Float32Array(particleCount * 4);
  for (let i = 0; i < particles.length; i += 4) {
    particles[i] = Math.random() * 2 - 1;
    particles[i + 1] = Math.random() * 2 - 1;
    particles[i + 2] = Math.random() * 2 - 1;
    particles[i + 3] = 1;
  }

  let sectors: Float32Array[] = [];
  let currentIndexes: number[] = [];


  for (let i = 0; i < axisDivisionCount; i++) {
    for (let j = 0; j < axisDivisionCount; j++) {
      for (let k = 0; k < axisDivisionCount; k++) {
        sectors.push(new Float32Array(particleCount * 4)) // even with radius = 1.0, one sector is unlikely to hold more than 1/4 of 
        currentIndexes.push(0)
      }
    }
  }

  function createSectorsBuffers() {
    for (let i = 0; i < particles.length; i += 4) {
      determineParticlesSector([particles[i], particles[i + 1], particles[i + 2]])
    }
  }

  console.log(particles)

  function determineParticlesSector(particle: number[]) {
    for (let i = 0; i < axisDivisionCount; i++) {
      for (let j = 0; j < axisDivisionCount; j++) {
        for (let k = 0; k < axisDivisionCount; k++) {
          if ((particle[0] > (-1) + (axisDivisionLength * i) && particle[0] < (-1) + (axisDivisionLength * (i + 1))) &&
            (particle[1] > (-1) + (axisDivisionLength * j) && particle[1] < (-1) + (axisDivisionLength * (j + 1))) &&
            (particle[2] > (-1) + (axisDivisionLength * k) && particle[2] < (-1) + (axisDivisionLength * (k + 1)))) {
            const sectorsIndex = i + (j * axisDivisionCount) + (k * Math.pow(axisDivisionCount, 2))
            const index = currentIndexes[sectorsIndex]

            sectors[sectorsIndex][index] = particle[0]
            sectors[sectorsIndex][index + 1] = particle[1]
            sectors[sectorsIndex][index + 2] = particle[1]
            sectors[sectorsIndex][index + 3] = 1

            currentIndexes[sectorsIndex] += 4
          }
        }
      }
    }
  }

  createSectorsBuffers()

  const sectorsBuffers: any = []

  sectors.forEach(sector => {
    const sectorBuffer = device.createBuffer({
      size: sector.byteLength,
      usage: GPUBufferUsage.VERTEX | GPUBufferUsage.COPY_DST | GPUBufferUsage.STORAGE,
    });

    sectorsBuffers.push(sectorBuffer)

    device.queue.writeBuffer(sectorBuffer, 0, sector);
  })

  for (let i = 0; i < positionBufferData.length; i += 4) {
    positionBufferData[i] = Math.random() * 2 - 1;
    positionBufferData[i + 1] = Math.random() * 2 - 1;
    positionBufferData[i + 2] = Math.random() * 2 - 1;
    positionBufferData[i + 3] = 1;
  }

  const positionBuffer = device.createBuffer({
    size: positionBufferData.byteLength,
    usage: GPUBufferUsage.VERTEX | GPUBufferUsage.COPY_DST | GPUBufferUsage.STORAGE,
  });

  device.queue.writeBuffer(positionBuffer, 0, positionBufferData);

  /*
 
 
  velocity buffer


  */


  const velocityBufferData = new Float32Array(particleCount * 4);
  for (let i = 0; i < velocityBufferData.length; i += 4) {
    velocityBufferData[i] = 0.0;
    velocityBufferData[i + 1] = 0.0;
    velocityBufferData[i + 2] = 0.0;
    velocityBufferData[i + 3] = 0;
  }

  const velocityBuffer = device.createBuffer({
    size: velocityBufferData.byteLength,
    usage: GPUBufferUsage.VERTEX | GPUBufferUsage.COPY_DST | GPUBufferUsage.STORAGE,
  });

  device.queue.writeBuffer(velocityBuffer, 0, velocityBufferData);

  /*
 
 
  attraction buffer


  */

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

  let attractionBufferData = new Float32Array(m * m);

  for (let i = 0; i < attractionBufferData.length; i++) {
    attractionBufferData[i] = matrix.flat()[i]
  }
  console.log(attractionBufferData)

  const attractionBuffer = device.createBuffer({
    size: attractionBufferData.byteLength,
    usage: GPUBufferUsage.COPY_DST | GPUBufferUsage.STORAGE,
  });

  device.queue.writeBuffer(attractionBuffer, 0, attractionBufferData);

  /*
 
 
  compute color buffer


  */

  let computeColorsBufferData = new Int32Array(particleCount);

  for (let i = 0; i < particleCount; i++) {
    computeColorsBufferData[i] = Math.floor(Math.random() * m);
  }

  const computeColorsBuffer = device.createBuffer({
    size: computeColorsBufferData.byteLength,
    usage: GPUBufferUsage.COPY_DST | GPUBufferUsage.STORAGE,
  });


  /*
 
 
  vertex buffer


  */

  const vertexBuffer = device.createBuffer({
    size: 32,
    usage: GPUBufferUsage.VERTEX | GPUBufferUsage.COPY_DST
  });
  device.queue.writeBuffer(vertexBuffer, 0, new Float32Array([
    -1.0, -1.0,
    1.0, -1.0,
    -1.0, 1.0,
    1.0, 1.0
  ]));


  /*
 
 
   color buffer


  */

  const colorBufferData = new Uint8Array(4 * particleCount);
  for (let i = 0; i < colorBufferData.length; i += 4) {
    const rgbValues = HSLToRGB(360 * 1 / m * computeColorsBufferData[i / 4], 100, 50)
    colorBufferData[i] = rgbValues[0];
    colorBufferData[i + 1] = rgbValues[1];
    colorBufferData[i + 2] = rgbValues[2];
    colorBufferData[i + 3] = 255; //opacity
  }

  const colorBuffer = device.createBuffer({
    size: colorBufferData.byteLength,
    usage: GPUBufferUsage.VERTEX | GPUBufferUsage.COPY_DST,
  });

  device.queue.writeBuffer(colorBuffer, 0, colorBufferData);

  /*
 
 
   Rendering module


  */

  const renderShaderModule = device.createShaderModule({
    code: `
            struct VertexUniforms {
                screenDimensions: vec2f,
                particleSize: f32
            };

            struct VSOut {
                @builtin(position) clipPosition: vec4f,
                @location(0) color: vec4f
            };

            @group(0) @binding(0) var<uniform> vertexUniforms: VertexUniforms;

            @vertex
            fn vs(
                @location(0) vertexPosition: vec2f,
                @location(1) color: vec4f,
                @location(2) position: vec3f
            ) -> VSOut {
                var vsOut: VSOut;
                var adjustedPosition = position;
                adjustedPosition.y *= ${screenRatio};
                vsOut.clipPosition = vec4f(vertexPosition * vertexUniforms.particleSize / vertexUniforms.screenDimensions + adjustedPosition.xy, adjustedPosition.z, 1.0);
                var normalizedColor = color;
                normalizedColor.w *= ((position.z * 0.5 ) + 0.5);
                vsOut.color = normalizedColor;

                return vsOut;
            }             

            @fragment 
            fn fs(@location(0) color: vec4f) -> @location(0) vec4f {
                return vec4f(color.rgb * color.a, color.a);
            } 
        `
  });

  const renderPipeline = device.createRenderPipeline({
    layout: "auto",
    vertex: {
      module: renderShaderModule,
      entryPoint: "vs",
      buffers: [
        {
          arrayStride: 8,
          attributes: [{
            shaderLocation: 0,
            format: "float32x2",
            offset: 0
          }]
        },
        {
          arrayStride: 4,
          stepMode: "instance",
          attributes: [{
            shaderLocation: 1,
            format: "unorm8x4",
            offset: 0
          }]
        },
        {
          arrayStride: 16,
          stepMode: "instance",
          attributes: [{
            shaderLocation: 2,
            format: "float32x4",
            offset: 0
          }]
        }
      ]
    },
    fragment: {
      module: renderShaderModule,
      entryPoint: "fs",
      targets: [{
        format: presentationFormat,
        blend: {
          color: {
            srcFactor: "one",
            dstFactor: "one-minus-src-alpha"
          },
          alpha: {
            srcFactor: "one",
            dstFactor: "one-minus-src-alpha"
          }
        }
      }]
    },
    primitive: {
      topology: "triangle-strip",
      stripIndexFormat: "uint32"
    }
  });

  // Rendering uniform buffer

  const vertexUniformBuffer = device.createBuffer({
    size: 16,
    usage: GPUBufferUsage.UNIFORM | GPUBufferUsage.COPY_DST
  });

  device.queue.writeBuffer(vertexUniformBuffer, 0, new Float32Array([
    canvas!.width, canvas!.height, particleSize
  ]));

  const vertexUniformBindGroup = device.createBindGroup({
    layout: renderPipeline.getBindGroupLayout(0),
    entries: [{
      binding: 0,
      resource: {
        buffer: vertexUniformBuffer
      }
    }]
  });


  // Render pass description

  const renderPassDescriptor = {
    colorAttachments: [{
      view: (context as any).getCurrentTexture().createView(),
      loadOp: "clear",
      storeOp: "store",
      clearValue: [0, 0, 0, 1]
    }]
  };

  let then = 0;
  requestAnimationFrame(async function draw(now) {
    now *= 0.001;                          // convert to seconds
    const deltaTime = now - then;          // compute time since last frame
    then = now;                            // remember time for next frame
    const fps = 1 / deltaTime;
    // console.log(fps.toFixed(1) + ' fps')

    // Set up command buffer

    const commandEncoder = device.createCommandEncoder();

    renderPassDescriptor.colorAttachments[0].view = (context as any).getCurrentTexture().createView();

    // Encode render pass

    const renderPass = commandEncoder.beginRenderPass(renderPassDescriptor);
    renderPass.setPipeline(renderPipeline);

    // First argument here refers to array index
    //   in renderPipeline vertexState.vertexBuffers
    renderPass.setVertexBuffer(0, vertexBuffer);
    renderPass.setVertexBuffer(1, colorBuffer);
    renderPass.setVertexBuffer(2, positionBuffer);


    renderPass.setBindGroup(0, vertexUniformBindGroup);

    renderPass.draw(4, particleCount);
    renderPass.end();

    // Submit command buffer

    device.queue.submit([commandEncoder.finish()]);
    requestAnimationFrame(draw);
  });

  function HSLToRGB(h: number, s: number, l: number) {
    s /= 100;
    l /= 100;

    let c = (1 - Math.abs(2 * l - 1)) * s,
      x = c * (1 - Math.abs((h / 60) % 2 - 1)),
      m = l - c / 2,
      r = 0,
      g = 0,
      b = 0;

    if (0 <= h && h < 60) {
      r = c; g = x; b = 0;
    } else if (60 <= h && h < 120) {
      r = x; g = c; b = 0;
    } else if (120 <= h && h < 180) {
      r = 0; g = c; b = x;
    } else if (180 <= h && h < 240) {
      r = 0; g = x; b = c;
    } else if (240 <= h && h < 300) {
      r = x; g = 0; b = c;
    } else if (300 <= h && h < 360) {
      r = c; g = 0; b = x;
    }
    r = Math.round((r + m) * 255);
    g = Math.round((g + m) * 255);
    b = Math.round((b + m) * 255);

    return [r, g, b]
  }


  ///end
})()
</script>

<template>
  <canvas id="webgpu-canvas" class="absolute top-0 left-0 "></canvas>
</template>

