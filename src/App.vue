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
  showShelfDivider: false,
  maxWeightPerShelf: '',
  skipAShelf: false,
  shelfHeights: {},
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
    format: 'a6',
    compress: true,
    precision: 4
  })
  
  // Get total pages and iterate in reverse to match preview order
  const totalPagesCount = totalPages.value
  for (let i = 0; i < totalPagesCount; i++) {
    if (i > 0) pdf.addPage()
    
    // Calculate the correct page index to match preview order
    const pageIndex = totalPagesCount - 1 - i
    const element = document.getElementById(`labelPage${pageIndex}`)
    const canvas = await html2canvas(element, {
      scale: 3,
      useCORS: true,
      scrollX: 0,
      scrollY: 0,
      imageTimeout: 0,
      removeContainer: true,
      logging: false,
      backgroundColor: '#ffffff',
      allowTaint: true,
      letterRendering: true,
      windowWidth: element.scrollWidth,
      windowHeight: element.scrollHeight,
      onclone: (clonedDoc) => {
        const clonedElement = clonedDoc.getElementById(`labelPage${pageIndex}`)
        if (clonedElement) {
          clonedElement.style.transform = 'none'
        }
      }
    })
    
    const imgData = canvas.toDataURL('image/jpeg', 1.0)
    pdf.addImage(imgData, 'JPEG', 0, 0, 105, 148, undefined, 'MEDIUM')
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

// Add weight distribution calculation
const calculateDistributedWeight = (totalWeight, numShelves, numRacks, skipAShelf) => {
  const effectiveShelves = skipAShelf ? numShelves - 1 : numShelves
  if (effectiveShelves <= 0 || numRacks <= 0) return 0
  
  const weightPerShelf = Math.floor(totalWeight / effectiveShelves)
  return Math.floor(weightPerShelf / numRacks)
}

// Watch for changes in weight-related configurations
watch(
  [
    () => labelConfig.value.maxWeightPerShelf,
    () => labelConfig.value.numLevels,
    () => labelConfig.value.numRacks,
    () => labelConfig.value.skipAShelf
  ],
  ([newWeight, numLevels, numRacks, skipAShelf]) => {
    if (!newWeight) return
    
    const totalWeight = parseInt(newWeight)
    if (isNaN(totalWeight)) return
    
    const distributedWeight = calculateDistributedWeight(totalWeight, numLevels, numRacks, skipAShelf)
    
    labelConfig.value.labels.forEach(label => {
      // If skipAShelf is true and this is shelf A, set weight to 0
      if (skipAShelf && label.letter === 'A') {
        label.maxWeight = 0
      } else {
        label.maxWeight = distributedWeight
      }
    })
  }
)

// Add watch for shelf heights
watch(
  () => labelConfig.value.shelfHeights,
  (newHeights) => {
    labelConfig.value.labels.forEach(label => {
      if (newHeights[label.letter] !== undefined) {
        label.height = newHeights[label.letter]
      }
    })
  },
  { deep: true }
)

// Add this new function after the existing imports
const parseCsvRow = (row) => {
  const [aisle, shelf, bin, height, maxWeight, arrow, qrContent] = row.split(',').map(val => val.trim())
  return {
    rack: aisle.padStart(3, '0'),
    letter: shelf.toUpperCase(),
    bin: bin.padStart(3, '0'),
    height: parseInt(height) || 220,
    maxWeight: parseInt(maxWeight) || 180,
    selectedCell: null, // This will be calculated later
    qrContent: qrContent || '',
    arrowDirection: arrow === '0' ? 'left' : 'right'
  }
}

// Add this method before the existing watchers
const handleCsvUpload = async (event) => {
  const file = event.target.files[0]
  if (!file) return

  try {
    const text = await file.text()
    const rows = text.split('\n')
      .map(row => row.trim())
      .filter(row => row.length > 0) // Remove empty rows
    
    // Remove header row if present
    if (rows[0].toLowerCase().includes('aisle') || rows[0].toLowerCase().includes('shelf')) {
      rows.shift()
    }

    // Parse CSV data
    const labels = rows.map(row => parseCsvRow(row))
    
    // Update configuration based on the imported data
    const uniqueShelves = new Set(labels.map(l => l.letter))
    const uniqueRacks = new Set(labels.map(l => l.bin))
    
    // Update the configuration
    labelConfig.value.numLevels = uniqueShelves.size
    labelConfig.value.numRacks = uniqueRacks.size
    labelConfig.value.currentRack = labels[0]?.rack || '001'
    
    // Calculate selectedCell for each label
    labels.forEach(label => {
      const shelfIndex = label.letter.charCodeAt(0) - 65
      const binIndex = parseInt(label.bin) - 1
      const reversedShelfIndex = (labelConfig.value.numLevels - 1) - shelfIndex
      label.selectedCell = (reversedShelfIndex * labelConfig.value.numRacks) + binIndex
    })
    
    // Update all labels
    labelConfig.value.labels = labels
    
    // Reset file input
    event.target.value = ''
  } catch (error) {
    console.error('Error processing CSV:', error)
    alert('Error processing CSV file. Please check the format and try again.')
  }
}

// Add this after handleCsvUpload function
const downloadExampleCsv = () => {
  const exampleData = `Aisle,Shelf,Bin,Height (cm),Max Weight (kg),Arrow (1=right/0=left),QR Code
1,A,1,220,150,1,custom_qr_1
1,A,2,220,180,0,custom_qr_2
1,B,1,240,200,1,custom_qr_3
1,B,2,240,190,0,custom_qr_4
2,A,1,220,160,1,custom_qr_5
2,A,2,220,170,0,custom_qr_6`

  const blob = new Blob([exampleData], { type: 'text/csv' })
  const url = window.URL.createObjectURL(blob)
  const a = document.createElement('a')
  a.setAttribute('href', url)
  a.setAttribute('download', 'warehouse-labels-example.csv')
  a.click()
  window.URL.revokeObjectURL(url)
}
</script>

<template>
  <div class="container mx-auto p-4">
    <h1 class="text-2xl font-bold mb-4">Warehouse Label Generator</h1>
    
    <!-- Shelf and Rack Configuration -->
    <div class="bg-white p-4 rounded shadow mb-4 grid grid-cols-1 md:grid-cols-2 gap-4">

      <!-- Layout Configuration -->
      <div>
        <div class="border-b pb-4 mb-4">
          <h3 class="text-sm font-semibold mb-2">Layout Configuration</h3>
          <div class="grid grid-cols-2 gap-4">
            <div>
              <label class="block text-xs font-medium mb-1">Number of Shelves</label>
              <div class="flex items-center">
                <button 
                  @click="labelConfig.numLevels = Math.max(1, labelConfig.numLevels - 1)"
                  class="px-2 py-1 bg-gray-200 hover:bg-gray-300 rounded-l border border-r-0"
                >
                  -
                </button>
                <input 
                  v-model="labelConfig.numLevels" 
                  type="number" 
                  min="1"
                  class="w-full p-1 border-y text-center text-sm"
                >
                <button 
                  @click="labelConfig.numLevels++"
                  class="px-2 py-1 bg-gray-200 hover:bg-gray-300 rounded-r border border-l-0"
                >
                  +
                </button>
              </div>
            </div>
            <div>
              <label class="block text-xs font-medium mb-1">Number of Racks</label>
              <div class="flex items-center">
                <button 
                  @click="labelConfig.numRacks = Math.max(1, labelConfig.numRacks - 1)"
                  class="px-2 py-1 bg-gray-200 hover:bg-gray-300 rounded-l border border-r-0"
                >
                  -
                </button>
                <input 
                  v-model="labelConfig.numRacks" 
                  type="number" 
                  min="1"
                  class="w-full p-1 border-y text-center text-sm"
                >
                <button 
                  @click="labelConfig.numRacks++"
                  class="px-2 py-1 bg-gray-200 hover:bg-gray-300 rounded-r border border-l-0"
                >
                  +
                </button>
              </div>
            </div>
          </div>
        </div>

        <!-- Height Configuration -->
        <div class="border-b pb-4 mb-4">
          <h3 class="text-sm font-semibold mb-2">Height Configuration</h3>
          <div class="grid grid-cols-4 gap-2">
            <div v-for="i in labelConfig.numLevels" :key="i">
              <label class="block text-xs font-medium mb-1">
                Shelf {{ String.fromCharCode(64 + i) }}
              </label>
              <input 
                v-model="labelConfig.shelfHeights[String.fromCharCode(64 + i)]"
                type="number"
                class="w-full p-1 border rounded text-center text-sm"
                placeholder="cm"
              >
            </div>
          </div>
        </div>

        <!-- Weight Configuration -->
        <div class="border-b pb-4 mb-4">
          <h3 class="text-sm font-semibold mb-2">Weight Configuration</h3>
          <div class="space-y-2">
            <div>
              <label class="block text-xs font-medium mb-1">Max Weight Per Shelf</label>
              <input 
                v-model="labelConfig.maxWeightPerShelf"
                type="number"
                min="0"
                class="w-full p-1 border rounded text-center text-sm"
                placeholder="Enter total weight in kg"
              >
            </div>
            <div class="flex items-center">
              <input 
                type="checkbox" 
                id="skipAShelf" 
                v-model="labelConfig.skipAShelf"
                class="w-3 h-3 text-blue-600 rounded focus:ring-blue-500"
              >
              <label for="skipAShelf" class="ml-2 text-xs font-medium">
                Skip A Shelf in Weight Distribution
              </label>
            </div>
          </div>
        </div>
        
        <!-- Visual Options -->
        <div>
          <h3 class="text-sm font-semibold mb-2">Visual Options</h3>
          <div class="flex items-center">
            <input 
              type="checkbox" 
              id="shelfDivider" 
              v-model="labelConfig.showShelfDivider"
              class="w-3 h-3 text-blue-600 rounded focus:ring-blue-500"
            >
            <label for="shelfDivider" class="ml-2 text-xs font-medium">
              Show Thick Divider Between Different Shelves
            </label>
          </div>
        </div>

      </div>

      <div>
        <!-- Bulk Import -->
        <div class="mt-4">
          <h3 class="text-sm font-semibold mb-2">Bulk Import</h3>
          <div class="space-y-2">
            <div>
              <label class="block text-xs font-medium mb-1">Import from CSV</label>
              <input 
                type="file" 
                accept=".csv"
                @change="handleCsvUpload"
                class="block w-full text-xs text-gray-500
                  file:mr-4 file:py-2 file:px-4
                  file:rounded file:border-0
                  file:text-sm file:font-semibold
                  file:bg-blue-50 file:text-blue-700
                  hover:file:bg-blue-100"
              >
            </div>
            <div class="text-xs text-gray-500">
              CSV Format: Aisle,Shelf,Bin,Height (cm),Max Weight (kg),Arrow (1=right/0=left),QR Code
            </div>
            <button 
              @click="downloadExampleCsv"
              class="mt-2 text-xs text-blue-600 hover:text-blue-800 underline"
            >
              Download Example CSV
            </button>
          </div>
        </div>
      </div>

    </div>

    <div class="grid grid-cols-1 md:grid-cols-2 gap-4">
      <!-- Label Configuration Form -->
      <div class="bg-white p-4 rounded shadow">
        <h2 class="text-xl font-semibold mb-4">Label Configuration</h2>
        <div class="space-y-3">
          <!-- Dynamic forms based on number of levels -->
          <div v-for="(label, index) in [...labelConfig.labels].reverse()" :key="index" 
            class="p-3 border rounded space-y-3 transition-all duration-200 hover:border-orange-400"
            :class="{ 'border-orange-500 border-2': activeLabel === index }"
            @mouseenter="activeLabel = index"
            @mouseleave="activeLabel = null"
          >
            <div class="flex justify-between items-center">
              <h3 class="text-sm font-semibold">Bin # {{ labelConfig.labels.length - index }}</h3>
              <div class="text-xs text-gray-500">{{ formattedLabel(label) }}</div>
            </div>
            
            <!-- Basic Info -->
            <div class="grid grid-cols-3 gap-2">
              <div>
                <label class="block text-xs font-medium mb-1">Aisle</label>
                <input 
                  v-model="label.rack" 
                  type="text" 
                  class="w-full p-1 border rounded text-sm text-center"
                  pattern="[0-9]{3}"
                  placeholder="001"
                  required
                  @input="updateRackNumber(label.rack)"
                >
              </div>
              <div>
                <label class="block text-xs font-medium mb-1">Shelf</label>
                <input 
                  v-model="label.letter" 
                  type="text" 
                  class="w-full p-1 border rounded text-sm text-center"
                  maxlength="1"
                  pattern="[A-Za-z]"
                  required
                >
              </div>
              <div>
                <label class="block text-xs font-medium mb-1">Bin</label>
                <input 
                  v-model="label.bin" 
                  type="text" 
                  class="w-full p-1 border rounded text-sm text-center"
                  pattern="[0-9]{3}"
                  placeholder="002"
                  required
                >
              </div>
            </div>

            <!-- Metrics -->
            <div class="grid grid-cols-2 gap-2">
              <div>
                <label class="block text-xs font-medium mb-1">Height (cm)</label>
                <input 
                  v-model="label.height" 
                  type="number" 
                  class="w-full p-1 border rounded text-sm text-center"
                  required
                >
              </div>
              <div>
                <label class="block text-xs font-medium mb-1">Max Weight (kg)</label>
                <input 
                  v-model="label.maxWeight" 
                  type="number" 
                  class="w-full p-1 border rounded text-sm text-center"
                  required
                >
              </div>
            </div>

            <div class="grid grid-cols-2 gap-4">
              <!-- Left Column -->
              <div class="space-y-2">
                <div>
                  <label class="block text-xs font-medium mb-1">QR Code Content</label>
                  <input 
                    v-model="label.qrContent" 
                    type="text" 
                    class="w-full p-1 border rounded text-sm"
                    placeholder="Enter QR code content"
                  >
                </div>

                <div>
                  <label class="block text-xs font-medium mb-1">Arrow Direction</label>
                  <select 
                    v-model="label.arrowDirection" 
                    class="w-full p-1 border rounded text-sm"
                    required
                  >
                    <option value="right">Right</option>
                    <option value="left">Left</option>
                  </select>
                </div>
              </div>

              <!-- Right Column -->
              <div>
                <label class="block text-xs text-center font-medium mb-1">Selected Position</label>
                <div class="grid gap-0.5 mx-auto" :style="gridStyle">
                  <div v-for="i in totalCells" :key="i-1" 
                    class="border border-black cursor-pointer hover:bg-gray-200"
                    :class="{ 'bg-black hover:bg-black': isSelectedCell(i-1, label.selectedCell) }"
                    @click="handleCellClick(label, i-1)"
                  ></div>
                </div>
              </div>
            </div>
          </div>
        </div>
      </div>

      <!-- Label Preview -->
      <div class="p-4 rounded shadow" style="background-color: #f0f8ff;">
        <h2 class="text-xl font-semibold mb-4">Label Preview - A6 Format</h2>
        
        <div v-for="(page, pageIndex) in [...labelPages].reverse()" :key="pageIndex" class="mb-8">
          <div 
            :id="'labelPage' + (labelPages.length - 1 - pageIndex)"
            class="border-2 border-gray-400 rounded mx-auto bg-white"
            style="width: 105mm; height: 148mm;"
          >
            <!-- Top Label -->
            <div v-if="page.topLabel" 
              class="h-[72mm] p-5 flex flex-col transition-colors duration-200 relative"
              :class="[
                { 'border-2 border-orange-500': activeLabel === ((labelPages.length - 1 - pageIndex) * 2 + 1) }
              ]"
            >
              <div 
                v-if="labelConfig.showShelfDivider && page.topLabel.letter !== page.bottomLabel.letter" 
                class="absolute bottom-0 left-0 right-0 flex flex-col items-center"
              >
                <div class="w-full h-[6px] bg-black"></div>
                <div class="w-full h-[6px] flex justify-between items-center mt-[2px]">
                  <div v-for="n in 20" :key="n" class="w-[6px] h-[6px] rounded-full bg-black"></div>
                </div>
              </div>
              <div 
                v-else 
                class="absolute bottom-0 left-0 right-0"
              >
                <div class="w-full border-b border-dotted border-black"></div>
              </div>
              <div class="text-5xl font-black tracking-wider text-center mb-2 text-black">
                {{ formattedLabel(page.topLabel) }}
              </div>
              <div class="text-1md font-bold tracking-wider text-center mb-2 text-black">
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
                          <div class="text-[10px]">MAX H (CM)</div>
                          <div class="text-lg font-bold">{{ page.topLabel.height }}</div>
                        </div>
                        <div class="border border-black px-1 py-0.5 whitespace-nowrap">
                          <div class="text-[10px]">MAX W (KG)</div>
                          <div class="text-lg font-bold leading-none">
                            <template v-if="labelConfig.skipAShelf && page.topLabel.letter === 'A'">
                              N/A
                            </template>
                            <template v-else>
                              ≤{{ page.topLabel.maxWeight }}
                            </template>
                          </div>
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
                <div class="text-5xl font-black tracking-wider text-center mb-1 text-black">
                  {{ formattedLabel(page.bottomLabel) }}
                </div>
                <div class="text-1md font-bold tracking-wider text-center mb-2 text-black">
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
                            <div class="text-[10px]">MAX H (CM)</div>
                            <div class="text-lg font-bold">{{ page.bottomLabel.height }}</div>
                          </div>
                          <div class="border border-black px-1 py-0.5 whitespace-nowrap">
                            <div class="text-[10px]">MAX W (KG)</div>
                            <div class="text-lg font-bold leading-none">
                              <template v-if="labelConfig.skipAShelf && page.bottomLabel.letter === 'A'">
                                N/A
                              </template>
                              <template v-else>
                                ≤{{ page.bottomLabel.maxWeight }}
                              </template>
                            </div>
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
  min-height: 100vh;
}

/* Fix for html2canvas PDF generation image shifting */
img {
  display: inline-block !important;
}

/* Ensure SVG elements (QR codes) are also properly handled */
svg {
  display: inline-block !important;
}

/* Add this to remove default margin and ensure content starts from top */
.container {
  margin: 0;
  margin-top: 0;
  padding-top: 1rem;
}
</style>
