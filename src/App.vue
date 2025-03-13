<script setup>
import { ref, computed, watch } from 'vue'
import QRCode from 'qrcode.vue'
import { jsPDF } from 'jspdf'
import html2canvas from 'html2canvas'
import './style.css'
import arrowSvg from './assets/arrow-1.svg'

const labelConfig = ref({
  numLevels: 1,
  numRacks: 1,
  currentRack: '001',
  labels: [{
    rack: '001',
    letter: 'A',
    bin: '001',
    height: 220,
    maxWeight: 180,
    selectedCell: null,
    qrContent: '',
    arrowDirection: 'right'
  }]
})

// Add method to update rack number
const updateRackNumber = (newRack) => {
  labelConfig.value.currentRack = newRack
  labelConfig.value.labels.forEach(label => {
    label.rack = newRack
  })
}

// Watch for changes in both numLevels and numRacks
watch([() => labelConfig.value.numLevels, () => labelConfig.value.numRacks], ([newLevels, newRacks]) => {
  const totalLabels = newLevels * newRacks
  
  // Create a new array of labels
  const newLabels = []
  
  // Generate all labels in the correct order
  for (let shelfIndex = 0; shelfIndex < newLevels; shelfIndex++) {
    for (let binIndex = 0; binIndex < newRacks; binIndex++) {
      const letter = String.fromCharCode(65 + shelfIndex)
      const bin = String(binIndex + 1).padStart(3, '0')
      
      // Calculate the cell index based on shelf and bin position
      // Reverse the shelf index to make A appear at the bottom
      const reversedShelfIndex = (newLevels - 1) - shelfIndex
      const cellIndex = (reversedShelfIndex * newRacks) + binIndex
      
      // Try to find an existing label with the same letter and bin
      const existingLabel = labelConfig.value.labels.find(l => 
        l.letter === letter && l.bin === bin
      )
      
      if (existingLabel) {
        // Keep existing label data but update selectedCell
        newLabels.push({
          ...existingLabel,
          selectedCell: cellIndex
        })
      } else {
        // Create new label with calculated selectedCell
        newLabels.push({
          rack: labelConfig.value.currentRack,
          letter: letter,
          bin: bin,
          height: 220,
          maxWeight: 180,
          selectedCell: cellIndex,
          qrContent: '',
          arrowDirection: 'right'
        })
      }
    }
  }
  
  // Replace all labels with the new array
  labelConfig.value.labels = newLabels
}, { immediate: true })

const totalPages = computed(() => {
  return Math.ceil((labelConfig.value.numLevels * labelConfig.value.numRacks) / 2)
})

// Group labels into pages
const labelPages = computed(() => {
  const pages = []
  const sortedLabels = [...labelConfig.value.labels].sort((a, b) => {
    // First sort by shelf letter
    if (a.letter !== b.letter) {
      return a.letter.localeCompare(b.letter)
    }
    // Then sort by bin number
    return parseInt(a.bin) - parseInt(b.bin)
  })
  
  for (let i = 0; i < sortedLabels.length; i += 2) {
    pages.push({
      topLabel: sortedLabels[i + 1],
      bottomLabel: sortedLabels[i]
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

// Computed property for total number of cells
const totalCells = computed(() => {
  return labelConfig.value.numLevels * labelConfig.value.numRacks
})

// Grid cell helper
const isSelectedCell = (index, selectedCell) => {
  return index === selectedCell
}

// Update handleCellClick to prevent manual cell selection
const handleCellClick = (label, cellIndex) => {
  // Disabled manual cell selection as it's now automatically managed
  return
}

// Template helper for grid style
const gridStyle = computed(() => {
  const baseSize = 100; // Base size in pixels
  const cellSize = baseSize / Math.max(labelConfig.value.numRacks, labelConfig.value.numLevels);
  
  return {
    'grid-template-columns': `repeat(${labelConfig.value.numRacks}, ${cellSize}px)`,
    'grid-template-rows': `repeat(${labelConfig.value.numLevels}, ${cellSize}px)`,
    'width': 'fit-content'
  }
})

// Helper to determine if letter is odd-indexed (A,C,E,...)
const isOddIndexedLetter = (letter) => {
  return (letter.charCodeAt(0) - 65) % 2 === 0
}

// Add new ref for tracking active label
const activeLabel = ref(null)
</script>

<template>
  <div class="container mx-auto p-4">
    <h1 class="text-2xl font-bold mb-4">Warehouse Label Generator</h1>
    
    <!-- Shelf and Rack Configuration -->
    <div class="bg-white p-4 rounded shadow mb-4">
      <div class="grid grid-cols-1 md:grid-cols-2 gap-4">
        <div>
          <label class="block text-sm font-medium mb-1">Number of Shelves</label>
          <div class="flex items-center">
            <button 
              @click="labelConfig.numLevels = Math.max(1, labelConfig.numLevels - 1)"
              class="px-3 py-2 bg-gray-200 hover:bg-gray-300 rounded-l border border-r-0"
            >
              -
            </button>
            <input 
              v-model="labelConfig.numLevels" 
              type="number" 
              min="1"
              class="w-full p-2 border-y text-center"
            >
            <button 
              @click="labelConfig.numLevels++"
              class="px-3 py-2 bg-gray-200 hover:bg-gray-300 rounded-r border border-l-0"
            >
              +
            </button>
          </div>
        </div>
        <div>
          <label class="block text-sm font-medium mb-1">Number of Racks</label>
          <div class="flex items-center">
            <button 
              @click="labelConfig.numRacks = Math.max(1, labelConfig.numRacks - 1)"
              class="px-3 py-2 bg-gray-200 hover:bg-gray-300 rounded-l border border-r-0"
            >
              -
            </button>
            <input 
              v-model="labelConfig.numRacks" 
              type="number" 
              min="1"
              class="w-full p-2 border-y text-center"
            >
            <button 
              @click="labelConfig.numRacks++"
              class="px-3 py-2 bg-gray-200 hover:bg-gray-300 rounded-r border border-l-0"
            >
              +
            </button>
          </div>
        </div>
      </div>
    </div>

    <div class="grid grid-cols-1 md:grid-cols-2 gap-4">
      <!-- Label Configuration Form -->
      <div class="bg-white p-4 rounded shadow">
        <h2 class="text-xl font-semibold mb-4">Label Configuration</h2>
        <div class="space-y-4">
          <!-- Dynamic forms based on number of levels -->
          <div v-for="(label, index) in [...labelConfig.labels].reverse()" :key="index" 
            class="p-4 border rounded space-y-4 transition-all duration-200 hover:border-orange-400"
            :class="{ 'border-orange-500 border-2': activeLabel === index }"
            @mouseenter="activeLabel = index"
            @mouseleave="activeLabel = null"
          >
            <h3 class="font-semibold">Shelf {{ labelConfig.labels.length - index }} Configuration</h3>
            
            <div class="grid grid-cols-3 gap-4">
              <div>
                <label class="block text-sm font-medium mb-1">Aisle Number</label>
                <input 
                  v-model="label.rack" 
                  type="text" 
                  class="w-full p-2 border rounded"
                  pattern="[0-9]{3}"
                  placeholder="001"
                  required
                  @input="updateRackNumber(label.rack)"
                >
              </div>
              <div>
                <label class="block text-sm font-medium mb-1">Shelf</label>
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

            <div class="grid grid-cols-1 md:grid-cols-2 gap-4">
              <div>
                <label class="block text-sm text-center font-medium mb-1">Selected Bin (click to select)</label>
                <div class="grid gap-0.5 mx-auto" :style="gridStyle">
                  <div v-for="i in totalCells" :key="i-1" 
                    class="border border-black cursor-pointer hover:bg-gray-200"
                    :class="{ 'bg-black hover:bg-black': isSelectedCell(i-1, label.selectedCell) }"
                    @click="handleCellClick(label, i-1)"
                  ></div>
                </div>
              </div>

              <div class="space-y-4">
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
        </div>
      </div>

      <!-- Label Preview -->
      <div class="bg-white p-4 rounded shadow">
        <h2 class="text-xl font-semibold mb-4">Label Preview - A6 Format</h2>
        
        <div v-for="(page, pageIndex) in [...labelPages].reverse()" :key="pageIndex" class="mb-8">
          <div 
            :id="'labelPage' + (labelPages.length - 1 - pageIndex)"
            class="border-2 border-gray-400 rounded mx-auto bg-white"
            style="width: 105mm; height: 148mm;"
          >
            <!-- Top Label -->
            <div v-if="page.topLabel" 
              class="h-[72mm] border-b border-dotted border-black p-5 flex flex-col transition-colors duration-200"
              :class="{ 'border-2 border-orange-500': activeLabel === ((labelPages.length - 1 - pageIndex) * 2 + 1) }"
            >
              <div class="text-5xl font-black tracking-wider text-center mb-2">
                {{ formattedLabel(page.topLabel) }}
              </div>
              <div class="text-1md font-bold tracking-wider text-center mb-2">
                {{ page.topLabel.qrContent }}
              </div>
              
              <div class="flex items-start">
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
                <div class="w-[60%]">
                  <!-- Grid and Metrics Container -->
                  <div class="flex items-start">
                    
                    <!-- Metrics -->
                    <div class="w-1/3">
                      <div class="space-y-1">
                        <div class="border border-black px-1 py-0.5 whitespace-nowrap">
                          <div class="text-[10px]">HEIGHT CM</div>
                          <div class="text-lg font-bold">{{ page.topLabel.height }}</div>
                        </div>
                        <div class="border border-black px-1 py-0.5 whitespace-nowrap">
                          <div class="text-[10px]">MAX WEIGHT</div>
                          <div class="text-lg font-bold">{{ page.topLabel.maxWeight }} KG</div>
                        </div>
                      </div>
                    </div>

                    <!-- Grid -->
                    <div class="w-2/3 pr-2 flex justify-end">
                      <div class="grid gap-0.5" :style="gridStyle">
                        <div v-for="i in totalCells" :key="i-1" 
                          class="border border-black"
                          :class="{ 'bg-black': isSelectedCell(i-1, page.topLabel.selectedCell) }"
                        ></div>
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
              'p-5 flex flex-col transition-colors duration-200',
              {
                'h-[72mm]': page.topLabel,
                'h-[148mm]': !page.topLabel,
                'border-2 border-orange-500': activeLabel === ((labelPages.length - 1 - pageIndex) * 2)
              }
            ]">
              <div class="flex-1"></div>
              <div>
                <div class="text-5xl font-black tracking-wider text-center mb-1">
                  {{ formattedLabel(page.bottomLabel) }}
                </div>
                <div class="text-1md font-bold tracking-wider text-center mb-2">
                  {{ page.bottomLabel.qrContent }}
                </div>
                
                <div class="flex items-start">
                  <!-- Left Column (60%) -->
                  <div class="w-2/3">
                    <!-- Grid and Metrics Container -->
                    <div class="flex items-start">
                      <!-- Grid -->
                      <div class="w-2/3 pr-2">
                        <div class="grid gap-0.5" :style="gridStyle">
                          <div v-for="i in totalCells" :key="i-1" 
                            class="border border-black"
                            :class="{ 'bg-black': isSelectedCell(i-1, page.bottomLabel.selectedCell) }"
                          ></div>
                        </div>
                      </div>
                      
                      <!-- Metrics -->
                      <div class="w-1/3">
                        <div class="space-y-1">
                          <div class="border border-black px-1 py-0.5 whitespace-nowrap">
                            <div class="text-[10px]">HEIGHT CM</div>
                            <div class="text-lg font-bold">{{ page.bottomLabel.height }}</div>
                          </div>
                          <div class="border border-black px-1 py-0.5 whitespace-nowrap">
                            <div class="text-[10px]">MAX WEIGHT</div>
                            <div class="text-lg font-bold">{{ page.bottomLabel.maxWeight }} KG</div>
                          </div>
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
