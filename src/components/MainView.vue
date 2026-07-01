<template>
  <div class="container">
    <!-- 项目2 iframe -->
    <div :class="['iframe-wrapper', { active: isTransitioned, 'no-anim': !enableTransitionAnimation }]">
      <iframe
        ref="mapIframe"
        src="http://localhost:5174/"
        frameborder="0"
        class="full-iframe"
      ></iframe>
    </div>

    <!-- 项目1 3D -->
    <div
      :class="[
        'three-layer',
        {
          hidden: isTransitioned,
          zooming: isTransitioning && enableTransitionAnimation,
          'no-anim': !enableTransitionAnimation,
        },
      ]"
    >
      <AnhuiMap ref="threeRef" @zoom-in-limit="switchTo2D" />
    </div>

    <!-- 颜色桥接层：仅在启用动画时显示 -->
    <div
      v-if="enableTransitionAnimation"
      class="color-bridge"
      :class="{ active: isTransitioning || isTransitioned }"
    ></div>

    <!-- 切换遮罩层：仅在启用动画时显示 -->
    <div
      v-if="transitionMaskVisible && enableTransitionAnimation"
      class="transition-mask"
      :class="{ fadeout: maskFadingOut }"
    >
      <div class="transition-core"></div>
      <div class="transition-ring"></div>
      <div class="transition-text">{{ transitionText }}</div>
    </div>
  </div>
</template>

<script setup>
import { ref, onMounted, onUnmounted } from "vue";
import AnhuiMap from "./AnhuiMap.vue";

// ==============================================
// ✨ 过渡动画总开关
//   true  - 使用完整过渡动画（遮罩、缩放、颜色桥接、发消息延迟）
//   false - 关闭动画，瞬间切换
// ==============================================
const enableTransitionAnimation = ref(true);

const isTransitioned = ref(false);
const isTransitioning = ref(false);
const transitionMaskVisible = ref(false);
const maskFadingOut = ref(false);

// 过渡提示文字（根据方向动态变化）
const transitionText = ref("正在进入二维地图");

const threeRef = ref(null);
const mapIframe = ref(null);

let transitionFallbackTimer = null;

/**
 * 3D -> 2D
 */
const switchTo2D = (data) => {
  if (isTransitioning.value || isTransitioned.value) return;

  isTransitioning.value = true;
  maskFadingOut.value = false;
  transitionText.value = "正在进入二维地图";

  // ============ 无动画模式：瞬间切换 ============
  if (!enableTransitionAnimation.value) {
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
    // 直接切到 2D，不等 iframe 回消息
    isTransitioned.value = true;
    isTransitioning.value = false;
    return;
  }

  // ============ 有动画模式 ============
  transitionMaskVisible.value = true;

  // 阶段1：给 3D 场景一点时间做"色调过渡+推近"（约 420ms）
  // 阶段2：再通知 iframe 定位
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

    // 兜底：如果 iframe 没有回消息，最多等 1.5 秒也切过去
    clearTimeout(transitionFallbackTimer);
    transitionFallbackTimer = setTimeout(() => {
      finishSwitchTo2D();
    }, 1500);
  }, 420);
};

/**
 * iframe 地图准备好后，真正显示 2D
 */
const finishSwitchTo2D = () => {
  clearTimeout(transitionFallbackTimer);

  // 无动画模式下：MAP_SYNC_DONE 消息回来时已经切换过了，直接返回
  if (!enableTransitionAnimation.value) {
    isTransitioned.value = true;
    isTransitioning.value = false;
    return;
  }

  // 有动画模式：先显示 iframe，再淡出遮罩
  isTransitioned.value = true;

  setTimeout(() => {
    maskFadingOut.value = true;
    setTimeout(() => {
      transitionMaskVisible.value = false;
      isTransitioning.value = false;
      maskFadingOut.value = false;
    }, 500);
  }, 350);
};

/**
 * 2D -> 3D
 */
const switchTo3D = (lonLat) => {
  if (isTransitioning.value || !isTransitioned.value) return;

  isTransitioning.value = true;
  maskFadingOut.value = false;
  transitionText.value = "正在进入三维地图";

  // ============ 无动画模式：瞬间切换 ============
  if (!enableTransitionAnimation.value) {
    if (threeRef.value?.syncViewFrom2D) {
      threeRef.value.syncViewFrom2D(lonLat);
    }
    isTransitioned.value = false;
    isTransitioning.value = false;
    return;
  }

  // ============ 有动画模式 ============
  transitionMaskVisible.value = true;

  setTimeout(() => {
    if (threeRef.value?.syncViewFrom2D) {
      threeRef.value.syncViewFrom2D(lonLat);
    }

    isTransitioned.value = false;

    setTimeout(() => {
      maskFadingOut.value = true;
      setTimeout(() => {
        transitionMaskVisible.value = false;
        isTransitioning.value = false;
        maskFadingOut.value = false;
      }, 500);
    }, 350);
  }, 200);
};

/**
 * 接收 iframe 消息
 */
const handleMessage = (event) => {
  if (event.origin !== "http://localhost:5174") return;

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
  background: #eef3ea;
  overflow: hidden;
}

/* ================= 3D层 ================= */
.three-layer {
  position: absolute;
  inset: 0;
  z-index: 5;
  opacity: 1;
  transform: scale(1);
  transform-origin: center center;
  transition:
    opacity 0.6s ease,
    transform 0.9s cubic-bezier(0.55, 0.05, 0.35, 1),
    filter 0.6s ease;
  will-change: transform, opacity, filter;
}

.three-layer.zooming {
  transform: scale(1.35);
  filter: blur(2px) saturate(1.1) brightness(1.03);
}

.three-layer.hidden {
  opacity: 0;
  transform: scale(1.6);
  filter: blur(6px) saturate(1.2) brightness(1.05);
  pointer-events: none;
}

/* ✨ 无动画模式：禁用所有过渡效果 */
.three-layer.no-anim {
  transition: none !important;
  transform: none !important;
  filter: none !important;
}

.three-layer.no-anim.hidden {
  opacity: 0;
  transform: none !important;
  filter: none !important;
  pointer-events: none;
}

/* ================= iframe层 ================= */
.iframe-wrapper {
  position: absolute;
  inset: 0;
  z-index: 3;
  opacity: 0;
  pointer-events: none;
  transform: scale(1.08);
  transform-origin: center center;
  filter: blur(6px) brightness(1.03);
  transition:
    opacity 0.7s ease,
    transform 0.9s cubic-bezier(0.25, 0.8, 0.3, 1),
    filter 0.7s ease;
  will-change: transform, opacity, filter;
  background: #eef3ea;
}

.iframe-wrapper.active {
  z-index: 10;
  opacity: 1;
  transform: scale(1);
  pointer-events: auto;
  filter: blur(0) brightness(1);
}

/* ✨ 无动画模式：禁用所有过渡效果 */
.iframe-wrapper.no-anim {
  transition: none !important;
  transform: none !important;
  filter: none !important;
}

.iframe-wrapper.no-anim.active {
  opacity: 1;
  transform: none !important;
  filter: none !important;
}

.full-iframe {
  width: 100%;
  height: 100%;
  border: none;
  display: block;
  background: #eef3ea;
}

/* ================= 颜色桥接层 ================= */
.color-bridge {
  position: absolute;
  inset: 0;
  z-index: 8;
  pointer-events: none;
  opacity: 0;
  background:
    radial-gradient(
      ellipse at center,
      rgba(238, 243, 234, 0.0) 0%,
      rgba(238, 243, 234, 0.15) 50%,
      rgba(238, 243, 234, 0.4) 100%
    );
  transition: opacity 0.7s ease;
}

.color-bridge.active {
  opacity: 1;
}

/* ================= 切换遮罩 ================= */
.transition-mask {
  position: absolute;
  inset: 0;
  z-index: 9999;
  pointer-events: none;
  background:
    radial-gradient(
      circle at center,
      rgba(238, 243, 234, 0.1) 0%,
      rgba(220, 230, 215, 0.5) 50%,
      rgba(200, 215, 195, 0.8) 100%
    );
  backdrop-filter: blur(6px) saturate(1.05);
  animation: maskIn 0.35s ease forwards;
}

.transition-mask.fadeout {
  animation: maskOut 0.5s ease forwards;
}

.transition-core {
  position: absolute;
  left: 50%;
  top: 50%;
  width: 160px;
  height: 160px;
  transform: translate(-50%, -50%);
  border-radius: 50%;
  border: 1px solid rgba(45, 106, 90, 0.7);
  box-shadow:
    0 0 40px rgba(120, 180, 150, 0.55),
    inset 0 0 40px rgba(180, 220, 200, 0.35);
  animation: pulse 1.1s ease-in-out infinite;
}

.transition-ring {
  position: absolute;
  left: 50%;
  top: 50%;
  width: 280px;
  height: 280px;
  transform: translate(-50%, -50%);
  border-radius: 50%;
  border: 1px dashed rgba(90, 140, 110, 0.5);
  animation: ringRotate 4s linear infinite;
}

.transition-text {
  position: absolute;
  left: 50%;
  top: calc(50% + 120px);
  transform: translateX(-50%);
  color: #2d5a4a;
  font-size: 16px;
  letter-spacing: 4px;
  text-shadow: 0 0 12px rgba(255, 255, 255, 0.95);
  font-weight: 600;
  white-space: nowrap;
}

@keyframes maskIn {
  from {
    opacity: 0;
  }
  to {
    opacity: 1;
  }
}

@keyframes maskOut {
  from {
    opacity: 1;
  }
  to {
    opacity: 0;
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

@keyframes ringRotate {
  from {
    transform: translate(-50%, -50%) rotate(0deg);
  }
  to {
    transform: translate(-50%, -50%) rotate(360deg);
  }
}
</style>