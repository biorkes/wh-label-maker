<script setup>
import { ref, computed, watch } from 'vue'
import QRCode from 'qrcode.vue'
import { jsPDF } from 'jspdf'
import html2canvas from 'html2canvas'
import './style.css'
import arrowSvg from './assets/arrow-1.svg'

const labelConfig = ref({
  numLevels: 1,
  labels: [{
    rack: '002',
    letter: 'A',
    bin: '002',
    height: 220,
    maxWeight: 180,
    selectedCell: null,
    qrContent: '',
    arrowDirection: 'right'
  }]
})

// Watch numLevels and update labels array
watch(() => labelConfig.value.numLevels, (newValue) => {
  const currentLength = labelConfig.value.labels.length
  if (newValue > currentLength) {
    // Add new forms
    for (let i = currentLength; i < newValue; i++) {
      labelConfig.value.labels.push({
        rack: '002',
        letter: String.fromCharCode(65 + i),
        bin: '002',
        height: 220,
        maxWeight: 180,
        selectedCell: null,
        qrContent: '',
        arrowDirection: 'right'
      })
    }
  } else if (newValue < currentLength) {
    // Remove excess forms
    labelConfig.value.labels.splice(newValue)
  }
}, { immediate: true })

const totalPages = computed(() => {
  return Math.ceil(labelConfig.value.numLevels / 2)
})

// Group labels into pages
const labelPages = computed(() => {
  const pages = []
  for (let i = 0; i < labelConfig.value.labels.length; i += 2) {
    pages.push({
      topLabel: labelConfig.value.labels[i + 1],
      bottomLabel: labelConfig.value.labels[i]
    })
  }
  return pages
})

const formattedLabel = (label) => {
  return `${label.rack} ${label.letter} ${label.bin}`
}

const qrCodeContent = (label) => {
  return label.qrContent || formattedLabel(label)
}

const generatePDF = async () => {
  const pdf = new jsPDF({
    orientation: 'portrait',
    unit: 'mm',
    format: 'a6'
  })
  
  for (let page = 0; page < totalPages.value; page++) {
    if (page > 0) pdf.addPage()
    
    const element = document.getElementById(`labelPage${page}`)
    const canvas = await html2canvas(element, {
      scale: 2,
      useCORS: true,
      scrollX: 0,
      scrollY: 0
    })
    
    const imgData = canvas.toDataURL('image/png')
    pdf.addImage(imgData, 'PNG', 0, 0, 105, 148)
  }
  
  pdf.save('warehouse-labels.pdf')
}

// Grid cell helper
const isSelectedCell = (index, selectedCell) => {
  return index === selectedCell
}

// Helper to determine if letter is odd-indexed (A,C,E,...)
const isOddIndexedLetter = (letter) => {
  return (letter.charCodeAt(0) - 65) % 2 === 0
}

// Add click handler
const handleCellClick = (label, cellIndex) => {
  label.selectedCell = cellIndex
}
</script>

<template>
  <div class="container mx-auto p-4">
    <h1 class="text-2xl font-bold mb-4">Warehouse Label Generator</h1>
    
    <div class="grid grid-cols-1 md:grid-cols-2 gap-4">
      <!-- Label Configuration Form -->
      <div class="bg-white p-4 rounded shadow">
        <h2 class="text-xl font-semibold mb-4">Label Configuration</h2>
        <div class="space-y-4">
          <div>
            <label class="block text-sm font-medium mb-1">Number of Levels</label>
            <input 
              v-model="labelConfig.numLevels" 
              type="number" 
              min="1"
              class="w-full p-2 border rounded"
            >
          </div>

          <!-- Dynamic forms based on number of levels -->
          <div v-for="(label, index) in labelConfig.labels" :key="index" class="p-4 border rounded space-y-4">
            <h3 class="font-semibold">Level {{ index + 1 }} Configuration</h3>
            
            <div class="grid grid-cols-3 gap-4">
              <div>
                <label class="block text-sm font-medium mb-1">Rack Number</label>
                <input 
                  v-model="label.rack" 
                  type="text" 
                  class="w-full p-2 border rounded"
                  pattern="[0-9]{3}"
                  placeholder="002"
                  required
                >
              </div>
              <div>
                <label class="block text-sm font-medium mb-1">Level Letter</label>
                <input 
                  v-model="label.letter" 
                  type="text" 
                  class="w-full p-2 border rounded"
                  maxlength="1"
                  pattern="[A-Za-z]"
                  required
                >
              </div>
              <div>
                <label class="block text-sm font-medium mb-1">Bin Number</label>
                <input 
                  v-model="label.bin" 
                  type="text" 
                  class="w-full p-2 border rounded"
                  pattern="[0-9]{3}"
                  placeholder="002"
                  required
                >
              </div>
            </div>

            <div class="grid grid-cols-2 gap-4">
              <div>
                <label class="block text-sm font-medium mb-1">Height (cm)</label>
                <input 
                  v-model="label.height" 
                  type="number" 
                  class="w-full p-2 border rounded"
                  required
                >
              </div>
              <div>
                <label class="block text-sm font-medium mb-1">Max Weight (kg)</label>
                <input 
                  v-model="label.maxWeight" 
                  type="number" 
                  class="w-full p-2 border rounded"
                  required
                >
              </div>
            </div>

            <div>
              <label class="block text-sm font-medium mb-1">Selected Cell (click to select)</label>
              <div class="grid grid-cols-3 gap-0.5 w-32 mx-auto">
                <div v-for="i in 9" :key="i-1" 
                  class="aspect-square border border-black cursor-pointer hover:bg-gray-200"
                  :class="{ 'bg-black hover:bg-black': isSelectedCell(i-1, label.selectedCell) }"
                  @click="handleCellClick(label, i-1)"
                ></div>
              </div>
            </div>

            <div>
              <label class="block text-sm font-medium mb-1">QR Code Content</label>
              <input 
                v-model="label.qrContent" 
                type="text" 
                class="w-full p-2 border rounded"
                placeholder="Enter QR code content"
                required
              >
            </div>

            <div>
              <label class="block text-sm font-medium mb-1">Arrow Direction</label>
              <select 
                v-model="label.arrowDirection" 
                class="w-full p-2 border rounded"
                required
              >
                <option value="right">Right</option>
                <option value="left">Left</option>
              </select>
            </div>
          </div>
        </div>
      </div>

      <!-- Label Preview -->
      <div class="bg-white p-4 rounded shadow">
        <h2 class="text-xl font-semibold mb-4">Label Preview</h2>
        
        <div v-for="(page, pageIndex) in labelPages" :key="pageIndex" class="mb-8">
          <div 
            :id="'labelPage' + pageIndex"
            class="border-2 border-gray-400 rounded mx-auto bg-white"
            style="width: 105mm; height: 148mm;"
          >
            <!-- Top Label -->
            <div v-if="page.topLabel" class="h-[72mm] border-b border-dotted border-black p-4 flex flex-col">
              <div class="text-5xl font-black tracking-wider text-center mb-4">
                {{ formattedLabel(page.topLabel) }}
              </div>
              
              <div class="flex flex-1 items-center">
                <!-- Left Column (40%) -->
                <div class="w-[40%] pr-2">
                  <!-- QR Code -->
                  <QRCode 
                    :value="qrCodeContent(page.topLabel)"
                    :size="135"
                    level="H"
                    render-as="svg"
                  />
                </div>

                <!-- Right Column (60%) -->
                <div class="w-[60%] pl-2">
                  <!-- Grid and Metrics Container -->
                  <div class="flex">
                    <!-- Left Half (50%) -->
                    <div class="w-1/2 pr-1">
                      <!-- Grid -->
                      <div class="grid grid-cols-3 gap-0.5 w-full">
                        <div v-for="i in 9" :key="i-1" 
                          class="aspect-square border border-black"
                          :class="{ 'bg-black': isSelectedCell(i-1, page.topLabel.selectedCell) }"
                        ></div>
                      </div>
                    </div>

                    <!-- Right Half (50%) -->
                    <div class="w-1/2 pl-1">
                      <!-- Metrics -->
                      <div class="space-y-1">
                        <div class="border border-black px-1 py-0.5">
                          <div class="text-[10px]">HEIGHT CM</div>
                          <div class="text-lg font-bold">{{ page.topLabel.height }}</div>
                        </div>
                        <div class="border border-black px-1 py-0.5">
                          <div class="text-[10px]">MAX WEIGHT</div>
                          <div class="text-lg font-bold">{{ page.topLabel.maxWeight }} KG</div>
                        </div>
                      </div>
                    </div>
                  </div>

                  <!-- Arrow (Below both columns) -->
                  <div class="relative h-[30px] mt-2">
                    <div :class="[
                      'absolute top-1/2 transform -translate-y-1/2',
                      page.topLabel.arrowDirection === 'right' ? 'right-0' : 'left-0'
                    ]">
                      <img 
                        :src="arrowSvg" 
                        :alt="page.topLabel.arrowDirection + ' arrow'" 
                        class="h-[40px] w-[200px] py-1"
                        :class="{ 'transform scale-x-[-1]': page.topLabel.arrowDirection === 'left' }"
                      />
                    </div>
                  </div>
                </div>
              </div>
            </div>

            <!-- Bottom Label -->
            <div :class="[
              'p-4 flex flex-col',
              {
                'h-[72mm]': page.topLabel,
                'h-[148mm]': !page.topLabel
              }
            ]">
              <div class="flex-1"></div>
              <div>
                <div class="text-5xl font-black tracking-wider text-center mb-4">
                  {{ formattedLabel(page.bottomLabel) }}
                </div>
                
                <div class="flex items-center">
                  <!-- Left Column (60%) -->
                  <div class="w-[60%] pr-2">
                    <!-- Grid and Metrics Container -->
                    <div class="flex">
                      <!-- Left Half (50%) -->
                      <div class="w-1/2 pr-1">
                        <!-- Metrics -->
                        <div class="space-y-1">
                          <div class="border border-black px-1 py-0.5">
                            <div class="text-[10px]">HEIGHT CM</div>
                            <div class="text-lg font-bold">{{ page.bottomLabel.height }}</div>
                          </div>
                          <div class="border border-black px-1 py-0.5">
                            <div class="text-[10px]">MAX WEIGHT</div>
                            <div class="text-lg font-bold">{{ page.bottomLabel.maxWeight }} KG</div>
                          </div>
                        </div>
                      </div>

                      <!-- Right Half (50%) -->
                      <div class="w-1/2 pl-1">
                        <!-- Grid -->
                        <div class="grid grid-cols-3 gap-0.5 w-full">
                          <div v-for="i in 9" :key="i-1" 
                            class="aspect-square border border-black"
                            :class="{ 'bg-black': isSelectedCell(i-1, page.bottomLabel.selectedCell) }"
                          ></div>
                        </div>
                      </div>
                    </div>

                    <!-- Arrow (Below both columns) -->
                    <div class="relative h-[30px] mt-2">
                      <div :class="[
                        'absolute top-1/2 transform -translate-y-1/2',
                        page.bottomLabel.arrowDirection === 'right' ? 'right-0' : 'left-0'
                      ]">
                        <img 
                          :src="arrowSvg" 
                          :alt="page.bottomLabel.arrowDirection + ' arrow'" 
                          class="h-[40px] w-[200px] py-1"
                          :class="{ 'transform scale-x-[-1]': page.bottomLabel.arrowDirection === 'left' }"
                        />
                      </div>
                    </div>
                  </div>

                  <!-- Right Column (40%) -->
                  <div class="w-[40%] pl-2">
                    <!-- QR Code -->
                    <QRCode 
                      :value="qrCodeContent(page.bottomLabel)"
                      :size="135"
                      level="H"
                      render-as="svg"
                    />
                  </div>
                </div>
              </div>
            </div>
          </div>
        </div>

        <button 
          @click="generatePDF" 
          class="mt-4 w-full bg-blue-500 text-white py-2 px-4 rounded hover:bg-blue-600"
        >
          Generate PDF
        </button>
      </div>
    </div>
  </div>
</template>

<style>
@import 'tailwindcss/base';
@import 'tailwindcss/components';
@import 'tailwindcss/utilities';

body {
  background-color: #f3f4f6;
}

/* Fix for html2canvas PDF generation image shifting */
img {
  display: inline-block !important;
}

/* Ensure SVG elements (QR codes) are also properly handled */
svg {
  display: inline-block !important;
}
</style>
