<template>
  <div class="pixel-board-container">
    <h2>Community Pixel Board</h2>
    <div class="canvas-container">
      <canvas 
        ref="canvas"
        :width="CANVAS_SIZE"
        :height="CANVAS_SIZE"
        @mousedown="startDrawing"
        @mousemove="draw"
        @mouseup="stopDrawing"
        @mouseleave="stopDrawing"
      ></canvas>
    </div>
    
    <div class="controls">
      <div class="color-picker">
        <input type="color" v-model="selectedColor" />
        <span>Selected Color</span>
      </div>
      
      <div class="cost-container">
        <div class="cost-info">
          <span class="label">Cost per pixel:</span> <span class="gala-amount">1 GALA</span>
        </div>

        <div v-if="modifiedPixels.size > 0" class="total-cost">
          <span class="label">Total cost:</span> <span class="gala-amount">{{ modifiedPixels.size + 1 }} GALA</span>
        </div>
      </div>

      <div class="action-container">
        <div class="button-group">
          <button 
            @click="saveCanvas" 
            :disabled="!hasChanges || isProcessing"
            :class="{ 'has-pixels': hasChanges && !isProcessing }"
          >
            {{ isProcessing ? 'Processing...' : 'Claim Pixels' }}
          </button>
          <button 
            v-if="hasChanges"
            @click="clearUnsaved"
            class="clear-button"
            :disabled="isProcessing"
          >
            Clear
          </button>
        </div>
        <div class="fee-notice">Network fee: 1 GALA</div>
        <div v-if="statusMessage" class="status-message" :class="{ error: isError }">
          {{ statusMessage }}
        </div>
      </div>
    </div>
  </div>
</template>

<script setup lang="ts">
import { ref, onMounted } from 'vue'
import type { MetamaskConnectClient } from '@gala-chain/connect'
import { initializeApp } from 'firebase/app'
import { getDatabase, ref as dbRef, set, onValue, get } from 'firebase/database'

// Initialize Firebase
const firebaseConfig = {
  apiKey: import.meta.env.VITE_FIREBASE_API_KEY,
  projectId: import.meta.env.VITE_FIREBASE_PROJECT_ID,
  databaseURL: import.meta.env.VITE_FIREBASE_DATABASE_URL,
}

console.log('Firebase config:', {
  projectId: firebaseConfig.projectId,
  databaseURL: firebaseConfig.databaseURL
})

const app = initializeApp(firebaseConfig)
const db = getDatabase(app)
console.log('Firebase initialized:', {
  dbRef: dbRef(db, 'pixelboard').toString()
})

const CANVAS_SIZE = 500
const PIXEL_SIZE = 10
const GRID_SIZE = CANVAS_SIZE / PIXEL_SIZE

const props = defineProps<{
  walletAddress: string
  metamaskClient: MetamaskConnectClient
}>()

const canvas = ref<HTMLCanvasElement | null>(null)
const ctx = ref<CanvasRenderingContext2D | null>(null)
const isDrawing = ref(false)
const selectedColor = ref('#000000')
const hasChanges = ref(false)
const isProcessing = ref(false)
const modifiedPixels = ref(new Set<string>())
const statusMessage = ref('')
const isError = ref(false)

interface PixelData {
  x: number
  y: number
  color: string
}

// Track saved and unsaved pixels separately
const savedPixels = ref<Record<string, PixelData>>({})
const unsavedPixels = ref<Record<string, PixelData>>({})

onMounted(async () => {
  if (!canvas.value) return
  ctx.value = canvas.value.getContext('2d')
  if (!ctx.value) return
  
  drawGrid()
  
  // Listen for pixel updates
  const pixelsRef = dbRef(db, 'pixels')
  onValue(pixelsRef, 
    (snapshot) => {
      const data = snapshot.val()
      if (data) {
        loadPixels(data)
      }
    },
    (error) => {
      console.error('Firebase read error:', error)
    }
  )
})

function drawGrid() {
  if (!ctx.value) return
  
  ctx.value.strokeStyle = '#ddd'
  ctx.value.beginPath()
  
  for (let i = 0; i <= CANVAS_SIZE; i += PIXEL_SIZE) {
    ctx.value.moveTo(i, 0)
    ctx.value.lineTo(i, CANVAS_SIZE)
    ctx.value.moveTo(0, i)
    ctx.value.lineTo(CANVAS_SIZE, i)
  }
  
  ctx.value.stroke()
}

function startDrawing(event: MouseEvent) {
  isDrawing.value = true
  draw(event)
}

function draw(event: MouseEvent) {
  if (!isDrawing.value || !ctx.value || !canvas.value) return
  
  const rect = canvas.value.getBoundingClientRect()
  const x = Math.floor((event.clientX - rect.left) / PIXEL_SIZE) * PIXEL_SIZE
  const y = Math.floor((event.clientY - rect.top) / PIXEL_SIZE) * PIXEL_SIZE
  
  ctx.value.fillStyle = selectedColor.value
  ctx.value.fillRect(x, y, PIXEL_SIZE, PIXEL_SIZE)
  
  // Track modified pixels with their colors
  const key = `${x},${y}`
  unsavedPixels.value[key] = {
    x,
    y,
    color: selectedColor.value
  }
  modifiedPixels.value.add(key)
  hasChanges.value = true
}

function stopDrawing() {
  isDrawing.value = false
}

async function saveCanvas() {
  if (!ctx.value || !canvas.value || modifiedPixels.value.size === 0) return
  
  isProcessing.value = true
  statusMessage.value = 'Waiting for wallet confirmation...'
  isError.value = false
  
  try {
    // Calculate GALA cost (1 GALA per pixel)
    const burnAmount = modifiedPixels.value.size
    
    // Create burn transaction
    const burnTokensDto = {
      owner: props.walletAddress,
      tokenInstances: [{
        quantity: burnAmount.toString(),
        tokenInstanceKey: {
          collection: "GALA",
          category: "Unit",
          type: "none",
          additionalKey: "none",
          instance: "0"
        }
      }],
      uniqueKey: `january-2025-event-${import.meta.env.VITE_PROJECT_ID}-${Date.now()}`
    }

    // Sign and submit burn transaction
    const signedBurnDto = await props.metamaskClient.sign("BurnTokens", burnTokensDto)
    const response = await fetch(`${import.meta.env.VITE_BURN_GATEWAY_API}/BurnTokens`, {
      method: 'POST',
      headers: { 'Content-Type': 'application/json' },
      body: JSON.stringify(signedBurnDto)
    })

    if (!response.ok) {
      throw new Error('Failed to burn tokens')
    }

    // Merge saved and unsaved pixels
    const updates = {
      ...savedPixels.value,
      ...unsavedPixels.value
    }
    
    // Save to Firebase
    await set(dbRef(db, 'pixels'), updates)
    console.log('Pixels saved successfully')
    
    // Update saved pixels
    savedPixels.value = updates
    
    // Clear unsaved tracking
    unsavedPixels.value = {}
    modifiedPixels.value.clear()
    hasChanges.value = false
    
    statusMessage.value = 'Pixels claimed successfully!'
    setTimeout(() => {
      statusMessage.value = ''
    }, 3000)
    
  } catch (error) {
    console.error('Failed to save canvas:', error)
    statusMessage.value = 'Failed to save changes'
    isError.value = true
  } finally {
    isProcessing.value = false
  }
}

function loadPixels(data: Record<string, PixelData>) {
  if (!ctx.value) return
  
  try {
    // Store as saved pixels
    savedPixels.value = data || {}
    
    // Clear canvas
    ctx.value.clearRect(0, 0, CANVAS_SIZE, CANVAS_SIZE)
    drawGrid()
    
    // Draw saved pixels
    Object.values(savedPixels.value).forEach(pixel => {
      ctx.value!.fillStyle = pixel.color
      ctx.value!.fillRect(pixel.x, pixel.y, PIXEL_SIZE, PIXEL_SIZE)
    })
    
    console.log('Pixels loaded successfully')
  } catch (error) {
    console.error('Failed to load pixels:', error)
  }
}

function clearUnsaved() {
  if (!ctx.value) return
  
  // Clear canvas
  ctx.value.clearRect(0, 0, CANVAS_SIZE, CANVAS_SIZE)
  drawGrid()
  
  // Redraw only saved pixels
  Object.values(savedPixels.value).forEach(pixel => {
    ctx.value!.fillStyle = pixel.color
    ctx.value!.fillRect(pixel.x, pixel.y, PIXEL_SIZE, PIXEL_SIZE)
  })
  
  // Clear unsaved tracking
  unsavedPixels.value = {}
  modifiedPixels.value.clear()
  hasChanges.value = false
  statusMessage.value = ''
}
</script>

<style scoped>
.pixel-board-container {
  margin: 20px 0;
  padding: 20px;
  border: 1px solid #ddd;
  border-radius: 8px;
}

.canvas-container {
  margin: 20px 0;
  display: flex;
  justify-content: center;
}

canvas {
  border: 1px solid #ccc;
  cursor: crosshair;
}

.controls {
  display: flex;
  gap: 20px;
  align-items: center;
  justify-content: center;
  margin: 20px 0;
}

.color-picker {
  display: flex;
  flex-direction: column;
  align-items: center;
  gap: 8px;
}

.color-picker input {
  width: 50px;
  height: 50px;
  padding: 0;
  border: 2px solid #ddd;
  border-radius: 4px;
  cursor: pointer;
}

.color-picker span {
  font-size: 0.9em;
  color: #666;
}

.ipfs-info {
  text-align: center;
  margin-top: 10px;
}

.ipfs-info a {
  color: #666;
  text-decoration: none;
}

.ipfs-info a:hover {
  text-decoration: underline;
}

.error-message {
  color: #ff4444;
  text-align: center;
  margin: 10px 0;
  padding: 10px;
  background: #ffeeee;
  border-radius: 4px;
}

.cost-container {
  display: flex;
  flex-direction: column;
  gap: 8px;
  min-width: 200px;
  text-align: center;
}

.cost-info, .total-cost {
  font-size: 0.95em;
}

.label {
  color: #666;
}

.gala-amount {
  font-weight: 600;
  color: var(--primary-color, #4CAF50);
}

.action-container {
  display: flex;
  flex-direction: column;
  align-items: center;
  gap: 8px;
}

.fee-notice {
  color: #666;
  font-size: 0.85em;
  margin-top: 4px;
}

.status-message {
  font-size: 0.9em;
  color: #4CAF50;
  text-align: center;
  min-height: 20px;
}

.status-message.error {
  color: #f44336;
}

button {
  padding: 10px 20px;
  border: 1px solid #ddd;
  border-radius: 4px;
  background: #f5f5f5;
  color: #333;
  cursor: pointer;
  transition: all 0.3s ease;
  font-size: 1em;
  min-width: 150px;
}

button.has-pixels {
  background: #4CAF50;
  color: white;
  border-color: #4CAF50;
}

button:disabled {
  background-color: #cccccc;
  cursor: not-allowed;
  opacity: 0.7;
}

button:hover:not(:disabled) {
  opacity: 0.9;
}

.button-group {
  display: flex;
  gap: 10px;
  align-items: center;
}

.clear-button {
  background: transparent;
  border-color: #666;
  color: #666;
  padding: 10px 15px;
  min-width: auto;
}

.clear-button:hover:not(:disabled) {
  background: #f0f0f0;
}
</style> 