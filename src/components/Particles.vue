<script setup lang="ts">
declare var GPUBufferUsage: any;

import { onMounted, ref } from 'vue';
import { HSLToRGB, createRandomMatrix, fix } from '../shared/functions.ts'
import Loader from './Loader.vue'
import { defaultConfig } from '../shared/defaultConfig.ts';

const emit = defineEmits(['fps'])
const text = ref('Loading...')
const parameters = window.localStorage.getItem('parameters') ? JSON.parse(window.localStorage.getItem('parameters')!) : defaultConfig

const m = parameters.colorsNumber;
const workGroupSize = 64;
const workGroupsNum = Math.floor(parameters.particlesCountGPU / 64);
const particleCount = workGroupSize * workGroupsNum;
const particleSize = 1;
const interactionRadius = parameters.interactionRadius * 0.9999;
const axisDivisionCount = (Math.floor(2 / interactionRadius))
const axisDivisionLength = 2 / axisDivisionCount;
const sectorsNumber = Math.pow(axisDivisionCount, 3);
const helperConst = interactionRadius > 1.0 ? interactionRadius > 1.5 ? 100 : 50 : interactionRadius > 0.5 ? 100 : 50
const forceFactor = fix(interactionRadius * helperConst * Math.pow(10, (1.25 * interactionRadius)));
console.log(forceFactor);

onMounted(async () => {
  const matrix = createRandomMatrix(m);

  //Set up WebGPU adapter - before deploying, we need to make sure users with browsers
  //not supporting WebGPU receive message

  const adapter = await (navigator as any).gpu.requestAdapter({
    powerPreference: "high-performance"
  });
  const presentationFormat = (navigator as any).gpu.getPreferredCanvasFormat(adapter);
  const device = await adapter.requestDevice();
  const canvas = document.getElementById("webgpu-canvas") as HTMLCanvasElement;

  canvas!.width = document.documentElement.clientWidth
  canvas!.height = document.documentElement.clientHeight

  const screenRatio = canvas!.width / canvas!.height
  const context = canvas!.getContext("webgpu");

  (context! as any).configure({
    device,
    format: presentationFormat
  });

  text.value = 'Preparing position buffer...'

  let positionBufferData = new Float32Array(particleCount * 4);
  for (let i = 0; i < positionBufferData.length; i += 4) {
    positionBufferData[i] = Math.random() * 2 - 1;
    positionBufferData[i + 1] = Math.random() * 2 - 1;
    positionBufferData[i + 2] = Math.random() * 2 - 1;
    positionBufferData[i + 3] = 1;
  }

  let sectors: any[] = [];
  let currentIndexes: number[] = [];

  for (let i = 0; i < axisDivisionCount; i++) {
    for (let j = 0; j < axisDivisionCount; j++) {
      for (let k = 0; k < axisDivisionCount; k++) {
        sectors.push([])
        currentIndexes.push(0)
      }
    }
  }

  text.value = 'Sorting position buffer...'

  for (let i = 0; i < positionBufferData.length; i += 4) {
    determineParticlesSector([positionBufferData[i], positionBufferData[i + 1], positionBufferData[i + 2]])
  }

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
            sectors[sectorsIndex][index + 2] = particle[2]
            sectors[sectorsIndex][index + 3] = 1

            currentIndexes[sectorsIndex] += 4
          }
        }
      }
    }
  }

  let currentIndex = 0
  let firstIndexes = new Uint32Array(sectorsNumber)

  for (let i = 0; i < sectors.length; i++) {
    if (0 === i) {
      firstIndexes[i] = 0;
    } else {
      firstIndexes[i] = (firstIndexes[i - 1] + sectors[i].length / 4);
    }
    for (let j = 0; j < sectors[i].length; j++) {
      positionBufferData[currentIndex] = (sectors[i][j]);
      currentIndex++
    }
  }

  text.value = 'Writing position buffer...'

  const positionBuffer = device.createBuffer({
    size: positionBufferData.byteLength,
    usage: GPUBufferUsage.VERTEX | GPUBufferUsage.COPY_DST | GPUBufferUsage.STORAGE,
  });

  device.queue.writeBuffer(positionBuffer, 0, positionBufferData);

  text.value = 'Preparing remaining buffers...'

  const indexesBuffer = device.createBuffer({
    size: firstIndexes.byteLength,
    usage: GPUBufferUsage.COPY_DST | GPUBufferUsage.STORAGE,
  });

  device.queue.writeBuffer(indexesBuffer, 0, firstIndexes);

  let computeColorsBufferData = new Int32Array(particleCount);
  for (let i = 0; i < positionBufferData.length; i++) {
    computeColorsBufferData[i] = Math.floor(Math.random() * m);
  }

  const computeColorsBuffer = device.createBuffer({
    size: computeColorsBufferData.byteLength,
    usage: GPUBufferUsage.COPY_DST | GPUBufferUsage.STORAGE,
  });

  device.queue.writeBuffer(computeColorsBuffer, 0, computeColorsBufferData);

  let attractionBufferData = new Float32Array(m * m);

  for (let i = 0; i < attractionBufferData.length; i++) {
    attractionBufferData[i] = matrix.flat()[i]
  }

  const attractionBuffer = device.createBuffer({
    size: attractionBufferData.byteLength,
    usage: GPUBufferUsage.COPY_DST | GPUBufferUsage.STORAGE,
  });

  device.queue.writeBuffer(attractionBuffer, 0, attractionBufferData);

  const velocityBuffer = device.createBuffer({
    size: 16 * particleCount,
    usage: GPUBufferUsage.VERTEX | GPUBufferUsage.COPY_DST | GPUBufferUsage.STORAGE,
  });

  const velocityBufferData = new Float32Array(particleCount * 4);
  for (let i = 0; i < velocityBufferData.length; i += 4) {
    velocityBufferData[i] = 0.0;
    velocityBufferData[i + 1] = 0.0;
    velocityBufferData[i + 2] = 0.0;
    velocityBufferData[i + 3] = 0;
  }

  device.queue.writeBuffer(velocityBuffer, 0, velocityBufferData);

  const computeShaderModule = device.createShaderModule({
    code: `
            @group(0) @binding(0) var<storage, read_write> positions: array<vec4f>;
            @group(0) @binding(1) var<storage, read_write> velocities: array<vec4f>; 
            @group(0) @binding(2) var<storage> colors: array<i32>; 
            @group(0) @binding(3) var<storage> attraction: array<f32>; 
            @group(0) @binding(4) var<storage> indexes: array<u32>; 

            @compute @workgroup_size(${workGroupSize})
            fn cs(@builtin(global_invocation_id) global_id: vec3u) { 
                let index = global_id.x;
                var position = positions[index].xyz;
                var velocity = velocities[index].xyz;
                var sectorsStartIndex: u32;
                var sectorsEndIndex: u32;

                for(var i: i32 = 0; i < ${firstIndexes.length}; i+=1) {
                  if (i <= ${firstIndexes.length - 2}) {
                    if (index > indexes[i] && index < indexes[i+1]) {
                      sectorsStartIndex = indexes[i];
                      sectorsEndIndex = indexes[i+1];
                    }
                  } else {
                    if (index > indexes[i] && index < ${particleCount}) {
                      sectorsStartIndex = indexes[i];
                      sectorsEndIndex = ${particleCount};
                    }
                  }
                }

                var forcex = 0.0;
                var forcey = 0.0;
                var forcez = 0.0;

                let interactionRadius = ${interactionRadius};
                let forceFactor = ${interactionRadius}/${forceFactor};

                for (var i = sectorsStartIndex;i < sectorsEndIndex; i++) {
                  let xdiff = positions[i].x - position.x;
                  let ydiff = positions[i].y - position.y;
                  let zdiff = positions[i].z - position.z;
                  
                  let dist = sqrt((xdiff*xdiff) + (ydiff*ydiff) + (zdiff*zdiff));

                  if (dist < interactionRadius) {
                    var forceV = attraction[(${m}*colors[index])+colors[i]];
                    let ratio = dist/interactionRadius;
                    let repF = 0.2;

                    var force: f32;

                    if (ratio > repF) {
                      force =  forceV * (1.0 - ratio) * forceFactor;
                      forcex += xdiff*force;
                      forcey += ydiff*force;
                      forcez += zdiff*force;
                    } else if (ratio <= repF) {
                      force =  forceV * (1.0 - ratio) * forceFactor  ;
                      forcex -= xdiff*force;
                      forcey -= ydiff*force;
                      forcez -= zdiff*force;
                    }
                  }
                }

                velocity.x+=forcex;
                velocity.y+=forcey;
                velocity.z+=forcez;

                velocity*=0.3; // friction

                positions[index] = vec4f(position + velocity, 1);
                velocities[index] = vec4f(velocity, 0);
            }

        `
  })

  const computePipeline = device.createComputePipeline({
    layout: "auto",
    compute: {
      module: computeShaderModule,
      entryPoint: "cs"
    }
  });

  const computeBindGroup = device.createBindGroup({
    layout: computePipeline.getBindGroupLayout(0),
    entries: [
      {
        binding: 0,
        resource: {
          buffer: positionBuffer
        }
      },
      {
        binding: 1,
        resource: {
          buffer: velocityBuffer
        }
      },
      {
        binding: 2,
        resource: {
          buffer: computeColorsBuffer
        }
      },
      {
        binding: 3,
        resource: {
          buffer: attractionBuffer
        }
      },
      {
        binding: 4,
        resource: {
          buffer: indexesBuffer
        }
      }
    ]
  });

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

  const colorBufferData = new Uint8Array(4 * particleCount);
  for (let i = 0; i < colorBufferData.length; i += 4) {
    const rgbValues = HSLToRGB(360 * 1 / m * computeColorsBufferData[i / 4], 100, 50)
    colorBufferData[i] = rgbValues[0];
    colorBufferData[i + 1] = rgbValues[1];
    colorBufferData[i + 2] = rgbValues[2];
    colorBufferData[i + 3] = Math.floor(parameters.opacity / 100 * 255);
  }

  const colorBuffer = device.createBuffer({
    size: colorBufferData.byteLength,
    usage: GPUBufferUsage.VERTEX | GPUBufferUsage.COPY_DST,
  });

  device.queue.writeBuffer(colorBuffer, 0, colorBufferData);

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
                var adjustedP = position;
                adjustedP.y *= ${screenRatio};
                vsOut.clipPosition = vec4f(vertexPosition * vertexUniforms.particleSize / vertexUniforms.screenDimensions + adjustedP.xy, adjustedP.z, 1.0);
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

  const renderPassDescriptor = {
    colorAttachments: [{
      view: (context! as any).getCurrentTexture().createView(),
      loadOp: "clear",
      storeOp: "store",
      clearValue: [0, 0, 0, 1]
    }]
  };

  let then = 0;
  requestAnimationFrame(async function draw(now) {
    now *= 0.001;
    const dt = now - then;
    then = now;
    emit('fps', (1 / dt).toFixed(1) + ' fps');

    const commandEncoder = device.createCommandEncoder();
    const computePass = commandEncoder.beginComputePass();

    computePass.setPipeline(computePipeline);
    computePass.setBindGroup(0, computeBindGroup);
    computePass.dispatchWorkgroups(workGroupsNum);
    computePass.end();

    renderPassDescriptor.colorAttachments[0].view = (context! as any).getCurrentTexture().createView();
    const renderPass = commandEncoder.beginRenderPass(renderPassDescriptor);
    renderPass.setPipeline(renderPipeline);
    renderPass.setVertexBuffer(0, vertexBuffer);
    renderPass.setVertexBuffer(1, colorBuffer);
    renderPass.setVertexBuffer(2, positionBuffer);
    renderPass.setBindGroup(0, vertexUniformBindGroup);

    renderPass.draw(4, particleCount);
    renderPass.end();

    device.queue.submit([commandEncoder.finish()]);
    requestAnimationFrame(draw);
  });

});

</script>

<template>
  <Loader :text="text" />
  <canvas id="webgpu-canvas" class="absolute top-0 left-0 "></canvas>
</template>../shared/defaultConfig.ts
