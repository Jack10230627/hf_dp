<template>
  <div class="container">
    <!-- 项目 2: OpenLayers (Iframe) -->
    <div :class="['iframe-wrapper', { active: isTransitioned }]">
      <iframe
        ref="mapIframe"
        src="http://localhost:5174/"
        frameborder="0"
        class="full-iframe"
      ></iframe>
    </div>

    <!-- 项目 1: Three.js (3D) -->
    <div :class="['three-layer', { 'fade-out': isTransitioned }]">
      <AnhuiMap ref="threeRef" @zoom-in-limit="switchTo2D" />
    </div>
  </div>
</template>

<script setup>
import { ref, onMounted, onUnmounted } from "vue";
import AnhuiMap from "./AnhuiMap.vue";

const isTransitioned = ref(false);
const threeRef = ref(null);
const mapIframe = ref(null);

// 3D 放大到极致 -> 切换到 2D
 const switchTo2D = (data) => {
  if (isTransitioned.value) return;
  isTransitioned.value = true;

  // 延迟发送消息给 iframe，确保 iframe 已经从 opacity: 0 变为可见
  setTimeout(() => {
    if (mapIframe.value?.contentWindow) {
      mapIframe.value.contentWindow.postMessage({
        type: "SYNC_MAP",
        lonLat: data.lonLat,
        zoom: 15 // 稍微大一点，拉开切换点的距离感
      }, "*");
    }
  }, 300);
};
// 接收来自 Iframe 的回退请求
const handleMessage = (event) => {
  if (event.origin !== "http://localhost:5174") return;
  
  if (event.data.type === "BACK_TO_3D") {
    console.log("检测到 2D 地图缩小，回退到 3D...");
    isTransitioningTo3D();
    threeRef.value.syncViewFrom2D(event.data.lonLat);
  }
};

const isTransitioningTo3D = () => {
  isTransitioned.value = false;
};

onMounted(() => {
  window.addEventListener("message", handleMessage);
});

onUnmounted(() => {
  window.removeEventListener("message", handleMessage);
});
</script>

<style scoped>
.container {
  position: relative;
  width: 100vw;
  height: 100vh;
  background: #000;
  overflow: hidden;
}

.three-layer {
  position: absolute;
  inset: 0;
  z-index: 5;
  transition: all 0.8s cubic-bezier(0.4, 0, 0.2, 1);
  opacity: 1;
}

.three-layer.fade-out {
  opacity: 0;
  transform: scale(2.5); /* 俯冲放大感 */
  pointer-events: none;
}

.iframe-wrapper {
  position: absolute;
  inset: 0;
  z-index: 1;
  opacity: 0;
  transition: opacity 1s ease;
  pointer-events: none;
}

.iframe-wrapper.active {
  z-index: 10;
  opacity: 1;
  pointer-events: auto;
}

.full-iframe {
  width: 100%;
  height: 100%;
  border: none;
}
</style>