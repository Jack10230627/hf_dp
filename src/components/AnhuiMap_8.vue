<template>
  <div ref="container" class="map-container">
    <div v-if="loading" class="loading">正在加载高清沙盘及水系...</div>
  </div>
</template>

<script>
import * as THREE from "three"
import { OrbitControls } from "three/examples/jsm/controls/OrbitControls"
import { EffectComposer } from "three/examples/jsm/postprocessing/EffectComposer"
import { RenderPass } from "three/examples/jsm/postprocessing/RenderPass"
import { UnrealBloomPass } from "three/examples/jsm/postprocessing/UnrealBloomPass"
import { fromUrl } from "geotiff"

export default {
  name: "AnhuiFinalMap",
  data() {
    return {
      loading: true,
      config: {
        width: 20000,
        height: 20000,
        maxH: 900,
        baseY: -400,
        hExp: 1.4,
        segments: 512,
        borderWidth: 30,
        riverBaseWidth: 8
      },
      labelConfig: {
        fontSize: 48,
        color: "#00ddff",
        strokeColor: "#003344",
        strokeWidth: 6,
        bgColor: "rgba(0, 30, 50, 0.7)",
        padding: 12,
        scale: 5,
        heightOffset: 200
      },
      riverMaterials: [],
      labelSprites: []
    }
  },
  mounted() {
    this.initThree()
    this.animate()
    window.addEventListener("resize", this.onResize)
  },
  methods: {
    initThree() {
      const container = this.$refs.container
      this.scene = new THREE.Scene()
      this.scene.background = new THREE.Color(0x02050a)

      this.camera = new THREE.PerspectiveCamera(35, container.clientWidth / container.clientHeight, 1, 500000)
      this.camera.position.set(0, 15000, 25000)

      this.renderer = new THREE.WebGLRenderer({ antialias: true, logarithmicDepthBuffer: true })
      this.renderer.setSize(container.clientWidth, container.clientHeight)
      this.renderer.setPixelRatio(window.devicePixelRatio)
      this.renderer.toneMapping = THREE.ACESFilmicToneMapping
      this.renderer.toneMappingExposure = 1.2
      container.appendChild(this.renderer.domElement)

      this.controls = new OrbitControls(this.camera, this.renderer.domElement)
      this.controls.enableDamping = true

      this.scene.add(new THREE.AmbientLight(0x4466aa, 0.6))
      const sun = new THREE.DirectionalLight(0xffffff, 2.0)
      sun.position.set(10000, 20000, 10000)
      this.scene.add(sun)

      this.composer = new EffectComposer(this.renderer)
      this.composer.addPass(new RenderPass(this.scene, this.camera))
      const bloom = new UnrealBloomPass(
        new THREE.Vector2(container.clientWidth, container.clientHeight),
        0.5, 0.4, 0.85
      )
      this.composer.addPass(bloom)

      this.loadData()
    },

    async loadData() {
      try {
        const [geoRes, tiffRes, riverRes] = await Promise.all([
          fetch("/anhui_border.geojson").then(res => res.json()),
          fromUrl("/anhui_height_float.tif"),
          fetch("/anhui_river.geojson").then(res => res.json())
        ])

        const coords = this.extractCoords(geoRes)
        this.calculateBounds(coords)

        const image = await tiffRes.getImage()
        this.demData = await image.readRasters()
        this.demW = image.getWidth(); this.demH = image.getHeight()
        let demMax = 0
        for (let i = 0; i < this.demData[0].length; i++) {
          const v = this.demData[0][i]; if (v < 10000) demMax = Math.max(demMax, v)
        }

        this.buildMap(coords, demMax)
        this.createRivers(riverRes, demMax)
        this.loading = false
      } catch (err) { console.error("加载失败:", err) }
    },

    calculateBounds(coords) {
      const lons = coords.map(c => c[0]), lats = coords.map(c => c[1])
      this.bounds = { minLon: Math.min(...lons), maxLon: Math.max(...lons), minLat: Math.min(...lats), maxLat: Math.max(...lats) }
    },

    getH(u, v, demMax) {
      const cx = u * (this.demW - 1), cy = (1 - v) * (this.demH - 1)
      const x1 = Math.floor(cx), y1 = Math.floor(cy)
      const dx = cx - x1, dy = cy - y1
      const data = this.demData[0]
      const getV = (x, y) => {
        x = Math.max(0, Math.min(this.demW - 1, x)); y = Math.max(0, Math.min(this.demH - 1, y))
        const v = data[y * this.demW + x]; return (isNaN(v) || v < 0) ? 0 : v
      }
      const v11 = getV(x1, y1), v21 = getV(x1 + 1, y1), v12 = getV(x1, y1 + 1), v22 = getV(x1 + 1, y1 + 1)
      return Math.pow(((v11 * (1 - dx) + v21 * dx) * (1 - dy) + (v12 * (1 - dx) + v22 * dx) * dy) / demMax, this.config.hExp) * this.config.maxH
    },

    // --- 修改后的核心函数 1 ---
    createRivers(riverJson, demMax) {
      const labeledRivers = new Set(); // 防止同一名称在多要素中重复标注

      riverJson.features.forEach(f => {
        const name = f.properties.name || "未命名"
        const lines = f.geometry.type === "LineString" ? [f.geometry.coordinates] : f.geometry.coordinates
        const allSegmentPoints = []

        // 核心要求：淮河加粗
        const radius = name === "淮河" ? this.config.riverBaseWidth * 5 : this.config.riverBaseWidth * 1.5

        lines.forEach(line => {
          const pts = line.map(c => {
            const u = (c[0] - this.bounds.minLon) / (this.bounds.maxLon - this.bounds.minLon)
            const v = (c[1] - this.bounds.minLat) / (this.bounds.maxLat - this.bounds.minLat)
            if (u < 0 || u > 1 || v < 0 || v > 1) return null
            return new THREE.Vector3((u - 0.5) * this.config.width, this.getH(u, v, demMax) + 15, -(v - 0.5) * this.config.height)
          }).filter(p => p !== null)

          if (pts.length < 2) return
          allSegmentPoints.push(pts)

          const curve = new THREE.CatmullRomCurve3(pts, false, 'catmullrom', 0.2)
          const tubeGeom = new THREE.TubeGeometry(curve, pts.length * 2, radius, 6, false)
          const mat = new THREE.ShaderMaterial({
            uniforms: { uTime: { value: 0 }, uColor: { value: name === "淮河" ? new THREE.Color(0x00ffff) : new THREE.Color(0x0088ff) } },
            vertexShader: `varying vec2 vUv; void main(){ vUv = uv; gl_Position = projectionMatrix * modelViewMatrix * vec4(position, 1.0); }`,
            fragmentShader: `
              uniform float uTime; uniform vec3 uColor; varying vec2 vUv;
              void main(){
                float f = fract(vUv.x * 3.0 - uTime * 0.8);
                float glow = smoothstep(0.4, 1.0, f);
                vec3 finalColor = mix(uColor * 0.5, uColor * 2.0, glow);
                gl_FragColor = vec4(finalColor, 0.8);
              }
            `,
            transparent: true,
            blending: THREE.AdditiveBlending,
            depthWrite: false
          })
          this.riverMaterials.push(mat)
          this.scene.add(new THREE.Mesh(tubeGeom, mat))
        })

        // 名称展示逻辑：重点河流且不重复
        if (allSegmentPoints.length > 0 && name !== "未命名" && !labeledRivers.has(name)) {
          const bestPosition = this.findBestLabelPosition(allSegmentPoints)
          if (bestPosition) {
            this.createRiverLabel(name, bestPosition)
            labeledRivers.add(name)
          }
        }
      })
    },

    // --- 修改后的核心函数 2 ---
    findBestLabelPosition(allSegmentPoints) {
      let longestSegment = null
      let maxLength = 0

      allSegmentPoints.forEach(points => {
        let length = 0
        for (let i = 1; i < points.length; i++) {
          length += points[i].distanceTo(points[i - 1])
        }
        if (length > maxLength) {
          maxLength = length
          longestSegment = points
        }
      })

      if (!longestSegment || longestSegment.length < 2) return null

      const halfLength = maxLength / 2
      let accumulated = 0

      for (let i = 1; i < longestSegment.length; i++) {
        const segLen = longestSegment[i].distanceTo(longestSegment[i - 1])
        if (accumulated + segLen >= halfLength) {
          const remaining = halfLength - accumulated
          const t = remaining / segLen
          return new THREE.Vector3().lerpVectors(longestSegment[i - 1], longestSegment[i], t)
        }
        accumulated += segLen
      }
      return longestSegment[Math.floor(longestSegment.length / 2)]
    },

    // --- 修改后的核心函数 3 ---
    createRiverLabel(text, position) {
      const cfg = this.labelConfig

      const canvas = document.createElement("canvas")
      const ctx = canvas.getContext("2d")

      ctx.font = `bold ${cfg.fontSize}px "Microsoft YaHei", sans-serif`
      const metrics = ctx.measureText(text)
      const textWidth = metrics.width
      const textHeight = cfg.fontSize * 1.4

      canvas.width = textWidth + cfg.padding * 2
      canvas.height = textHeight + cfg.padding * 2

      ctx.font = `bold ${cfg.fontSize}px "Microsoft YaHei", sans-serif`
      ctx.textAlign = "center"
      ctx.textBaseline = "middle"

      if (cfg.bgColor) {
        ctx.fillStyle = cfg.bgColor
        const r = 8
        ctx.beginPath()
        ctx.moveTo(r, 0)
        ctx.lineTo(canvas.width - r, 0)
        ctx.quadraticCurveTo(canvas.width, 0, canvas.width, r)
        ctx.lineTo(canvas.width, canvas.height - r)
        ctx.quadraticCurveTo(canvas.width, canvas.height, canvas.width - r, canvas.height)
        ctx.lineTo(r, canvas.height)
        ctx.quadraticCurveTo(0, canvas.height, 0, canvas.height - r)
        ctx.lineTo(0, r)
        ctx.quadraticCurveTo(0, 0, r, 0)
        ctx.closePath()
        ctx.fill()
      }

      if (cfg.strokeWidth > 0) {
        ctx.strokeStyle = cfg.strokeColor
        ctx.lineWidth = cfg.strokeWidth
        ctx.strokeText(text, canvas.width / 2, canvas.height / 2)
      }

      ctx.fillStyle = cfg.color
      ctx.fillText(text, canvas.width / 2, canvas.height / 2)

      const texture = new THREE.CanvasTexture(canvas)
      texture.minFilter = THREE.LinearFilter
      texture.magFilter = THREE.LinearFilter

      const spriteMat = new THREE.SpriteMaterial({
        map: texture,
        transparent: true,
        toneMapped: false,
        depthTest: false
      })

      const sprite = new THREE.Sprite(spriteMat)
      const aspect = canvas.width / canvas.height
      sprite.scale.set(cfg.scale * aspect * 100, cfg.scale * 100, 1)

      sprite.position.copy(position)
      sprite.position.y += cfg.heightOffset
      sprite.renderOrder = 999

      this.scene.add(sprite)
      this.labelSprites.push(sprite)
    },

    buildMap(coords, demMax) {
      const { width, height, segments, baseY } = this.config
      const topGeom = new THREE.PlaneGeometry(width, height, segments, segments)
      topGeom.rotateX(-Math.PI / 2)
      const pos = topGeom.attributes.position, uv = topGeom.attributes.uv
      for (let i = 0; i < pos.count; i++) {
        pos.setY(i, this.getH(uv.getX(i), uv.getY(i), demMax))
      }
      topGeom.computeVertexNormals()

      const maskTex = this.createMask(coords)
      const topMat = new THREE.ShaderMaterial({
        uniforms: { uMask: { value: maskTex }, uMaxH: { value: this.config.maxH } },
        vertexShader: `
          varying vec2 vUv; varying float vH; varying vec3 vNormal;
          void main(){
            vUv = uv; vH = position.y; vNormal = normalize(normalMatrix * normal);
            gl_Position = projectionMatrix * modelViewMatrix * vec4(position, 1.0);
          }
        `,
        fragmentShader: `
          uniform sampler2D uMask; uniform float uMaxH;
          varying vec2 vUv; varying float vH; varying vec3 vNormal;
          void main(){
            if(texture2D(uMask, vUv).r < 0.5) discard;
            float h = clamp(vH / uMaxH, 0.0, 1.0);
            vec3 c1 = vec3(0.02, 0.1, 0.25); vec3 c2 = vec3(0.1, 0.4, 0.3); vec3 c3 = vec3(0.5, 0.45, 0.25);
            vec3 color = h < 0.3 ? mix(c1, c2, h/0.3) : mix(c2, c3, (h-0.3)/0.7);
            float diff = max(dot(vNormal, normalize(vec3(0.5, 1.0, 0.5))), 0.0);
            gl_FragColor = vec4(color * (diff * 0.9 + 0.3), 1.0);
          }
        `
      })
      this.scene.add(new THREE.Mesh(topGeom, topMat))
      this.createGoldenBorder(coords, demMax)
      this.createWalls(coords, demMax, baseY)
    },

    createGoldenBorder(coords, demMax) {
      const points = coords.map(c => {
        const u = (c[0] - this.bounds.minLon) / (this.bounds.maxLon - this.bounds.minLon)
        const v = (c[1] - this.bounds.minLat) / (this.bounds.maxLat - this.bounds.minLat)
        return new THREE.Vector3((u - 0.5) * this.config.width, this.getH(u, v, demMax) + 25, -(v - 0.5) * this.config.height)
      })
      points.push(points[0].clone())
      const tubeGeom = new THREE.TubeGeometry(new THREE.CatmullRomCurve3(points), points.length, this.config.borderWidth, 8, true)
      this.borderMat = new THREE.ShaderMaterial({
        uniforms: { uTime: { value: 0 } },
        vertexShader: `varying vec2 vUv; void main(){ vUv = uv; gl_Position = projectionMatrix * modelViewMatrix * vec4(position, 1.0); }`,
        fragmentShader: `
          uniform float uTime; varying vec2 vUv;
          void main(){
            float flow = fract(vUv.x * 5.0 - uTime * 0.5);
            float spark = pow(smoothstep(0.6, 1.0, flow), 4.0);
            vec3 color = mix(vec3(0.6, 0.4, 0.1), vec3(1.0, 0.9, 0.5), spark);
            gl_FragColor = vec4(color * 1.5, 1.0);
          }
        `
      })
      this.scene.add(new THREE.Mesh(tubeGeom, this.borderMat))
    },

    createMask(coords) {
      const canvas = document.createElement("canvas"); canvas.width = canvas.height = 2048
      const ctx = canvas.getContext("2d")
      ctx.fillStyle = "black"; ctx.fillRect(0, 0, 2048, 2048); ctx.fillStyle = "white"; ctx.beginPath()
      coords.forEach((c, i) => {
        const u = (c[0] - this.bounds.minLon) / (this.bounds.maxLon - this.bounds.minLon)
        const v = (c[1] - this.bounds.minLat) / (this.bounds.maxLat - this.bounds.minLat)
        if (i === 0) ctx.moveTo(u * 2048, (1 - v) * 2048); else ctx.lineTo(u * 2048, (1 - v) * 2048)
      })
      ctx.fill(); return new THREE.CanvasTexture(canvas)
    },

    createWalls(coords, demMax, baseY) {
      const vertices = []
      for (let i = 0; i < coords.length; i++) {
        const u = (coords[i][0] - this.bounds.minLon) / (this.bounds.maxLon - this.bounds.minLon)
        const v = (coords[i][1] - this.bounds.minLat) / (this.bounds.maxLat - this.bounds.minLat)
        const h = this.getH(u, v, demMax)
        vertices.push((u - 0.5) * this.config.width, h, -(v - 0.5) * this.config.height, (u - 0.5) * this.config.width, baseY, -(v - 0.5) * this.config.height)
      }
      const geom = new THREE.BufferGeometry(), indices = []
      for (let i = 0; i < (vertices.length / 6) - 1; i++) {
        const a = i * 2, b = i * 2 + 1, c = (i + 1) * 2, d = (i + 1) * 2 + 1; indices.push(a, b, c, c, b, d)
      }
      geom.setAttribute("position", new THREE.Float32BufferAttribute(vertices, 3))
      geom.setIndex(indices); geom.computeVertexNormals()
      const mat = new THREE.ShaderMaterial({
        uniforms: { uBaseY: { value: baseY }, uMaxH: { value: this.config.maxH } },
        vertexShader: `varying float vY; void main(){ vY = position.y; gl_Position = projectionMatrix * modelViewMatrix * vec4(position, 1.0); }`,
        fragmentShader: `
          uniform float uBaseY; uniform float uMaxH; varying float vY;
          void main(){
            float f = (vY - uBaseY) / (uMaxH - uBaseY);
            gl_FragColor = vec4(mix(vec3(0.1, 0.1, 0.2), vec3(0.4, 0.3, 0.1), f), 1.0);
          }
        `,
        side: THREE.DoubleSide
      })
      this.scene.add(new THREE.Mesh(geom, mat))
    },

    extractCoords(json) {
      const feat = json.features[0].geometry
      return (feat.type === "Polygon" ? feat.coordinates[0] : feat.coordinates.reduce((a, b) => a[0].length > b[0].length ? a : b)[0]).filter(c => c && !isNaN(c[0]))
    },

    animate() {
      requestAnimationFrame(() => this.animate())
      this.controls.update()
      if (this.borderMat) this.borderMat.uniforms.uTime.value += 0.005
      this.riverMaterials.forEach(m => m.uniforms.uTime.value += 0.02)

      // 保持标签比例与视角距离同步
      this.labelSprites.forEach(s => {
        const dist = this.camera.position.distanceTo(s.position)
        const sFactor = dist / 20000
        const baseS = this.labelConfig.scale * 100
        s.scale.set(baseS * sFactor * (s.material.map.image.width / s.material.map.image.height), baseS * sFactor, 1)
      })

      this.composer.render()
    },

    onResize() {
      const w = this.$refs.container.clientWidth, h = this.$refs.container.clientHeight
      this.camera.aspect = w / h; this.camera.updateProjectionMatrix()
      this.renderer.setSize(w, h); this.composer.setSize(w, h)
    }
  }
}
</script>

<style scoped>
.map-container {
  width: 100%;
  height: 100vh;
  background: #02050a;
  overflow: hidden;
  position: relative;
}

.loading {
  position: absolute;
  top: 50%;
  left: 50%;
  transform: translate(-50%, -50%);
  color: #00ffff;
  font-family: sans-serif;
  letter-spacing: 2px;
}</style>