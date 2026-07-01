<template>
  <canvas ref="canvasRef" class="cyberpunk-bg"></canvas>
</template>

<script>
export default {
  name: 'CyberpunkBackground',
  data() {
    return {
      time: 0,
      animationId: null,
      SCALE: 0.8,
      colors: {
        brightCyan: 'rgba(0, 230, 245, 0.6)',
        darkCyan: 'rgba(0, 124, 139, 0.3)',
        accentCyan: 'rgba(0, 228, 242, 0.4)'
      }
    }
  },
  mounted() {
    this.resizeCanvas()
    window.addEventListener('resize', this.resizeCanvas)
    this.render()
  },
  beforeDestroy() {
    if (this.animationId) cancelAnimationFrame(this.animationId)
    window.removeEventListener('resize', this.resizeCanvas)
  },
  methods: {
    resizeCanvas() {
      const canvas = this.$refs.canvasRef
      if (!canvas) return
      canvas.width = window.innerWidth
      canvas.height = window.innerHeight
    },
    drawConcentricArcs(ctx, cx, cy, w, h) {
      const rBase = Math.min(w, h) * 0.35
      const rotation = this.time * 0.0003

      ctx.save()
      ctx.translate(cx, cy)

      // 1. 最外圈实线
      ctx.strokeStyle = this.colors.accentCyan
      ctx.lineWidth = 2
      ctx.beginPath()
      ctx.arc(0, 0, rBase * 1.2, 0, Math.PI * 2)
      ctx.stroke()

      // 2. 旋转虚线层
      ctx.rotate(rotation)
      ctx.setLineDash([10, 15])
      ctx.strokeStyle = this.colors.brightCyan
      ctx.lineWidth = 1
      ctx.beginPath()
      ctx.arc(0, 0, rBase * 1.0, 0, Math.PI * 2)
      ctx.stroke()

      ctx.rotate(-rotation * 2) // 反向旋转
      ctx.setLineDash([4, 8])
      ctx.beginPath()
      ctx.arc(0, 0, rBase * 0.85, 0, Math.PI * 2)
      ctx.stroke()

      // 3. 装饰点
      ctx.setLineDash([])
      for (let i = 0; i < 12; i++) {
        const angle = (i / 12) * Math.PI * 2
        const x = Math.cos(angle) * rBase * 0.7
        const y = Math.sin(angle) * rBase * 0.7
        ctx.beginPath()
        ctx.arc(x, y, 3, 0, Math.PI * 2)
        ctx.fillStyle = this.colors.brightCyan
        ctx.fill()
      }

      ctx.restore()
      
      // 4. 中心发光渐变 (不使用擦除，改用叠加)
      const gradient = ctx.createRadialGradient(cx, cy, 0, cx, cy, rBase * 1.5)
      gradient.addColorStop(0, 'rgba(0, 20, 40, 0)')
      gradient.addColorStop(0.6, 'rgba(0, 40, 60, 0.2)')
      gradient.addColorStop(1, 'rgba(0, 0, 0, 0)')
      ctx.fillStyle = gradient
      ctx.fillRect(0, 0, w, h)
    },
    render(timestamp = 0) {
      this.time = timestamp
      const canvas = this.$refs.canvasRef
      if (!canvas) return
      const ctx = canvas.getContext('2d')
      ctx.clearRect(0, 0, canvas.width, canvas.height)
      this.drawConcentricArcs(ctx, canvas.width / 2, canvas.height / 2, canvas.width, canvas.height)
      this.animationId = requestAnimationFrame(this.render)
    }
  }
}
</script>

<style scoped>
.cyberpunk-bg {
  position: fixed;
  inset: 0;
  width: 100vw;
  height: 100vh;
  pointer-events: none;
  background: transparent;
}
</style>