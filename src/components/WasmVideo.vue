<template>
  <div id="wasm-video">
    <el-container>
      <el-header>
        <h2>
          帧率：<span class="fps-num">{{ fpsRef }}</span> FPS
        </h2>
        <el-radio-group v-model="radio">
          <el-radio :label="1">不开启卷积</el-radio>
          <el-radio :label="2">使用JavaScript</el-radio>
          <el-radio :label="3">使用WebAssembly</el-radio>
        </el-radio-group>
      </el-header>
      <el-main>
        <canvas id="cvs" ref="cvs"></canvas>
        <video
          src="../assets/video.mp4"
          class="vdo"
          muted="true"
          loop="true"
          autoplay="true"
          type="video/mp4"
          style="display: none"
          ref="vdo"
        ></video>
      </el-main>
    </el-container>
  </div>
</template>

<script lang="ts" setup>
import { onMounted, ref } from "vue";

/** DOM元素实例 */
const cvs = ref<HTMLCanvasElement>();
const vdo = ref<HTMLVideoElement>();

/** 无需响应式 */
let clientX: number, clientY: number;
let consumeRecords: number[] = [];
let cppConvFilter: any, cppGetkernelPtr, cppGetDataPtr, memory;
let dataOffset: number, kernOffset;
let Uint8View: Uint8Array, Int8View: Int8Array;
const kernel = flipKernel([
  [-1, -1, 1],
  [-1, 14, -1],
  [1, -1, -1],
]);

/** 页面响应式数据 */
const radio = ref(1);
const fpsRef = ref("NaN");

/** 生命周期 */
onMounted(async () => {
  let { instance } = await WebAssembly.instantiateStreaming(
    fetch("http://121.199.160.202/wasm/dip.wasm")
  );
  cppConvFilter = instance.exports.cppConvFilter as any;
  cppGetkernelPtr = instance.exports.cppGetkernelPtr as any;
  cppGetDataPtr = instance.exports.cppGetDataPtr as any;
  memory = instance.exports.memory as any;

  dataOffset = cppGetDataPtr();
  kernOffset = cppGetkernelPtr();

  const flatKernel = kernel.flat();
  Uint8View = new Uint8Array(memory.buffer);
  Int8View = new Int8Array(memory.buffer);
  Int8View.set(flatKernel, kernOffset);

  vdo.value!.play();
  vdo.value!.addEventListener("loadeddata", () => {
    cvs.value!.setAttribute("height", vdo.value!.videoHeight.toString());
    cvs.value!.setAttribute("width", vdo.value!.videoWidth.toString());

    clientX = cvs.value!.clientWidth;
    clientY = cvs.value!.clientHeight;

    draw();
  });
});

function flipKernel(kernel: number[][]) {
  const h = kernel.length;
  const half = Math.floor(h / 2);
  for (let i = 0; i < half; ++i) {
    for (let j = 0; j < h; ++j) {
      let _t = kernel[i][j];
      kernel[i][j] = kernel[h - i - 1][h - j - 1];
      kernel[h - i - 1][h - j - 1] = _t;
    }
  }
  if (h & 1) {
    for (let j = 0; j < half; ++j) {
      let _t = kernel[half][j];
      kernel[half][j] = kernel[half][h - j - 1];
      kernel[half][h - j - 1] = _t;
    }
  }
  return kernel;
}

function jsConvFilter(
  data: Uint8ClampedArray,
  width: number,
  height: number,
  kernel: number[][]
) {
  const divisor = 4;
  const h = kernel.length,
    w = h;
  const half = Math.floor(h / 2);

  // picture iteration.
  for (let y = half; y < height - half; ++y) {
    for (let x = half; x < width - half; ++x) {
      const px = (y * width + x) * 4; // pixel index.
      let r = 0,
        g = 0,
        b = 0;
      // core iteration.
      for (let cy = 0; cy < h; ++cy) {
        for (let cx = 0; cx < w; ++cx) {
          // dealing edge case.
          const cpx = ((y + (cy - half)) * width + (x + (cx - half))) * 4;

          r += data[cpx + 0] * kernel[cy][cx];
          g += data[cpx + 1] * kernel[cy][cx];
          b += data[cpx + 2] * kernel[cy][cx];
        }
      }
      data[px + 0] =
        r / divisor > 255 ? 255 : r / divisor < 0 ? 0 : r / divisor;
      data[px + 1] =
        g / divisor > 255 ? 255 : g / divisor < 0 ? 0 : g / divisor;
      data[px + 2] =
        b / divisor > 255 ? 255 : b / divisor < 0 ? 0 : b / divisor;
    }
  }
  return data;
}

function filterJS(pixelData: Uint8ClampedArray, width: number, height: number) {
  return jsConvFilter(pixelData, width, height, kernel);
}

function filterWasm(
  pixelData: Uint8ClampedArray,
  width: number,
  height: number
) {
  const arLen = pixelData.length;

  Uint8View.set(pixelData, dataOffset);

  // core.
  cppConvFilter(width, height, 4);

  // retrieve data.
  return Uint8View.subarray(dataOffset, dataOffset + arLen);
}

function calcFPS(arr: number[]) {
  const LRU_MAX_COUNT = 20;
  if (arr.length > LRU_MAX_COUNT) {
    arr.shift();
  } else {
    return "NaN";
  }
  let averageTime = arr.reduce((pre, cur) => pre + cur, 0) / LRU_MAX_COUNT;
  return (1000 / averageTime).toFixed(2);
}

function draw() {
  const timeStart = performance.now();
  const context2D = cvs.value?.getContext("2d");
  context2D?.drawImage(vdo.value as any, 0, 0);

  const pixels = context2D!.getImageData(
    0,
    0,
    vdo.value!.videoWidth,
    vdo.value!.videoHeight
  );

  switch (radio.value) {
    case 2: {
      pixels?.data.set(filterJS(pixels.data, clientX, clientY));
      break;
    }
    case 3: {
      pixels?.data.set(filterWasm(pixels.data, clientX, clientY));
      break;
    }
  }

  context2D!.putImageData(pixels, 0, 0);

  const timeUsed = performance.now() - timeStart;
  consumeRecords.push(timeUsed);
  fpsRef.value = calcFPS(consumeRecords);
  requestAnimationFrame(draw);
}
</script>

<style scoped>
#wasm-video {
  display: flex;
  flex-direction: column;
  height: 100%;
  width: 100%;
}

#cvs {
  width: calc(100vw - 40px);
  height: calc(100vh - 100px);
}

.el-main {
  flex: 1;
  display: flex;
  justify-content: center;
  align-items: center;
}

.el-header {
  display: flex;
  justify-content: space-between;
}

.el-header h2 {
  display: flex;
  align-items: center;
}
</style>
