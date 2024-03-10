<script setup lang="ts">
declare var GPUBufferUsage: any;
import { onMounted, ref } from 'vue';
import Loader from './Loader.vue'
import { HSLToRGB } from '../shared/functions';

const NUM_COLORS = 1
const WORKGROUP_SIZE = 64;
const NUM_WORKGROUPS = 500;
const NUM_PARTICLES = WORKGROUP_SIZE * NUM_WORKGROUPS;
const PARTICLE_SIZE = 2;
const text = ref('Loading...')

onMounted(async () => {
  let adapter;
  let presentationFormat;
  let device;
  try {
    adapter = await (navigator as any).gpu.requestAdapter({
      powerPreference: "high-performance"
    });
    presentationFormat = (navigator as any).gpu.getPreferredCanvasFormat(adapter);
    device = await adapter.requestDevice();
  } catch (e) {
    text.value = 'Your browser does not support webGPU. Use different browser, enable webGPU, or switch to non-gpu simulation.'
  }

  ////////////////////////////////////
  // Set up device and canvas context
  ////////////////////////////////////

  const canvas = document.getElementById("webgpu-canvas") as HTMLCanvasElement;

  canvas!.width = document.documentElement.clientWidth
  canvas!.height = document.documentElement.clientHeight

  const screenRatio = canvas!.width / canvas!.height
  const context = canvas!.getContext("webgpu");

  (context as any)!.configure({
    device,
    format: presentationFormat
  });

  ////////////////////////////////////////////////////////
  // Create buffers for compute pass
  // (positionBuffer also used in render pass)
  ////////////////////////////////////////////////////////

  const positionBuffer = device.createBuffer({
    size: 16 * NUM_PARTICLES,
    usage: GPUBufferUsage.VERTEX | GPUBufferUsage.COPY_DST | GPUBufferUsage.STORAGE,
  });

  let positionBufferData = new Float32Array(NUM_PARTICLES * 4);
  for (let i = 0; i < positionBufferData.length; i += 4) {
    positionBufferData[i] = Math.random() * 2 - 1;
    positionBufferData[i + 1] = Math.random() * 2 - 1;
    positionBufferData[i + 2] = Math.random() * 2 - 1;
    positionBufferData[i + 3] = 1;
  }

  device.queue.writeBuffer(positionBuffer, 0, positionBufferData);

  const velocityBuffer = device.createBuffer({
    size: 16 * NUM_PARTICLES,
    usage: GPUBufferUsage.VERTEX | GPUBufferUsage.COPY_DST | GPUBufferUsage.STORAGE,
  });
  const velocityBufferData = new Float32Array(NUM_PARTICLES * 4);
  for (let i = 0; i < velocityBufferData.length; i += 4) {
    velocityBufferData[i] = 0.0;
    velocityBufferData[i + 1] = 0.0;
    velocityBufferData[i + 2] = 0.0;
    velocityBufferData[i + 3] = 0;
  }
  device.queue.writeBuffer(velocityBuffer, 0, velocityBufferData);

  ///////////////////////////////////
  // Create compute shader module
  //////////////////////////////////

  const computeShaderModule = device.createShaderModule({
    code: `
            @group(0) @binding(0) var<storage, read_write> positions: array<vec4f>;
            @group(0) @binding(1) var<storage, read_write> velocities: array<vec4f>; 

            @compute @workgroup_size(${WORKGROUP_SIZE})
            fn cs(@builtin(global_invocation_id) global_id: vec3u) {
                let index = global_id.x;
                var position = positions[index].xyz;
                var velocity = velocities[index].xyz;

                var forcex = 0.0;
                var forcey = 0.0;
                var forcez = 0.0;

                var interactionRadius = 0.015;
                var forceFactor = 0.003;
                for (var i = 0; i < ${NUM_PARTICLES}; i++) {
                  var xdiff = positions[i].x - position.x;
                  var ydiff = positions[i].y - position.y;
                  var zdiff = positions[i].z - position.z;
                  var dist = (xdiff*xdiff) + (ydiff*ydiff) + (zdiff*zdiff);


                if (dist < interactionRadius && dist > interactionRadius*0.5) {
                  forcex += xdiff*forceFactor ;
                  forcey += ydiff*forceFactor;
                  forcez += zdiff*forceFactor ;
                } else if (dist < interactionRadius*0.5) {
                  forcex -= xdiff*forceFactor ;
                  forcey -= ydiff*forceFactor;
                  forcez -= zdiff*forceFactor ;
                }}

                velocity.x+=forcex;
                velocity.y+=forcey;
                velocity.z+=forcez;

                velocity*=0.5;

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
      }
    ]
  });

  ///////////////////////////////////
  // Create buffers for render pass
  ///////////////////////////////////

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

  const colorBufferData = new Uint8Array(4 * NUM_PARTICLES);
  for (let i = 0; i < colorBufferData.length; i += 4) {
    const rgbValues = HSLToRGB(360 * (Math.floor(Math.random() * NUM_COLORS) / NUM_COLORS), 100, 50)
    colorBufferData[i] = rgbValues[0];
    colorBufferData[i + 1] = rgbValues[1];
    colorBufferData[i + 2] = rgbValues[2];
    colorBufferData[i + 3] = 70;
  }

  const colorBuffer = device.createBuffer({
    size: colorBufferData.byteLength,
    usage: GPUBufferUsage.VERTEX | GPUBufferUsage.COPY_DST,
  });

  device.queue.writeBuffer(colorBuffer, 0, colorBufferData);

  /////////////////////////////////
  // Create render shader module
  /////////////////////////////////

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
                vsOut.color = color;
                return vsOut;
            }             

            @fragment 
            fn fs(@location(0) color: vec4f) -> @location(0) vec4f {
                return vec4f(color.rgb * color.a, color.a);
            } 
        `
  });

  /////////////////////////////////
  // Create render pipeline
  /////////////////////////////////

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

  ///////////////////////////////////////////////
  // Rendering uniform buffer
  ///////////////////////////////////////////////

  const vertexUniformBuffer = device.createBuffer({
    size: 16,
    usage: GPUBufferUsage.UNIFORM | GPUBufferUsage.COPY_DST
  });

  device.queue.writeBuffer(vertexUniformBuffer, 0, new Float32Array([
    canvas!.width, canvas!.height, PARTICLE_SIZE
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


  ///////////////////////////
  // Render pass description
  ///////////////////////////

  const renderPassDescriptor = {
    colorAttachments: [{
      view: (context as any).getCurrentTexture().createView(),
      loadOp: "clear",
      storeOp: "store",
      clearValue: [0, 0, 0, 1]
    }]
  };

  requestAnimationFrame(function draw() {

    /////////////////////////
    // Set up command buffer
    /////////////////////////

    const commandEncoder = device.createCommandEncoder();

    ///////////////////////
    // Encode compute pass
    ///////////////////////

    const computePass = commandEncoder.beginComputePass();
    computePass.setPipeline(computePipeline);
    computePass.setBindGroup(0, computeBindGroup);
    computePass.dispatchWorkgroups(NUM_WORKGROUPS);
    computePass.end();

    ////////////////////////////////
    // Get current canvas texture
    ////////////////////////////////

    renderPassDescriptor.colorAttachments[0].view = (context as any).getCurrentTexture().createView();

    ///////////////////////
    // Encode render pass
    ///////////////////////

    const renderPass = commandEncoder.beginRenderPass(renderPassDescriptor);
    renderPass.setPipeline(renderPipeline);

    // First argument here refers to array index
    // in renderPipeline vertexState.vertexBuffers
    renderPass.setVertexBuffer(0, vertexBuffer);
    renderPass.setVertexBuffer(1, colorBuffer);
    renderPass.setVertexBuffer(2, positionBuffer);
    renderPass.setBindGroup(0, vertexUniformBindGroup);

    renderPass.draw(4, NUM_PARTICLES);
    renderPass.end();

    //////////////////////////
    // Submit command buffer
    //////////////////////////

    device.queue.submit([commandEncoder.finish()]);

    requestAnimationFrame(draw);
  });
});
</script>

<template>
  <Loader :text="text" />
  <canvas id="webgpu-canvas" class="absolute top-0 left-0 "></canvas>
</template>
