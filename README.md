# mlop-sr-deployment
1x Image Restoration inference using [ASID](https://github.com/saturnian77/ASID), powered by Vue.js, Bootstrap and ONNX Runtime Web, targeting real-life photo degradation removal (blur, noise, compression). Currently supports WebGPU and WASM EPs with both FP32 and FP16 precision (requires Chrome 135 or newer). Uses custom tiling method to prevent ASID artifacts on image borders. It is available [here](https://matejzverka.github.io/mlop-sr-deployment/).
# Warning
- Inference with 1x1 tiling not recommended for large images.
- Model is lightweight but inference can cause very high VRAM usage with large tiles.
- Start with default 4x4, change depending on RAM/VRAM usage and inference time.
- Your browser might crash if tile size is too large.
- Despite using relevant Cross-origin isolation headers, WASM multi-threading doesn't work on deployed app (works in local project).
- Model is experimental and might not remove complex or extreme degradations.