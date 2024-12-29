<template>
  <div class="app">
    <div className="header">
      <h1>📷 YOLOv5 Live Detection App</h1>
      <p>
        YOLOv5 live detection application on browser powered by <code>tensorflow.js</code>
      </p>
      <p>
        Serving : <code className="code">yolov5n</code>
      </p>
    </div>
    <AnimalIdentification />
  </div>
</template>

<script setup lang="ts">
import AnimalIdentification from './components/animalIdentification/index.vue'
</script>


<style scoped>
.app {
  margin-top: 60px;
  display: flex;
  flex-direction: column;
  align-items: center;
  justify-content: center;
}

.header {
  text-align: center;
}

.header p {
  margin: 5px 0;
}

.code {
  padding: 5px;
  color: greenyellow;
  background-color: black;
  border-radius: 5px;
}

.content>img {
  display: none;
  width: 100%;
  max-width: 720px;
  max-height: 500px;
  border-radius: 10px;
}
</style>
