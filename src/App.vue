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
  </div>
</template>