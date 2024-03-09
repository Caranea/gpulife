<script setup lang="ts">

const NUM_COLORS = 1
const WORKGROUP_SIZE = 64;
const NUM_WORKGROUPS = 1000;
const NUM_PARTICLES = WORKGROUP_SIZE * NUM_WORKGROUPS;
const PARTICLE_SIZE = 2;

(async () => {

  console.log('num particles ', NUM_PARTICLES)

  //////////////////////////////////////////
  // Set up WebGPU adapter
  //////////////////////////////////////////

  const adapter = await navigator.gpu.requestAdapter();
  const presentationFormat = navigator.gpu.getPreferredCanvasFormat(adapter);

  ////////////////////////////////////
  // Set up device and canvas context
  ////////////////////////////////////

  const device = await adapter.requestDevice();

  const canvas = document.getElementById("webgpu-canvas");
  canvas!.width = window.innerWidth * 0.5;
  canvas!.height = canvas!.width;
  const context = canvas!.getContext("webgpu");

  context.configure({
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




  ////////////////////////////////////////////////////////
  // Set up sectors for space partitioning
  ////////////////////////////////////////////////////////

  const sectionBufferData = new Int32Array(8 * 2); //starting and ending index

  const sectionBuffer = device.createBuffer({
    size: sectionBufferData.byteLength,
    usage: GPUBufferUsage.COPY_DST | GPUBufferUsage.STORAGE,
  });


  for (let i = 0; i < sectionBufferData.length; i++) {
    sectionBufferData[i] = 0
  }
  const sectorsParticles: any = [[], [], [], [], [], [], [], []]

  for (let i = 0; i < positionBufferData.length; i += 4) {
    const particle = new Float32Array(4);
    particle[0] = positionBufferData[i]
    particle[1] = positionBufferData[i + 1]
    particle[2] = positionBufferData[i + 2]
    particle[3] = positionBufferData[i + 3]

    if (positionBufferData[i + 1] >= 0) {
      if (positionBufferData[i] <= -0.5) {
        sectorsParticles[0].push(...particle)
      } else if (positionBufferData[i] <= 0 && positionBufferData[i] > -0.5) {
        sectorsParticles[1].push(...particle)
      } else if (positionBufferData[i] <= 0.5 && positionBufferData[i] > 0) {
        sectorsParticles[2].push(...particle)
      } else if (positionBufferData[i] <= 1 && positionBufferData[i] > 0.5) {
        sectorsParticles[3].push(...particle)
      }
    } else if (positionBufferData[i + 1] < 0) {
      if (positionBufferData[i] <= -0.5) {
        sectorsParticles[4].push(...particle)
      } else if (positionBufferData[i] <= 0 && positionBufferData[i] > -0.5) {
        sectorsParticles[5].push(...particle)
      } else if (positionBufferData[i] <= 0.5 && positionBufferData[i] > 0) {
        sectorsParticles[6].push(...particle)
      } else if (positionBufferData[i] <= 1 && positionBufferData[i] > 0.5) {
        sectorsParticles[7].push(...particle)
      }
    }

  }


  let positionBufferDataNew = new Float32Array(NUM_PARTICLES * 4);

  let helperIndex = 0

  sectorsParticles.forEach((sector: any, index: any) => {
    // sectionBufferData[index] = sector.length;

    if (index > 0) {
      const prevEndingIndex = (index * 2) - 1
      const startingIndex = (index * 2)
      sectionBufferData[(index * 2) - 1] = helperIndex - 4
      sectionBufferData[index * 2] = helperIndex
    }

    for (let i = 0; i < sector.length; i += 4) {
      positionBufferDataNew[helperIndex] = sector[i]
      positionBufferDataNew[helperIndex + 1] = sector[i + 1]
      positionBufferDataNew[helperIndex + 2] = sector[i + 2]
      positionBufferDataNew[helperIndex + 3] = 1;
      helperIndex += 4
    }

    if (index == 7) {
      sectionBufferData[index * 2 + 1] = helperIndex
    }
  })
  // console.log(sectorsParticles)
  // console.log(positionBufferData)
  // console.log(positionBufferDataNew)

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
  // device.queue.writeBuffer(sectionBuffer, 0, sectionBufferData);




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
    // colorBufferData[i + 3] = 255;

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
                vsOut.clipPosition = vec4f(vertexPosition * vertexUniforms.particleSize / vertexUniforms.screenDimensions + position.xy, position.z, 1.0);
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
      view: context.getCurrentTexture().createView(),
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

    renderPassDescriptor.colorAttachments[0].view = context.getCurrentTexture().createView();

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
})();
function HSLToRGB(h, s, l) {
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
</script>

<template>
  <canvas id="webgpu-canvas"></canvas>
</template>

