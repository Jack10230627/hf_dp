<template>
  <div class="container">
    <!-- 项目2 iframe -->
    <div :class="['iframe-wrapper', { active: isTransitioned }]">
      <iframe
        ref="mapIframe"
        src="http://localhost:5173/"
        frameborder="0"
        class="full-iframe"
      ></iframe>
    </div>

    <!-- 项目1 3D -->
    <div :class="['three-layer', { hidden: isTransitioned }]">
      <AnhuiMap ref="threeRef" @zoom-in-limit="switchTo2D" />
    </div>

    <!-- 切换遮罩层 -->
    <div
      v-if="transitionMaskVisible"
      class="transition-mask"
    >
      <div class="transition-core"></div>
      <div class="transition-text">正在进入二维地图</div>
    </div>
  </div>
</template>

<script setup>
import { ref, onMounted, onUnmounted } from "vue";
import AnhuiMap from "./AnhuiMap.vue";

const isTransitioned = ref(false);
const isTransitioning = ref(false);
const transitionMaskVisible = ref(false);

const threeRef = ref(null);
const mapIframe = ref(null);

let transitionFallbackTimer = null;

/**
 * 3D -> 2D
 */
const switchTo2D = (data) => {
  if (isTransitioning.value || isTransitioned.value) return;

  isTransitioning.value = true;
  transitionMaskVisible.value = true;

  // 先显示遮罩，让用户看不到 iframe 准备过程
  setTimeout(() => {
    if (mapIframe.value?.contentWindow) {
      mapIframe.value.contentWindow.postMessage(
        {
          type: "SYNC_MAP",
          lonLat: data.lonLat,
          zoom: data.zoom || 14,
        },
        "http://localhost:5174"
      );
    }

    // 兜底：如果 iframe 没有回消息，最多等 1.2 秒也切过去
    clearTimeout(transitionFallbackTimer);
    transitionFallbackTimer = setTimeout(() => {
      finishSwitchTo2D();
    }, 1200);
  }, 120);
};

/**
 * iframe 地图准备好后，真正显示 2D
 */
const finishSwitchTo2D = () => {
  clearTimeout(transitionFallbackTimer);

  isTransitioned.value = true;

  setTimeout(() => {
    transitionMaskVisible.value = false;
    isTransitioning.value = false;
  }, 450);
};

/**
 * 2D -> 3D
 */
const switchTo3D = (lonLat) => {
  if (isTransitioning.value || !isTransitioned.value) return;

  isTransitioning.value = true;
  transitionMaskVisible.value = true;

  setTimeout(() => {
    if (threeRef.value?.syncViewFrom2D) {
      threeRef.value.syncViewFrom2D(lonLat);
    }

    isTransitioned.value = false;

    setTimeout(() => {
      transitionMaskVisible.value = false;
      isTransitioning.value = false;
    }, 450);
  }, 120);
};

/**
 * 接收 iframe 消息
 */
const handleMessage = (event) => {
  if (event.origin !== "http://localhost:5173") return;

  const data = event.data || {};

  // 项目2通知：地图已经完成定位和渲染
  if (data.type === "MAP_SYNC_DONE") {
    finishSwitchTo2D();
  }

  // 项目2通知：缩小到阈值，需要回到3D
  if (data.type === "BACK_TO_3D") {
    switchTo3D(data.lonLat);
  }
};

onMounted(() => {
  window.addEventListener("message", handleMessage);
});

onUnmounted(() => {
  window.removeEventListener("message", handleMessage);
  clearTimeout(transitionFallbackTimer);
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

/* 3D层 */
.three-layer {
  position: absolute;
  inset: 0;
  z-index: 5;
  opacity: 1;
  transition:
    opacity 0.45s ease,
    filter 0.45s ease;
}

.three-layer.hidden {
  opacity: 0;
  filter: blur(4px);
  pointer-events: none;
}

/* iframe层 */
.iframe-wrapper {
  position: absolute;
  inset: 0;
  z-index: 3;
  opacity: 0;
  pointer-events: none;
  transition:
    opacity 0.45s ease,
    filter 0.45s ease;
  filter: blur(4px);
}

.iframe-wrapper.active {
  z-index: 10;
  opacity: 1;
  pointer-events: auto;
  filter: blur(0);
}

.full-iframe {
  width: 100%;
  height: 100%;
  border: none;
  display: block;
  background: #000;
}

/* 切换遮罩 */
.transition-mask {
  position: absolute;
  inset: 0;
  z-index: 9999;
  pointer-events: none;
  background:
    radial-gradient(circle at center, rgba(0, 255, 255, 0.18), transparent 28%),
    rgba(0, 8, 16, 0.72);
  backdrop-filter: blur(8px);
  animation: maskIn 0.28s ease forwards;
}

.transition-core {
  position: absolute;
  left: 50%;
  top: 50%;
  width: 180px;
  height: 180px;
  transform: translate(-50%, -50%);
  border-radius: 50%;
  border: 1px solid rgba(0, 255, 255, 0.8);
  box-shadow:
    0 0 30px rgba(0, 255, 255, 0.5),
    inset 0 0 30px rgba(0, 255, 255, 0.25);
  animation: pulse 1s ease-in-out infinite;
}

.transition-text {
  position: absolute;
  left: 50%;
  top: calc(50% + 120px);
  transform: translateX(-50%);
  color: #8ffcff;
  font-size: 16px;
  letter-spacing: 4px;
  text-shadow: 0 0 12px rgba(0, 255, 255, 0.8);
}

@keyframes maskIn {
  from {
    opacity: 0;
  }
  to {
    opacity: 1;
  }
}

@keyframes pulse {
  0% {
    transform: translate(-50%, -50%) scale(0.86);
    opacity: 0.55;
  }
  50% {
    transform: translate(-50%, -50%) scale(1.08);
    opacity: 1;
  }
  100% {
    transform: translate(-50%, -50%) scale(0.86);
    opacity: 0.55;
  }
}
</style>