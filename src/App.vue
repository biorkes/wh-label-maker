<script setup>
import { ref, computed } from 'vue'
import QRCode from 'qrcode.vue'
import { jsPDF } from 'jspdf'
import html2canvas from 'html2canvas'
import './style.css'

const labelConfig = ref({
  rack: '001',
  level: 'A',
  bin: '001',
  width: 100,
  height: 50,
  maxWeight: 100,
  arrow: '',
  arrowPosition: 'left',
  customQRContent: ''
})

const formattedLabel = computed(() => {
  return `${labelConfig.value.rack} ${labelConfig.value.level} ${labelConfig.value.bin}`
})

const qrCodeContent = computed(() => {
  return labelConfig.value.customQRContent || formattedLabel.value
})

const generatePDF = async () => {
  const element = document.getElementById('labelPreview')
  const canvas = await html2canvas(element, {
    scale: 2,
    useCORS: true
  })
  
  const imgData = canvas.toDataURL('image/png')
  const pdf = new jsPDF({
    orientation: 'landscape',
    unit: 'cm',
    format: [15, 10] // A6 format landscape
  })
  
  pdf.addImage(imgData, 'PNG', 0, 0, 15, 10)
  pdf.save('warehouse-label.pdf')
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
          <div class="grid grid-cols-3 gap-4">
            <div>
              <label class="block text-sm font-medium mb-1">Rack Number</label>
              <input 
                v-model="labelConfig.rack" 
                type="text" 
                class="w-full p-2 border rounded"
                pattern="[0-9]{3}"
                placeholder="001"
              >
            </div>
            <div>
              <label class="block text-sm font-medium mb-1">Level</label>
              <input 
                v-model="labelConfig.level" 
                type="text" 
                class="w-full p-2 border rounded"
                pattern="[A-Z]"
                placeholder="A"
              >
            </div>
            <div>
              <label class="block text-sm font-medium mb-1">Bin Number</label>
              <input 
                v-model="labelConfig.bin" 
                type="text" 
                class="w-full p-2 border rounded"
                pattern="[0-9]{3}"
                placeholder="001"
              >
            </div>
          </div>

          <div class="grid grid-cols-2 gap-4">
            <div>
              <label class="block text-sm font-medium mb-1">Bin Width (cm)</label>
              <input 
                v-model="labelConfig.width" 
                type="number" 
                class="w-full p-2 border rounded"
              >
            </div>
            <div>
              <label class="block text-sm font-medium mb-1">Bin Height (cm)</label>
              <input 
                v-model="labelConfig.height" 
                type="number" 
                class="w-full p-2 border rounded"
              >
            </div>
          </div>

          <div>
            <label class="block text-sm font-medium mb-1">Max Weight (kg)</label>
            <input 
              v-model="labelConfig.maxWeight" 
              type="number" 
              class="w-full p-2 border rounded"
            >
          </div>

          <div>
            <label class="block text-sm font-medium mb-1">Arrow Direction</label>
            <select v-model="labelConfig.arrow" class="w-full p-2 border rounded">
              <option value="">No Arrow</option>
              <option value="up">Up Arrow</option>
              <option value="down">Down Arrow</option>
            </select>
          </div>

          <div>
            <label class="block text-sm font-medium mb-1">Arrow Position</label>
            <select 
              v-model="labelConfig.arrowPosition" 
              class="w-full p-2 border rounded"
              :disabled="!labelConfig.arrow"
            >
              <option value="left">Left</option>
              <option value="right">Right</option>
            </select>
          </div>

          <div>
            <label class="block text-sm font-medium mb-1">
              Custom QR Code Content 
              <span class="text-gray-500 text-xs">(Optional - leave empty to use location code)</span>
            </label>
            <input 
              v-model="labelConfig.customQRContent" 
              type="text" 
              class="w-full p-2 border rounded"
              placeholder="Enter custom content for QR code"
            >
          </div>
        </div>
      </div>

      <!-- Label Preview -->
      <div class="bg-white p-4 rounded shadow">
        <h2 class="text-xl font-semibold mb-4">Label Preview</h2>
        <div 
          id="labelPreview" 
          class="border-2 border-gray-400 rounded p-4 w-[15cm] h-[10cm] mx-auto flex flex-col"
          style="aspect-ratio: 3/2;"
        >
          <!-- Top Section - Location Code -->
          <div class="flex-none h-[30%] flex items-center justify-center">
            <div class="text-8xl font-black tracking-wider whitespace-nowrap">
              {{ formattedLabel }}
            </div>
          </div>

          <!-- Bottom Section - Grid Layout -->
          <div class="flex-1 grid grid-cols-3 gap-4">
            <!-- 1. Arrow Section -->
            <div class="flex items-center justify-center">
              <div v-if="labelConfig.arrow" class="flex items-center justify-center w-full h-full">
                <div v-if="labelConfig.arrow === 'up'" class="text-[200px] leading-none font-black">⬆</div>
                <div v-else-if="labelConfig.arrow === 'down'" class="text-[200px] leading-none font-black">⬇</div>
              </div>
            </div>

            <!-- 2. Metrics Section -->
            <div class="flex items-center justify-center">
              <div class="text-xl font-bold space-y-3">
                <div class="flex items-center gap-2">
                  <span class="text-gray-600">Width:</span>
                  <span class="text-2xl">{{ labelConfig.width }}</span>
                  <span class="text-gray-600">cm</span>
                </div>
                <div class="flex items-center gap-2">
                  <span class="text-gray-600">Height:</span>
                  <span class="text-2xl">{{ labelConfig.height }}</span>
                  <span class="text-gray-600">cm</span>
                </div>
                <div class="flex items-center gap-2">
                  <span class="text-gray-600">Max:</span>
                  <span class="text-2xl">{{ labelConfig.maxWeight }}</span>
                  <span class="text-gray-600">kg</span>
                </div>
              </div>
            </div>

            <!-- 3. QR Code Section -->
            <div class="flex items-center justify-center">
              <QRCode 
                :value="qrCodeContent"
                :size="250"
                level="H"
                render-as="svg"
              />
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
</style>
