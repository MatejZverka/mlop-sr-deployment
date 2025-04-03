<template>
  <div class="container mt-4">
    <header class="mb-3">
      <h1 class="text-start">ONNX Image Restoration</h1>
      <p class="text-start">1x scale Image Restoration using ASID, powered by Vue.js, Bootstrap and ONNX Runtime Web,
        targeting real-life photo degradation removal (blur, noise, compression).
        Currently supports WebGPU and WASM EPs with both FP32 and FP16 precision (requires Chrome 135 or newer). Uses
        custom tiling method to prevent ASID artifacts on image borders.
        <br>Made by Matej Zverka.
      </p>
    </header>
    <div class="row mb-3">
      <div class="col-md-6 canvas-container" @click="triggerFileUpload">
        <canvas ref="inputCanvas" class="border w-100" height="200"></canvas>
        <div v-if="!imageUploaded" class="canvas-placeholder">Click to upload an image</div>
        <input type="file" ref="fileInput" accept="image/*" class="d-none" @change="handleImageUpload" />
      </div>
      <div class="col-md-6 canvas-container">
        <canvas ref="outputCanvas" class="border w-100" height="200"></canvas>
        <div v-if="!outputImageDrawn" class="canvas-placeholder">Output image</div>
      </div>
    </div>
    <div class="text-center mb-1">
      <p id="statusText" class="text-center">Select EP, precision and tiling.</p>
    </div>
    <div class="d-flex justify-content-center gap-2">
      <div>
        <p class="ms-1 mb-1">EP:</p>
        <select v-model="executionProvider" class="form-select w-auto">
          <option>WebGPU</option>
          <option>WASM</option>
        </select>
      </div>
      <div>
        <p class="ms-1 mb-1">Precision:</p>
        <select v-model="precision" class="form-select w-auto">
          <option>FP32</option>
          <option>FP16</option>
        </select>
      </div>
      <div>
        <p class="ms-1 mb-1">Horizontal tiles:</p>
        <input v-model.number="horizontalTiles" type="number" class="form-control w-auto" placeholder="4" min="1">
      </div>
      <div>
        <p class="ms-1 mb-1">Vertical tiles:</p>
        <input v-model.number="verticalTiles" type="number" class="form-control w-auto" placeholder="4" min="1">
      </div>
      <div>
        <p class="ms-1 mb-1"></p>
        <br>
        <button class="btn btn-primary" :disabled="!imageUploaded" @click="loadModel">Process</button>
      </div>
      <div>
        <p class="ms-1 mb-1"></p>
        <br>
        <button id="downloadButton" class="btn btn-secondary" :disabled="!outputImageDrawn">Download</button>
      </div>
      <div>
        <p class="ms-1 mb-1"></p>
        <br>
        <button class="btn btn-success" :disabled="!imageUploaded" @click="clearCanvases">Clear</button>
      </div>
    </div>
  </div>
</template>

<script>
import * as ort from "https://cdn.jsdelivr.net/npm/onnxruntime-web@1.22.0-dev.20250325-afaf4a5e63/dist/ort.webgpu.mjs"

// define ORT environment variables (proxy does not work with WebGPU)
ort.env.wasm.proxy = false
ort.env.wasm.numThreads = 16
ort.env.wasm.simd = true

// clear console output during dev and from previous inference session
console.clear()

// define global variables
var session = null
var useFp16 = false
const padding = 8
const upscaleFactor = 1
const canvasScalingFactor = 2

// dynamically get the first occurence of model files with precision post-fixes
//const modelFiles = import.meta.glob('../models/*.onnx')
const modelFiles = import.meta.glob('/models/*.onnx')
const modelFilesKeys = Object.keys(modelFiles)
const fp32Model = modelFilesKeys.find(file => file.includes('_fp32'))
const fp16Model = modelFilesKeys.find(file => file.includes('_fp16'))
const modelPathFp32 = new URL(import.meta.env.BASE_URL + fp32Model.substring(1), import.meta.url).href
const modelPathFp16 = new URL(import.meta.env.BASE_URL + fp16Model.substring(1), import.meta.url).href

export default {
  name: "App",
  data() {
    return {
      originalImage: null,
      imageUploaded: false,        // controls inputCanvas placeholder text visibility
      outputImageDrawn: false,     // controls outputCanvas placeholder text visibility
      executionProvider: "WebGPU", // Default dropdown value
      precision: "FP32",           // Default dropdown value
      horizontalTiles: 4,          // Default tile value
      verticalTiles: 4,            // Default tile value
      processingStartTime: null,
      processingEndTime: null,
      totalProcessingTime: null
    };
  },
  methods: {
    triggerFileUpload() {
      this.$refs.fileInput.click()
    },

    handleImageUpload(event) {
      const file = event.target.files[0]
      //if (!file) return

      this.clearCanvases()

      const reader = new FileReader()
      reader.onload = (e) => {
        const img = new Image()
        img.src = e.target.result
        img.onload = () => {
          this.originalImage = img
          this.imageUploaded = true; // hide inputCanvas placeholder text

          const canvas = this.$refs.inputCanvas
          const ctx = canvas.getContext("2d")

          const displayWidth = canvas.clientWidth
          const displayHeight = canvas.clientHeight
          canvas.width = displayWidth * canvasScalingFactor
          canvas.height = displayHeight * canvasScalingFactor

          // scale inputCanvas context for high-resolution drawing
          ctx.scale(canvasScalingFactor, canvasScalingFactor)

          // clear previous inputCanvas contents
          ctx.clearRect(0, 0, canvas.width, canvas.height)

          // preserve aspect ratio inside inputCanvas
          const imgAspect = img.width / img.height
          const canvasAspect = displayWidth / displayHeight

          let drawWidth, drawHeight, offsetX, offsetY

          if (imgAspect > canvasAspect) {
            drawWidth = displayWidth
            drawHeight = displayWidth / imgAspect
            offsetX = 0;
            offsetY = (displayHeight - drawHeight) / 2
          } else {
            drawHeight = displayHeight
            drawWidth = displayHeight * imgAspect
            offsetX = (displayWidth - drawWidth) / 2
            offsetY = 0
          }

          // use higher quality inputCanvas rendering
          ctx.imageSmoothingEnabled = true
          ctx.imageSmoothingQuality = "high"

          ctx.drawImage(img, offsetX, offsetY, drawWidth, drawHeight)
        };
      };
      reader.readAsDataURL(file)
    },

    async loadModel() {
      if (this.horizontalTiles < 1) {
        this.horizontalTiles = 1
      }
      if (this.verticalTiles < 1) {
        this.verticalTiles = 1
      }

      document.getElementById("statusText").textContent = "Processing image..."
      this.processingStartTime = performance.now()

      const outputCanvas = this.$refs.outputCanvas
      const outputCtx = outputCanvas.getContext("2d")
      outputCtx.clearRect(0, 0, outputCanvas.width, outputCanvas.height)

      if (this.precision == "FP16") {
        useFp16 = true
      } else {
        useFp16 = false
      }

      var modelPath = useFp16 ? modelPathFp16 : modelPathFp32

      if (this.executionProvider == "WebGPU") {
        try {
          session = await ort.InferenceSession.create(modelPath, {
            executionProviders: [{
              name: 'webgpu',
              preferredLayout: 'NCHW' // WebGPU doesn't work without this; layout is different from CPU inference for some reason
            }],
            deviceType: 'gpu',
            enable_cpu_mem_arena: false, // disabled CPU memory arena since ORT memory usage is unpredictable
            preferredOutputLocation: 'gpu-buffer' // 'gpu' might also work
          });
          console.log(`Model ${modelPath} loaded successfully.`)
        } catch (e) {
          console.error('Failed to load model:', e)
          alert('Failed to load ONNX model. Try using WASM EP or adjusting tiling.')
        }
      } else {
        // use WebAssembly EP as an alternative to WebGPU since WebNN and WebGL don't work
        try {
          session = await ort.InferenceSession.create(modelPath, {
            executionProviders: [{
              name: 'wasm'
            }],
            deviceType: 'cpu',
            enable_cpu_mem_arena: false
          });
          console.log(`Model ${modelPath} loaded successfully.`)
        } catch (e) {
          console.error('Failed to load model:', e)
          alert('Failed to load ONNX model. Try using FP32 precision or adjusting tiling.')
        }
      }
      this.processImage()
    },

    async processImage() {
      let tileWidth = Math.floor(this.originalImage.width / this.horizontalTiles)
      let tileHeight = Math.floor(this.originalImage.height / this.verticalTiles)

      let numOfHorizontalTiles = this.horizontalTiles
      let numOfVerticalTiles = this.verticalTiles

      let tileWidthMod = this.originalImage.width % this.horizontalTiles
      let tileHeightMod = this.originalImage.height % this.verticalTiles

      const tiles = []

      for (let i = 0; i < numOfVerticalTiles; i++) {
        for (let j = 0; j < numOfHorizontalTiles; j++) {
          let x = j * tileWidth
          let y = i * tileHeight

          // calculate padded tile dimensions
          let paddedWidth = tileWidth
          let paddedHeight = tileHeight
          if (tileWidthMod > 0 && j == (numOfHorizontalTiles - 1)) {
            paddedWidth += tileWidthMod
          }
          if (tileHeightMod > 0 && i == (numOfVerticalTiles - 1)) {
            paddedHeight += tileHeightMod
          }

          // create temporary canvas for padding the tile
          const tempCanvas = document.createElement('canvas')
          tempCanvas.width = paddedWidth
          tempCanvas.height = paddedHeight

          const tempCtx = tempCanvas.getContext('2d', { willReadFrequently: true })
          tempCtx.drawImage(this.originalImage, x, y, paddedWidth, paddedHeight, 0, 0, paddedWidth, paddedHeight)

          // extract the tile data and draw it onto the temporary canvas with padding
          const tileImageData = tempCtx.getImageData(0, 0, paddedWidth, paddedHeight)

          // add edge pixels for padding
          const paddedTileCanvas = document.createElement('canvas')
          paddedTileCanvas.width = paddedWidth + 2 * padding
          paddedTileCanvas.height = paddedHeight + 2 * padding
          const paddedTileCtx = paddedTileCanvas.getContext('2d', { willReadFrequently: true })

          paddedTileCtx.putImageData(tileImageData, padding, padding)

          // top padding
          const topPadding = paddedTileCtx.getImageData(padding, padding, paddedWidth, 1)
          for (let p = 0; p < padding; p++) {
            paddedTileCtx.putImageData(topPadding, padding, p)
          }

          // bottom padding
          const bottomPadding = paddedTileCtx.getImageData(padding, paddedHeight + padding - 1, paddedWidth, 1)
          for (let p = 0; p < padding; p++) {
            paddedTileCtx.putImageData(bottomPadding, padding, paddedHeight + padding + p)
          }

          // left padding
          const leftPadding = paddedTileCtx.getImageData(padding, padding, 1, paddedHeight)
          for (let p = 0; p < padding; p++) {
            paddedTileCtx.putImageData(leftPadding, p, padding)
          }

          // right padding
          const rightPadding = paddedTileCtx.getImageData(paddedWidth + padding - 1, padding, 1, paddedHeight)
          for (let p = 0; p < padding; p++) {
            paddedTileCtx.putImageData(rightPadding, paddedWidth + padding + p, padding)
          }

          // convert tile into a tensor
          const tensor = await this.loadImageToTensor(paddedTileCanvas)

          // push tile tensor to list of tiles
          tiles.push({ tensor, x, y, width: tileWidth, height: tileHeight, paddedWidth, paddedHeight, paddedTileCanvas })
        }
      }

      const outputTiles = []

      // perform inference on tile tensors and push them into list of output tiles
      for (const tile of tiles) {
        let result = await this.processTile(tile)
        outputTiles.push(result)
        result = null
      }

      // pass list of output tiles to function for converting them into RGBA canvases and splicing the tiles together
      await this.renderOutput(outputTiles, upscaleFactor)

      // hide outputCanvas placeholder text
      this.outputImageDrawn = true
    },

    async loadImageToTensor(paddedTileCanvas) {
      const width = paddedTileCanvas.width
      const height = paddedTileCanvas.height
      const ctx = paddedTileCanvas.getContext('2d', { willReadFrequently: true })
      const imageData = ctx.getImageData(0, 0, width, height)
      const data = imageData.data

      // convert to float32/float16 array depending on model used and normalize to [0, 1]
      const TypedArray = useFp16 ? Float16Array : Float32Array
      const floatData = new TypedArray(width * height * 3)

      for (let i = 0; i < width * height; i++) {
        floatData[i * 3] = data[i * 4] / 255.0         // R
        floatData[i * 3 + 1] = data[i * 4 + 1] / 255.0 // G
        floatData[i * 3 + 2] = data[i * 4 + 2] / 255.0 // B
      }

      // reshape to (1, C, H, W) [Batch, Channel, Height, Width]
      const transposedData = new TypedArray(3 * height * width)
      for (let c = 0; c < 3; c++) {
        for (let h = 0; h < height; h++) {
          for (let w = 0; w < width; w++) {
            transposedData[c * height * width + h * width + w] = floatData[h * width * 3 + w * 3 + c]
          }
        }
      }

      const tensorData = new TypedArray(transposedData.length)
      for (let i = 0; i < transposedData.length; i++) {
        tensorData[i] = transposedData[i]
      }

      const tensor = new ort.Tensor(useFp16 ? 'float16' : 'float32', tensorData, [1, 3, height, width])

      return tensor
    },

    async processTile(tile) {
      // expected tensor names to and from model are 'input' and 'output' respectively
      const feeds = { 'input': tile.tensor }
      let results = await session.run(feeds)
      const outputTensor = results['output']
      results = null

      return { outputTensor, x: tile.x, y: tile.y, width: tile.width, height: tile.height, paddedWidth: tile.paddedWidth, paddedHeight: tile.paddedHeight, paddedTileCanvas: tile.paddedTileCanvas }
    },

    async renderOutput(outputTiles, upscaleFactor) {
      // get outputCanvas context
      const ctx = this.$refs.outputCanvas.getContext('2d')

      // initialize offscreen canvas
      const offscreenCanvas = document.createElement('canvas')
      offscreenCanvas.width = this.originalImage.width * upscaleFactor
      offscreenCanvas.height = this.originalImage.height * upscaleFactor
      const offscreenCtx = offscreenCanvas.getContext('2d', { willReadFrequently: true })

      offscreenCtx.clearRect(0, 0, offscreenCanvas.width, offscreenCanvas.height)
      ctx.clearRect(0, 0, this.$refs.outputCanvas.width, this.$refs.outputCanvas.height)

      // convert tensors of each tile to RGBA canvas tile
      for (const tile of outputTiles) {
        const dataPromise = tile.outputTensor.getData({ releaseData: true })
        const outputData = await dataPromise

        const outputPaddedWidth = tile.paddedTileCanvas.width
        const outputPaddedHeight = tile.paddedTileCanvas.height
        const tileX = tile.x
        const tileY = tile.y

        const paddedTileCanvas = offscreenCtx.createImageData(outputPaddedWidth, outputPaddedHeight)
        const tileData = paddedTileCanvas.data

        // convert tile tensor data to RGBA tile data
        for (let y = 0; y < outputPaddedHeight; y++) {
          for (let x = 0; x < outputPaddedWidth; x++) {
            const index = (y * outputPaddedWidth + x) * 4
            tileData[index] = Math.round(outputData[y * outputPaddedWidth + x] * 255)
            tileData[index + 1] = Math.round(outputData[outputPaddedHeight * outputPaddedWidth + y * outputPaddedWidth + x] * 255)
            tileData[index + 2] = Math.round(outputData[2 * outputPaddedHeight * outputPaddedWidth + y * outputPaddedWidth + x] * 255)
            tileData[index + 3] = 255
          }
        }

        const tempCanvas = document.createElement('canvas')
        tempCanvas.width = outputPaddedWidth
        tempCanvas.height = outputPaddedHeight
        const tempCtx = tempCanvas.getContext('2d')
        tempCtx.putImageData(paddedTileCanvas, 0, 0)

        let adjustedTileX = tileX;
        let adjustedTileY = tileY;

        // draw tile on offscreen canvas context
        offscreenCtx.drawImage(
          tempCanvas,
          padding,
          padding,
          paddedTileCanvas.width - 2 * padding,
          paddedTileCanvas.height - 2 * padding,
          adjustedTileX,
          adjustedTileY,
          paddedTileCanvas.width - 2 * padding,
          paddedTileCanvas.height - 2 * padding
        );
      }

      this.processingEndTime = performance.now()
      this.totalProcessingTime = (this.processingEndTime - this.processingStartTime) / 1000

      document.getElementById("statusText").textContent = `Processed image in ${this.totalProcessingTime.toFixed(2)} seconds.`

      // create bitmap image from offscreen canvas and send it to drawOutputImage() for scaled visualization
      var imageBitmap = await createImageBitmap(offscreenCanvas)
      this.drawOutputImage(imageBitmap)

      // bind the offscreenCanvas to Download button
      this.attachDownloadButton(offscreenCanvas)
    },

    drawOutputImage(image) {
      if (!image) return

      const canvas = this.$refs.outputCanvas
      const ctx = canvas.getContext("2d")

      const displayWidth = canvas.clientWidth
      const displayHeight = canvas.clientHeight
      canvas.width = displayWidth * canvasScalingFactor
      canvas.height = displayHeight * canvasScalingFactor

      // scale outputCanvas context for high-resolution drawing
      ctx.scale(canvasScalingFactor, canvasScalingFactor)

      // clear previous outputCanvas contents
      ctx.clearRect(0, 0, canvas.width, canvas.height)

      // preserve aspect ratio inside outputCanvas
      const imgAspect = image.width / image.height
      const canvasAspect = displayWidth / displayHeight

      let drawWidth, drawHeight, offsetX, offsetY

      if (imgAspect > canvasAspect) {
        drawWidth = displayWidth
        drawHeight = displayWidth / imgAspect
        offsetX = 0
        offsetY = (displayHeight - drawHeight) / 2
      } else {
        drawHeight = displayHeight
        drawWidth = displayHeight * imgAspect
        offsetX = (displayWidth - drawWidth) / 2
        offsetY = 0
      }

      // use higher quality scaling
      ctx.imageSmoothingEnabled = true
      ctx.imageSmoothingQuality = "high"
      ctx.drawImage(image, offsetX, offsetY, drawWidth, drawHeight)

      // hide outputCanvas placeholder text
      this.outputImageDrawn = true
    },

    attachDownloadButton(canvas) {
      const button = document.getElementById("downloadButton")

      // remove any existing event listeners before adding a new one
      button.replaceWith(button.cloneNode(true))
      const newButton = document.getElementById("downloadButton")

      // re-attach a new event listener and disable button
      newButton.addEventListener("click", () => this.downloadOutputCanvasImage(canvas))
      newButton.disabled = false
    },

    downloadOutputCanvasImage(canvas) {
      canvas.toBlob((blob) => {
        if (blob) {
          const link = document.createElement("a")
          link.href = URL.createObjectURL(blob)
          link.download = "output.png"
          link.click()
          URL.revokeObjectURL(link.href)
        }
      }, "image/png")
    },

    clearCanvases() {
      // clear inputCanvas
      const inputCanvas = this.$refs.inputCanvas
      const inputCtx = inputCanvas.getContext("2d")
      inputCtx.clearRect(0, 0, inputCanvas.width, inputCanvas.height)

      // clear outputCanvas
      const outputCanvas = this.$refs.outputCanvas
      const outputCtx = outputCanvas.getContext("2d")
      outputCtx.clearRect(0, 0, outputCanvas.width, outputCanvas.height)

      // set status text to original value
      document.getElementById("statusText").textContent = "Select EP, precision and tiling."

      // reset image state variables
      this.originalImage = null
      this.imageUploaded = false
      this.outputImageDrawn = false

      // disable downloadButton and its reference to the previous image
      const button = document.getElementById("downloadButton")
      button.disabled = true
      button.replaceWith(button.cloneNode(true))

      session = null
      console.clear()
    }
  }
}
</script>

<style scoped>
canvas {
  cursor: default;
  background-color: #f8f9faf8;
}

.canvas-container {
  position: relative;
  display: flex;
  align-items: center;
  justify-content: center;
}

.canvas-container:first-child canvas {
  cursor: pointer;
}

.canvas-placeholder {
  position: absolute;
  top: 50%;
  left: 50%;
  transform: translate(-50%, -50%);
  color: gray;
  font-size: 16px;
  text-align: center;
  pointer-events: none;
}

button:disabled {
  opacity: 0.5;
  cursor: not-allowed;
}
</style>