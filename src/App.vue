<script setup lang="ts">
import { Dialog, DialogPanel, DialogTitle, TransitionChild, TransitionRoot } from '@headlessui/vue'
import { XMarkIcon, ChevronDoubleLeftIcon } from '@heroicons/vue/24/outline'
import { ref, Ref } from 'vue';
import { defaultConfig } from './shared/defaultConfig';
import Particles from './components/Particles.vue';
import BasicPL from './components/BasicPL.vue';
import Osci from './components/Osci.vue';
import Optimized from './components/Optimized.vue';

const open = ref(true)
const fps = ref('')
const parameters = ref(window.localStorage.getItem('parameters') ? JSON.parse(window.localStorage.getItem('parameters')!) : defaultConfig);
const activeSim = ref(window.localStorage.getItem('sim') as unknown as Ref<string> || 'webGPU')
const maxRadius = ref((2 * (64000 / parameters.value.particlesCountGPU)) > 2.0 ? 2.0 : (2 * (64000 / parameters.value.particlesCountGPU)).toFixed(1))

function switchSim(sim: string) {
  window.localStorage.setItem('sim', sim);
  reload()
}

function reload() {
  window.location.reload();
}

function updateParameters() {
  window.localStorage.setItem('parameters', JSON.stringify(parameters.value))
  reload()
}

function resetParameters() {
  window.localStorage.setItem('parameters', JSON.stringify(defaultConfig))
  reload()
}
</script>

<template>
  <div class="flex">
    <div v-if="activeSim === 'webGPU'">
      <Particles @fps="fps = $event" />
    </div>
    <div v-if="activeSim === 'osci'">
      <Osci />
    </div>
    <div v-if="activeSim === 'basicLife'">
      <BasicPL />
    </div>
    <div v-if="activeSim === 'optimizedLife'">
      <Optimized />
    </div>
  </div>
  <button v-if="!open" type="button" class="absolute right-0
                       rounded-md bg-gray-50 dark:bg-black py text-gray-400 hover:text-gray-500 focus:outline-none "
    @click="open = true">
    <span class="absolute -inset-2.5" />
    <span class="sr-only">Menu</span>
    <ChevronDoubleLeftIcon class="h-6 w-6" aria-hidden="true" />
  </button>

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
                      <label
                        class="text-xs  w-[100px] block mr-2 text-sm font-medium text-gray-900 dark:text-white">Particles
                      </label>
                      <input @change="updateParameters()" v-model="parameters.particlesCountJS" type="range" step="100"
                        min="500" max="5000"
                        class="w-[120px] mr-2 w-full h-2 bg-gray-200 rounded-lg appearance-none cursor-pointer dark:bg-gray-700">
                      <span class="w-[30px]">{{ parameters.particlesCountJS }}</span>
                    </div>
                    <div v-if="activeSim === 'webGPU'" class="flex items-center">
                      <label
                        class="text-xs  w-[100px] block mr-2 text-sm font-medium text-gray-900 dark:text-white">Particles
                      </label>
                      <input @change="updateParameters()" v-model="parameters.particlesCountGPU" type="range"
                        step="6400" min="6400" max="1024000"
                        class="w-[120px] mr-2 w-full h-2 bg-gray-200 rounded-lg appearance-none cursor-pointer dark:bg-gray-700">
                      <span class="w-[30px]">{{ parameters.particlesCountGPU }}</span>
                    </div>
                    <div v-if="activeSim === 'optimizedLife'" class="flex items-center">
                      <label
                        class="text-xs  w-[100px] block mr-2 text-sm font-medium text-gray-900 dark:text-white">Particles
                      </label>
                      <input @change="updateParameters()" v-model="parameters.particlesCountQTJS" type="range"
                        step="1000" min="5000" max="50000"
                        class="w-[120px] mr-2 w-full h-2 bg-gray-200 rounded-lg appearance-none cursor-pointer dark:bg-gray-700">
                      <span class="w-[30px]">{{ parameters.particlesCountQTJS }}</span>
                    </div>
                    <div v-if="activeSim === 'optimizedLife'" class="flex items-center">
                      <label
                        class="text-xs w-[100px] block mr-2 text-sm font-medium text-gray-900 dark:text-white">Radius
                      </label>
                      <input @change="updateParameters()" v-model="parameters.interactionRadiusQTJS" step="0.005"
                        type="range" min="0.01" max="0.1"
                        class="w-[120px] mr-2 w-full h-2 bg-gray-200 rounded-lg appearance-none cursor-pointer dark:bg-gray-700">
                      <span class="w-[30px]">{{ parameters.interactionRadiusQTJS }}</span>
                    </div>
                    <div v-if="activeSim === 'webGPU'" class="flex items-center">
                      <label
                        class="text-xs w-[100px] block mr-2 text-sm font-medium text-gray-900 dark:text-white">Radius
                      </label>
                      <input @change="updateParameters()" v-model="parameters.interactionRadius" step="0.1" type="range"
                        min="0.1" :max="maxRadius"
                        class="w-[120px] mr-2 w-full h-2 bg-gray-200 rounded-lg appearance-none cursor-pointer dark:bg-gray-700">
                      <span class="w-[30px]">{{ parameters.interactionRadius }}</span>
                    </div>
                    <div v-if="activeSim === 'basicLife'" class="flex items-center">
                      <label
                        class="text-xs w-[100px] block mr-2 text-sm font-medium text-gray-900 dark:text-white">Radius
                      </label>
                      <input @change="updateParameters()" v-model="parameters.interactionRadiusJS" step="0.1"
                        type="range" min="0.1" max="1.0"
                        class="w-[120px] mr-2 w-full h-2 bg-gray-200 rounded-lg appearance-none cursor-pointer dark:bg-gray-700">
                      <span class="w-[30px]">{{ parameters.interactionRadiusJS }}</span>
                    </div>
                    <div v-if="activeSim === 'webGPU'" class="flex items-center">
                      <label class="text-xs w-[100px] block mr-2 text-sm font-medium text-gray-900 dark:text-white">Max.
                        opacity
                      </label>
                      <input @change="updateParameters()" v-model="parameters.opacity" step="5" type="range" min="25"
                        max="100"
                        class="w-[120px] mr-2 w-full h-2 bg-gray-200 rounded-lg appearance-none cursor-pointer dark:bg-gray-700">
                      <span class="w-[30px]">{{ parameters.opacity }}%</span>
                    </div>
                    <div v-if="activeSim !== 'osci'" class="flex items-center">
                      <label
                        class="text-xs w-[100px] block mr-2 text-sm font-medium text-gray-900 dark:text-white">Colors</label>
                      <input @change="updateParameters()" v-model="parameters.colorsNumber" step="1" type="range"
                        min="1" max="9"
                        class="w-[120px] mr-2 w-full h-2 bg-gray-200 rounded-lg appearance-none cursor-pointer dark:bg-gray-700">
                      <span class="w-[30px]">{{ parameters.colorsNumber }}</span>
                    </div>
                    <div v-if="activeSim === 'basicLife'" class="flex items-center">
                      <label class="text-xs w-[100px] block mr-2 text-sm font-medium text-gray-900 dark:text-white">
                        Repulsive radius</label>
                      <input @change="updateParameters()" v-model="parameters.repulsiveRadius" step="0.05" type="range"
                        min="0.05" max="0.8"
                        class="w-[120px] mr-2 w-full h-2 bg-gray-200 rounded-lg appearance-none cursor-pointer dark:bg-gray-700">
                      <span class="w-[30px]">{{ parameters.repulsiveRadius }}</span>
                    </div>
                    <div v-if="activeSim === 'basicLife'">
                      <div class="flex items-center">
                        <label for="countries"
                          class="w-[100px] text-xs block mb-2 text-sm font-medium text-gray-900 dark:text-white">Force
                          function</label>
                        <select @change="updateParameters()" v-model="parameters.forceFunction" id="countries"
                          class=" w-[120px]  bg-gray-50 border border-gray-300 text-gray-900 text-sm rounded-lg focus:ring-blue-500 focus:border-blue-500 block w-full  dark:bg-gray-700 dark:border-gray-600 dark:placeholder-gray-400 dark:text-white dark:focus:ring-blue-500 dark:focus:border-blue-500">
                          <option selected value="1"> Variant 1 </option>
                          <option value="2"> Variant 2</option>
                          <option value="3"> Variant 3</option>

                        </select>
                      </div>
                      <img v-if="parameters.forceFunction === '2'" class="block h-[120px] mt-2"
                        src="https://programmingattack.netlify.app/articles/graph.png" />
                      <img v-if="parameters.forceFunction === '1'" class="block h-[120px] mt-2"
                        src="https://programmingattack.netlify.app/articles/graph%20(3).png" /><img
                        v-if="parameters.forceFunction == '3'" class="block h-[120px] mt-2"
                        src="https://programmingattack.netlify.app/articles/graph%20(1).png" />
                    </div>
                  </div>
                  <div class="flex flex-col  mx-4">
                    <button class="mt-6  text-xs bg-indigo-500" @click="reload()">New Random Combination</button>
                    <button class="mt-2 text-xs bg-indigo-500" @click="resetParameters()">Reset Parameters</button>
                  </div>
                  <h4 class="mt-6 px-4 sm:px-6 text-base font-semibold leading-6 text-gray-400">Active simulation
                  </h4>

                  <div class="flex flex-col  mx-4">
                    <button class="mt-2 text-xs" @click="switchSim('webGPU')">WebGPU 3D Particle Life</button>
                    <button class="mt-2 text-xs" @click="switchSim('osci')">WebGPU 3D Oscillators</button>
                    <button class="mt-2 text-xs" @click="switchSim('basicLife')">JS 2D</button>
                    <button class="mt-2 text-xs" @click="switchSim('optimizedLife')"> JS 2D (QuadTree)</button>
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