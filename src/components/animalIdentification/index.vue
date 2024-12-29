<template>
  <div class="animal-identification">

    <div v-if="loading.loading" class="loading-text">
      加载模型中... {{ (loading.progress * 100).toFixed(2) }}%
    </div>
    <template v-else>
      <div class="animal-identification-container">
        <video autoPlay muted ref="videoRef" :onPlay="onPlay" />

        <video autoPlay muted ref="hlsVideoRef" :onPlay="onHlsPlay" />

        <canvas :width="model.inputShape[1]" :height="model.inputShape[2]" ref="canvasRef" />
      </div>

      <div style="display: flex; flex-direction: column; align-items: center; justify-content: center;">
        <div>
          <input ref="inputVideoRef" type="file" accept="video/*" style="display: none" :onChange="changeVideo" />
          <button @click="selectVideo">
            {{ streaming === "video" ? "Close" : "Open" }} Video
          </button>

          <button @click="setVideo(0)">
            Example Video 1
          </button>

          <button @click="setVideo(1)">
            Example Video 2
          </button>

        </div>

        <div>
          <input ref="inputM3u8Ref" type="input" v-model="m3u8" />
          <button @click="defaultM3u8" v-if="!m3u8">
            default HLS
          </button>
          <button v-else @click="selectM3u8">
            {{ streaming === "hls" ? "Close" : "Play" }} HLS
          </button>
        </div>
      </div>
    </template>
  </div>
</template>

<script name="animal-identification" setup lang="ts">
import Hls from 'hls.js';
import * as tf from "@tensorflow/tfjs";
import { ref, reactive, toRaw, onMounted, nextTick } from 'vue'
import "@tensorflow/tfjs-backend-webgl"; // set backend to webgl
import { detectVideo } from "./utils/detect";

// https://sf1-cdn-tos.huoshanstatic.com/obj/media-fe/xgplayer_doc_video/hls/xgplayer-demo.m3u8
const m3u8 = ref('https://linear-144.frequency.stream/dist/localnow/144/hls/hd/playlist.m3u8')

const loading = reactive({
  loading: false,
  progress: 0
})

const model = ref<{
  net: tf.GraphModel | null,
  inputShape: number[]
}>({
  net: null,
  inputShape: [1, 0, 0, 3],
})

const hls = ref(null)

// references
const cameraRef = ref(null);
const videoRef = ref(null);
const canvasRef = ref(null);

const hlsVideoRef = ref(null);

const inputVideoRef = ref(null);

// model configs
const modelName = "yolov5n";
const classThreshold = 0.2;

const streaming = ref(null)

const onPlay = (e) => {
  detectVideo(videoRef.value, toRaw(model.value), classThreshold, canvasRef.value)
}

const onHlsPlay = (e) => {
  detectVideo(hlsVideoRef.value, toRaw(model.value), classThreshold, canvasRef.value)
}

const changeVideo = (e) => {
  const url = URL.createObjectURL(e.target.files[0]); // create blob url
  videoRef.value.src = url; // set video source
  videoRef.value.addEventListener("ended", () => closeVideo()); // add ended video listener
  videoRef.value.style.display = "block"; // show video
  streaming.value = "video"; // set streaming to video 
}

const setVideo = (index: number) => {
  closeVideo()
  closeHlsVideo()
  if (index === 0) {
    videoRef.value.src = `${window.location.href}/car.mp4`
  } else {
    videoRef.value.src = `${window.location.href}/qq.mp4`
  }
  videoRef.value.addEventListener("ended", () => closeVideo()); // add ended video listener
  videoRef.value.style.display = "block"; // show video
  streaming.value = "video"; // set streaming to video 
}

const setHlsVideo = async () => {
  if (!m3u8.value) return
  // 检查浏览器是否支持HLS
  if (Hls.isSupported()) {
    await initHls()

    hls.value.loadSource(m3u8.value);
    hls.value.attachMedia(hlsVideoRef.value);

    hlsVideoRef.value.addEventListener("ended", () => closeVideo()); // add ended video listener
    hlsVideoRef.value.style.display = "block"; // show video
    streaming.value = "hls"; // set streaming to video 
  }
}

const selectVideo = () => {
  if (streaming.value === "video") {
    closeVideo()
  } else {
    closeHlsVideo()
    inputVideoRef.value.click()
  }
}

const selectM3u8 = () => {
  if (streaming.value === "hls") {
    closeHlsVideo()
  } else {
    closeVideo()
    setHlsVideo()
  }
}

const defaultM3u8 = () => {
  m3u8.value = 'https://linear-144.frequency.stream/dist/localnow/144/hls/hd/playlist.m3u8'

  selectM3u8()
}

// closing video streaming
const closeVideo = () => {
  const url = videoRef.value.src;
  videoRef.value.src = ""; // restore video source
  URL.revokeObjectURL(url); // revoke url

  // set streaming to null
  streaming.value = null
  inputVideoRef.value.value = ""; // reset input video
  videoRef.value.style.display = "none"; // hide video
};

const closeHlsVideo = () => {
  const url = hlsVideoRef.value.src;
  hlsVideoRef.value.src = ""; // restore video source
  URL.revokeObjectURL(url); // revoke url

  if (hls.value) {
    hls.value?.stopLoad()
    hls.value?.destroy()
    hls.value = null
  }
  streaming.value = null
  m3u8.value = ''
  hlsVideoRef.value.style.display = "none"; // hide video
}

onMounted(() => {
  // 确保先设置并初始化WebGL后端
  tf.setBackend('webgl').then(() => {
    tf.ready().then(async () => {
      const yolov5 = await tf.loadGraphModel(
        `${window.location.href}/${modelName}_web_model/model.json`,
        {
          onProgress: (fractions) => {
            loading.loading = true
            loading.progress = fractions
          },
        }
      );

      // warming up model
      const modelShape = yolov5.inputs[0]?.shape;
      if (!modelShape) {
        throw new Error('Model input shape is undefined');
      }
      const dummyInput = tf.ones(modelShape);
      const warmupResult = await yolov5.executeAsync(dummyInput);
      tf.dispose(warmupResult); // cleanup memory
      tf.dispose(dummyInput); // cleanup memory

      loading.loading = false
      loading.progress = 1

      model.value.net = yolov5
      model.value.inputShape = modelShape

      initHls()

      nextTick(() => {
        // setVideo(0)
        setHlsVideo()
      })
    })
  });
})


const initHls = () => {
  if (Hls.isSupported()) {
    hls.value = new Hls({
      debug: false, // 添加调试信息
      xhrSetup: function (xhr) {
        xhr.withCredentials = false
      },
      // 添加重试配置
      maxLoadingDelay: 4,
      maxMaxBufferLength: 60,
      levelLoadingMaxRetry: 4,
      manifestLoadingMaxRetry: 4,
      fragLoadingMaxRetry: 4,
    });

    // 绑定错误处理
    hls.value.on(Hls.Events.ERROR, function (event, data) {
      console.warn('HLS Error:', data)
      if (data.fatal) {
        switch (data.type) {
          case Hls.ErrorTypes.NETWORK_ERROR:
            console.error('网络错误，尝试恢复...', data)
            hls.startLoad()
            break
          case Hls.ErrorTypes.MEDIA_ERROR:
            console.error('媒体错误，尝试恢复...', data)
            hls.recoverMediaError()
            break
          default:
            console.error('无法恢复的错误，重新初始化播放器...', data)
            initHlsPlayer()
            break
        }
      }
    })

    hls.value.on(Hls.Events.MANIFEST_LOADED, () => {
      console.log('清单加载成功')
    })

    hls.value.on(Hls.Events.MANIFEST_PARSED, () => {
      console.log('清单文件解析完成，开始播放');
      hlsVideoRef.value.play().catch(e => console.error('播放失败:', e));
    });
  }
}

const initHlsPlayer = () => {
  if (hls.value) { hls.value.destroy() }
  hls.value = new Hls()
  hls.value.loadSource('https://linear-144.frequency.stream/dist/localnow/144/hls/hd/playlist.m3u8')
  hls.value.attachMedia(hlsVideoRef.value)
}
</script>

<style scoped>
.animal-identification {
  display: flex;
  flex-direction: column;
  align-items: center;
  justify-content: center;
  position: relative;
}

.animal-identification-container {
  position: relative;
  margin: 12px;
}

.animal-identification-container video {
  display: none;
  width: 100%;
  max-width: 1000px;
  max-height: 600px;
  border-radius: 10px;
}

.animal-identification-container canvas {
  position: absolute;
  top: 0;
  left: 0;
  width: 100%;
  height: 100%;
}

.loading-text {
  height: 600px;
  display: flex;
  align-items: center;
  justify-content: center;
  width: 1000px;
  background-color: #f5f5f50a;
  border-radius: 10px;
  margin: 12px 0;
}

button {
  text-decoration: none;
  color: white;
  background-color: black;
  border: 2px solid black;
  margin: 12px 5px;
  padding: 5px;
  border-radius: 5px;
  cursor: pointer;
}

button:hover {
  color: black;
  background-color: white;
  border: 2px solid black;
}

input {
  width: 600px;
  color: white;
  background-color: black;
  border: 2px solid black;
  margin: 12px 5px;
  padding: 5px;
  border-radius: 5px;
  cursor: pointer;
}
</style>
