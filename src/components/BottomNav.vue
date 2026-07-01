<template>
  <div class="bottom-nav">
    <nav class="nav-bar">
      <button v-for="item in navItems" :key="item.id" class="nav-button" :class="{ active: activeNav === item.id }"
        @click="activeNav = item.id">
        <span class="nav-icon">{{ item.icon }}</span>
        <span class="nav-label">{{ item.label }}</span>
        <span v-if="activeNav === item.id" class="nav-indicator"></span>
      </button>
    </nav>
  </div>
</template>

<script setup>
import { ref } from 'vue'

const activeNav = ref(0)

const navItems = ref([
  { id: 0, icon: '📊', label: '实况' },
  { id: 1, icon: '📈', label: '预报' },
  { id: 2, icon: '🚨', label: '预警' },
  { id: 3, icon: '⚙️', label: '预案' },
  { id: 4, icon: '🔧', label: '设置' }
])
</script>

<style scoped lang="scss">
.bottom-nav {
  display: flex;
  justify-content: center;
  width: 100%;
  padding: 16px 0;

  .nav-bar {
    display: flex;
    gap: 6px;
    background: rgba(9, 22, 40, 0.8);
    border: 1px solid #00e0ee;
    border-radius: 0;
    padding: 8px 4px;
    backdrop-filter: blur(10px);
    box-shadow:
      0 0 10px #00e0ee,
      0 0 20px rgba(0, 224, 238, 0.3),
      0 0 30px rgba(0, 224, 238, 0.15),
      inset 0 0 15px rgba(0, 224, 238, 0.08);
    animation: nav-bar-glow 3s ease-in-out infinite;
  }

  .nav-button {
    position: relative;
    display: flex;
    align-items: center;
    justify-content: center;
    gap: 10px;


    padding: 0; // ✅ 去掉内边距

    background: rgba(8, 19, 34, 0.9);
    border: 1px solid #00e0ee;
    border-radius: 6px;

    color: #00e8f2;

    font-weight: 600;
    letter-spacing: 1px;
    width: 140px;
    height: 42px;
    font-size: 16px;
    cursor: pointer;
    transition: all 0.3s ease;

    .nav-icon {
      font-size: 22px;
    }

    .nav-label {
      font-size: 18px;
      line-height: 1;
    }

    .nav-label {
      display: inline-block;
    }

    &:hover:not(.active) {
      border-color: #00e8f2;
      color: #00e8f2;
      background: rgba(8, 19, 34, 0.95);
      box-shadow: 0 0 15px rgba(0, 232, 242, 0.1);
    }

    &.active {
      border-color: #00e8f2;
      color: #ffffff;
      background: linear-gradient(135deg, #007a88, #00b9c9);
      box-shadow: 
        0 0 20px rgba(0, 232, 242, 0.5),
        0 0 30px rgba(0, 232, 242, 0.3),
        inset 0 0 15px rgba(255, 255, 255, 0.1);
      animation: button-active-glow 2s ease-in-out infinite;

      .nav-indicator {
        position: absolute;
        bottom: -12px;
        left: 50%;
        transform: translateX(-50%);
        width: 8px;
        height: 8px;
        background: #00e8f2;
        border-radius: 50%;
        box-shadow: 0 0 16px #00e8f2;
        animation: pulse-dot 1.5s ease-in-out infinite;
      }
    }
  }
}

@keyframes pulse-dot {

  0%,
  100% {
    transform: translateX(-50%) scale(1);
    opacity: 1;
    box-shadow: 0 0 16px #00e8f2;
  }

  50% {
    transform: translateX(-50%) scale(1.2);
    opacity: 0.8;
    box-shadow: 0 0 24px #00e8f2;
  }
}

@keyframes nav-bar-glow {

  0%,
  100% {
    box-shadow:
      0 0 10px #00e0ee,
      0 0 20px rgba(0, 224, 238, 0.3),
      0 0 30px rgba(0, 224, 238, 0.15),
      inset 0 0 15px rgba(0, 224, 238, 0.08);
  }

  50% {
    box-shadow:
      0 0 15px #00e0ee,
      0 0 30px rgba(0, 224, 238, 0.5),
      0 0 40px rgba(0, 224, 238, 0.25),
      inset 0 0 20px rgba(0, 224, 238, 0.12);
  }
}

@keyframes button-active-glow {

  0%,
  100% {
    box-shadow:
      0 0 20px rgba(0, 232, 242, 0.5),
      0 0 30px rgba(0, 232, 242, 0.3),
      inset 0 0 15px rgba(255, 255, 255, 0.1);
  }

  50% {
    box-shadow:
      0 0 25px rgba(0, 232, 242, 0.7),
      0 0 40px rgba(0, 232, 242, 0.4),
      inset 0 0 20px rgba(255, 255, 255, 0.15);
  }
}

@media (max-width: 1600px) {
  .bottom-nav {
    .nav-bar {
      gap: 4px;
      padding: 8px 3px;
    }

    .nav-button {
      padding: 8px 14px;
      font-size: 11px;

      .nav-icon {
        font-size: 14px;
      }
    }
  }
}
</style>