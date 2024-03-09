<script setup lang="ts">
import { Dialog, DialogPanel, DialogTitle, TransitionChild, TransitionRoot } from '@headlessui/vue'
import { XMarkIcon } from '@heroicons/vue/24/outline'
import { ref, Ref } from 'vue';
import { defaultConfig } from './shared/defaultComfig';

const open = ref(true)
const fps = ref('')
const parameters = ref(window.localStorage.getItem('parameters') ? JSON.parse(window.localStorage.getItem('parameters')!) : defaultConfig);

import Particles from './components/Particles.vue';
import Unpotimized from './components/Unpotimized.vue';

import BasicPL from './components/BasicPL.vue';
import Osci from './components/Osci.vue';
import Optimized from './components/Optimized.vue';


const activeSim = ref(window.localStorage.getItem('sim') as unknown as Ref<string> || 'webGPU')

function switchSim(sim: string) {
  window.localStorage.setItem('sim', sim);
  window.location.reload();
}

function reload() {
  window.location.reload();
}

function updateParameters() {
  window.localStorage.setItem('parameters', JSON.stringify(parameters.value))
  window.location.reload()
}

function resetParameters() {
  window.localStorage.setItem('parameters', JSON.stringify(defaultConfig))
  window.location.reload()
}
const maxRadius = ref((2 * (64000 / parameters.value.particlesCountGPU)) > 2.0 ? 2.0 : (2 * (64000 / parameters.value.particlesCountGPU)).toFixed(1))
console.log(maxRadius.value)
</script>

<template>
  <div class="flex">

    <!-- <div>
      <Particles @fps="fps = $event" />
    </div> -->
    <!-- <div>
      <Unpotimized />
    </div> -->
    <div v-if="activeSim === 'webGPU'">
      <Particles @fps="fps = $event" />
    </div>
    <!-- <div v-if="activeSim === 'osci'">
      <Osci />
    </div>-->
    <div v-if="activeSim === 'basicLife'">
      <BasicPL />
    </div>
    <div v-if="activeSim === 'optimizedLife'">
      <Optimized />
    </div>
  </div>
  <TransitionRoot as="template" :show="open">
    <Dialog as="div" class="relative z-10" @close="open = false">
      <div class="fixed inset-0" />

      <div class="fixed inset-0 overflow-hidden ">
        <div class="absolute inset-0 overflow-hidden">
          <div class="pointer-events-none fixed inset-y-0 right-0 flex max-w-full pl-10">
            <TransitionChild as="template" enter="transform transition ease-in-out duration-500 sm:duration-700"
              enter-from="translate-x-full" enter-to="translate-x-0"
              leave="transform transition ease-in-out duration-500 sm:duration-700" leave-from="translate-x-0"
              leave-to="translate-x-full">
              <DialogPanel class="pointer-events-auto md:w-[20vw] max-w-md">
                <div
                  class="flex h-full flex-col  z-[1000] shadow-lg dark:shadow-gray-900 bg-gray-50 dark:bg-black py-6">
                  <div class="px-4 sm:px-6">
                    <button type="button" class="absolute right-0
                       rounded-md bg-gray-50 dark:bg-black py text-gray-400 hover:text-gray-500 focus:outline-none "
                      @click="open = false">
                      <span class="absolute -inset-2.5" />
                      <span class="sr-only">Close panel</span>
                      <XMarkIcon class="h-6 w-6" aria-hidden="true" />
                    </button>
                    <h1 class="text-lg">Particle Life</h1>
                    <div class="flex items-start justify-between">
                      <DialogTitle class="text-base font-semibold leading-6 text-gray-400">System parameters
                      </DialogTitle>
                      <div class="ml-3 flex h-7 items-center">
                      </div>
                    </div>
                  </div>
                  <div class="relative mt-6 flex-1 px-4 sm:px-6">
                    <small>
                      {{ fps }}
                    </small>
                    <div v-if="activeSim === 'basicLife'" class="flex items-center">
                      <label for="disabled-range"
                        class=" w-[100px] block mr-2 text-sm font-medium text-gray-900 dark:text-white">Particles
                      </label>
                      <input @change="updateParameters()" v-model="parameters.particlesCountJS" type="range" step="100"
                        min="500" max="5000"
                        class="w-[100px] mr-2 w-full h-2 bg-gray-200 rounded-lg appearance-none cursor-pointer dark:bg-gray-700">
                      <span>{{ parameters.particlesCountJS }}</span>
                    </div>
                    <div v-if="activeSim === 'webGPU'" class="flex items-center">
                      <label for="disabled-range"
                        class=" w-[100px] block mr-2 text-sm font-medium text-gray-900 dark:text-white">Particles
                      </label>
                      <input @change="updateParameters()" v-model="parameters.particlesCountGPU" type="range"
                        step="6400" min="6400" max="1024000"
                        class="w-[100px] mr-2 w-full h-2 bg-gray-200 rounded-lg appearance-none cursor-pointer dark:bg-gray-700">
                      <span>{{ parameters.particlesCountGPU }}</span>
                    </div>
                    <div class="flex items-center">
                      <label for="disabled-range"
                        class="w-[100px] block mr-2 text-sm font-medium text-gray-900 dark:text-white">Radius </label>
                      <input @change="updateParameters()" v-model="parameters.interactionRadius" step="0.1" type="range"
                        min="0.1" :max="maxRadius"
                        class="w-[100px] mr-2 w-full h-2 bg-gray-200 rounded-lg appearance-none cursor-pointer dark:bg-gray-700">
                      <span>{{ parameters.interactionRadius }}</span>
                    </div>
                    <div class="flex items-center">
                      <label for="disabled-range"
                        class="w-[100px] block mr-2 text-sm font-medium text-gray-900 dark:text-white">Max. opacity
                      </label>
                      <input @change="updateParameters()" v-model="parameters.opacity" step="5" type="range" min="25"
                        max="100"
                        class="w-[100px] mr-2 w-full h-2 bg-gray-200 rounded-lg appearance-none cursor-pointer dark:bg-gray-700">
                      <span>{{ parameters.opacity }}%</span>
                    </div>
                    <div class="flex items-center">
                      <label for="disabled-range"
                        class="w-[100px] block mr-2 text-sm font-medium text-gray-900 dark:text-white">Colors</label>
                      <input @change="updateParameters()" v-model="parameters.colorsNumber" step="1" type="range"
                        min="1" max="9"
                        class="w-[100px] mr-2 w-full h-2 bg-gray-200 rounded-lg appearance-none cursor-pointer dark:bg-gray-700">
                      <span>{{ parameters.colorsNumber }}</span>
                    </div>
                  </div>
                  <div class="flex flex-col  mx-4">
                    <button class="mt-6 bg-indigo-500" @click="reload()">New Random Combination</button>
                    <button class="mt-2 bg-indigo-500" @click="resetParameters()">Reset Parameters</button>
                  </div>
                  <h4 class="mt-6 px-4 sm:px-6 text-base font-semibold leading-6 text-gray-400">Active simulation
                  </h4>

                  <div class="flex flex-col  mx-4">
                    <button class="mt-2" @click="switchSim('webGPU')">WebGPU 3D Particle Life</button>
                    <button class="mt-2" @click="switchSim('osci')">WebGPU 3D Oscillators</button>
                    <button class="mt-2" @click="switchSim('basicLife')">JS 2D</button>
                    <button class="mt-2" @click="switchSim('optimizedLife')"> JS 2D (QuadTree)</button>
                  </div>
                </div>
              </DialogPanel>
            </TransitionChild>
          </div>
        </div>
      </div>
    </Dialog>
  </TransitionRoot>
</template>