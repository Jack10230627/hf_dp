<template>
  <div ref="container" class="map-container">
    <div v-if="loading" class="loading">正在加载安徽水利高清沙盘...</div>
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
  name: "AnhuiSpaceMapFixed",
  data() {
    return {
      loading: true 
    }
  },
  created() {
    // 初始化非响应式变量
    this.scene = null
    this.camera = null
    this.renderer = null
    this.composer = null
    this.controls = null
    this.mapGroup = null 
    this.stars = null
    this.riverMaterials = []
    this.labelSprites = []
    this.borderMat = null

    this.config = {
      width: 20000,
      height: 20000,
      maxH: 900,
      baseY: -400,
      hExp: 1.4,
      segments: 512,
      borderWidth: 30,
      riverBaseWidth: 8
    }
    this.labelConfig = {
      fontSize: 48,
      color: "#00ddff",
      strokeColor: "#003344",
      strokeWidth: 6,
      bgColor: "rgba(0, 30, 50, 0.7)",
      padding: 12,
      scale: 5,
      heightOffset: 200
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
      this.scene.background = new THREE.Color(0x010409)

      this.camera = new THREE.PerspectiveCamera(35, container.clientWidth / container.clientHeight, 1, 1000000)
      // 调整相机位置以匹配图2的视角（更接近正上方的俯视）
      this.camera.position.set(0, 25000, 25000)

      this.renderer = new THREE.WebGLRenderer({ antialias: true, logarithmicDepthBuffer: true })
      this.renderer.setSize(container.clientWidth, container.clientHeight)
      this.renderer.setPixelRatio(Math.min(window.devicePixelRatio, 2))
      this.renderer.toneMapping = THREE.ACESFilmicToneMapping
      this.renderer.toneMappingExposure = 1.3
      container.appendChild(this.renderer.domElement)

      this.controls = new OrbitControls(this.camera, this.renderer.domElement)
      this.controls.enableDamping = true

      // 地图整体组
      this.mapGroup = new THREE.Group()
      // --- 关键修改：将旋转设为 0 以匹配图2的纵向效果 ---
      this.mapGroup.rotation.y = 0 
      this.scene.add(this.mapGroup)

      this.scene.add(new THREE.AmbientLight(0x4466aa, 0.7))
      const sun = new THREE.DirectionalLight(0xffffff, 2.2)
      sun.position.set(10000, 20000, 10000)
      this.scene.add(sun)

      this.initSpaceBackground()

      this.composer = new EffectComposer(this.renderer)
      this.composer.addPass(new RenderPass(this.scene, this.camera))
      const bloom = new UnrealBloomPass(
        new THREE.Vector2(container.clientWidth, container.clientHeight),
        0.55, 0.4, 0.85
      )
      this.composer.addPass(bloom)

      this.loadData()
    },

    initSpaceBackground() {
      const starGeometry = new THREE.BufferGeometry()
      const starMaterial = new THREE.PointsMaterial({ color: 0xffffff, size: 15, transparent: true, opacity: 0.8, sizeAttenuation: true })
      const starVertices = []
      for (let i = 0; i < 8000; i++) {
        const r = 200000 + Math.random() * 300000
        const theta = Math.random() * Math.PI * 2
        const phi = Math.acos(2 * Math.random() - 1)
        starVertices.push(r * Math.sin(phi) * Math.cos(theta), r * Math.sin(phi) * Math.sin(theta), r * Math.cos(phi))
      }
      starGeometry.setAttribute('position', new THREE.Float32BufferAttribute(starVertices, 3))
      this.stars = new THREE.Points(starGeometry, starMaterial)
      this.scene.add(this.stars) 

      const skyGeo = new THREE.SphereGeometry(600000, 32, 32)
      const skyMat = new THREE.ShaderMaterial({
        side: THREE.BackSide,
        uniforms: { topColor: { value: new THREE.Color(0x020a1a) }, bottomColor: { value: new THREE.Color(0x000000) } },
        vertexShader: `varying vec3 vWorldPosition; void main() { vec4 worldPosition = modelMatrix * vec4( position, 1.0 ); vWorldPosition = worldPosition.xyz; gl_Position = projectionMatrix * modelViewMatrix * vec4( position, 1.0 ); }`,
        fragmentShader: `varying vec3 vWorldPosition; uniform vec3 topColor; uniform vec3 bottomColor; void main() { float h = normalize(vWorldPosition).y * 0.5 + 0.5; gl_FragColor = vec4(mix(bottomColor, topColor, h), 1.0); }`
      })
      this.scene.add(new THREE.Mesh(skyGeo, skyMat))
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
      const data = this.demData[0]
      const getV = (x, y) => {
        x = Math.max(0, Math.min(this.demW - 1, x)); y = Math.max(0, Math.min(this.demH - 1, y))
        const v = data[Math.floor(y) * this.demW + Math.floor(x)]; return (isNaN(v) || v < 0) ? 0 : v
      }
      return Math.pow(getV(cx, cy) / demMax, this.config.hExp) * this.config.maxH
    },

    buildMap(coords, demMax) {
      const topGeom = new THREE.PlaneGeometry(this.config.width, this.config.height, this.config.segments, this.config.segments)
      topGeom.rotateX(-Math.PI / 2)
      const pos = topGeom.attributes.position, uv = topGeom.attributes.uv
      for (let i = 0; i < pos.count; i++) pos.setY(i, this.getH(uv.getX(i), uv.getY(i), demMax))
      topGeom.computeVertexNormals()

      const topMat = new THREE.ShaderMaterial({
        uniforms: { uMask: { value: this.createMask(coords) }, uMaxH: { value: this.config.maxH } },
        vertexShader: `varying vec2 vUv; varying float vH; varying vec3 vNormal; void main(){ vUv = uv; vH = position.y; vNormal = normalize(normalMatrix * normal); gl_Position = projectionMatrix * modelViewMatrix * vec4(position, 1.0); }`,
        fragmentShader: `uniform sampler2D uMask; uniform float uMaxH; varying vec2 vUv; varying float vH; varying vec3 vNormal; void main(){ if(texture2D(uMask, vUv).r < 0.5) discard; float h = clamp(vH / uMaxH, 0.0, 1.0); vec3 c1 = vec3(0.01, 0.1, 0.25); vec3 c2 = vec3(0.1, 0.4, 0.35); vec3 c3 = vec3(0.5, 0.45, 0.25); vec3 color = h < 0.3 ? mix(c1, c2, h/0.3) : mix(c2, c3, (h-0.3)/0.7); float diff = max(dot(vNormal, normalize(vec3(0.5, 1.0, 0.5))), 0.0); gl_FragColor = vec4(color * (diff * 0.9 + 0.35), 1.0); }`
      })
      this.mapGroup.add(new THREE.Mesh(topGeom, topMat)) 
      this.createGoldenBorder(coords, demMax)
      this.createWalls(coords, demMax, this.config.baseY)
    },

    createRivers(riverJson, demMax) {
      const labeledRivers = new Set()
      riverJson.features.forEach(f => {
        const name = f.properties.name || "未命名"
        const lines = f.geometry.type === "LineString" ? [f.geometry.coordinates] : f.geometry.coordinates
        const allSegmentPoints = []
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
          const tubeGeom = new THREE.TubeGeometry(new THREE.CatmullRomCurve3(pts, false, 'catmullrom', 0.2), pts.length * 2, radius, 6, false)
          const mat = new THREE.ShaderMaterial({
            uniforms: { uTime: { value: 0 }, uColor: { value: name === "淮河" ? new THREE.Color(0x00ffff) : new THREE.Color(0x0088ff) } },
            vertexShader: `varying vec2 vUv; void main(){ vUv = uv; gl_Position = projectionMatrix * modelViewMatrix * vec4(position, 1.0); }`,
            fragmentShader: `uniform float uTime; uniform vec3 uColor; varying vec2 vUv; void main(){ float f = fract(vUv.x * 3.0 - uTime * 0.8); float glow = smoothstep(0.4, 1.0, f); gl_FragColor = vec4(mix(uColor * 0.5, uColor * 2.0, glow), 0.8); }`,
            transparent: true, blending: THREE.AdditiveBlending, depthWrite: false
          })
          this.riverMaterials.push(mat)
          this.mapGroup.add(new THREE.Mesh(tubeGeom, mat)) 
        })

        if (allSegmentPoints.length > 0 && name !== "未命名" && !labeledRivers.has(name)) {
          const bestPos = this.findBestLabelPosition(allSegmentPoints)
          if (bestPos) { this.createRiverLabel(name, bestPos); labeledRivers.add(name) }
        }
      })
    },

    findBestLabelPosition(allSegmentPoints) {
      let longest = null, maxL = 0
      allSegmentPoints.forEach(pts => {
        let l = 0; for (let i = 1; i < pts.length; i++) l += pts[i].distanceTo(pts[i - 1])
        if (l > maxL) { maxL = l; longest = pts }
      })
      if (!longest) return null
      const halfL = maxL / 2; let acc = 0
      for (let i = 1; i < longest.length; i++) {
        const sl = longest[i].distanceTo(longest[i - 1])
        if (acc + sl >= halfL) return new THREE.Vector3().lerpVectors(longest[i - 1], longest[i], (halfL - acc) / sl)
        acc += sl
      }
      return longest[Math.floor(longest.length / 2)]
    },

    createRiverLabel(text, position) {
      const cfg = this.labelConfig
      const canvas = document.createElement("canvas")
      const ctx = canvas.getContext("2d")
      ctx.font = `bold ${cfg.fontSize}px "Microsoft YaHei", sans-serif`
      const tw = ctx.measureText(text).width, th = cfg.fontSize * 1.4
      canvas.width = tw + cfg.padding * 2; canvas.height = th + cfg.padding * 2
      ctx.font = `bold ${cfg.fontSize}px "Microsoft YaHei", sans-serif`
      ctx.textAlign = "center"; ctx.textBaseline = "middle"
      ctx.fillStyle = cfg.bgColor; ctx.beginPath(); ctx.roundRect(0, 0, canvas.width, canvas.height, 10); ctx.fill()
      ctx.strokeStyle = cfg.strokeColor; ctx.lineWidth = 4; ctx.stroke()
      ctx.fillStyle = cfg.color; ctx.fillText(text, canvas.width / 2, canvas.height / 2)
      
      const spriteMat = new THREE.SpriteMaterial({ map: new THREE.CanvasTexture(canvas), transparent: true, toneMapped: false, depthTest: false })
      const sprite = new THREE.Sprite(spriteMat)
      const aspect = canvas.width / canvas.height
      sprite.scale.set(cfg.scale * aspect * 100, cfg.scale * 100, 1)
      
      // 这里的 position 是原始本地坐标
      sprite.position.copy(position)
      sprite.position.y += cfg.heightOffset
      sprite.renderOrder = 999
      
      // --- 关键修改：将标签添加到 mapGroup 组中 ---
      // 这样当地图旋转时，标签会跟随 mapGroup 自动变换坐标
      this.mapGroup.add(sprite) 
      this.labelSprites.push(sprite)
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
        fragmentShader: `uniform float uTime; varying vec2 vUv; void main(){ float flow = fract(vUv.x * 5.0 - uTime * 0.5); float spark = pow(smoothstep(0.6, 1.0, flow), 4.0); gl_FragColor = vec4(mix(vec3(0.6, 0.4, 0.1), vec3(1.0, 0.9, 0.5), spark) * 1.5, 1.0); }`
      })
      this.mapGroup.add(new THREE.Mesh(tubeGeom, this.borderMat))
    },

    createMask(coords) {
      const canvas = document.createElement("canvas"); canvas.width = canvas.height = 2048
      const ctx = canvas.getContext("2d"); ctx.fillStyle = "black"; ctx.fillRect(0, 0, 2048, 2048)
      ctx.fillStyle = "white"; ctx.beginPath()
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
        vertices.push((u - 0.5) * this.config.width, this.getH(u, v, demMax), -(v - 0.5) * this.config.height, (u - 0.5) * this.config.width, baseY, -(v - 0.5) * this.config.height)
      }
      const geom = new THREE.BufferGeometry(), indices = []
      for (let i = 0; i < (vertices.length / 6) - 1; i++) indices.push(i * 2, i * 2 + 1, (i + 1) * 2, (i + 1) * 2, i * 2 + 1, (i + 1) * 2 + 1)
      geom.setAttribute("position", new THREE.Float32BufferAttribute(vertices, 3))
      geom.setIndex(indices); geom.computeVertexNormals()
      this.mapGroup.add(new THREE.Mesh(geom, new THREE.ShaderMaterial({
        uniforms: { uBaseY: { value: baseY }, uMaxH: { value: this.config.maxH } },
        vertexShader: `varying float vY; void main(){ vY = position.y; gl_Position = projectionMatrix * modelViewMatrix * vec4(position, 1.0); }`,
        fragmentShader: `uniform float uBaseY; uniform float uMaxH; varying float vY; void main(){ float f = (vY - uBaseY) / (uMaxH - uBaseY); gl_FragColor = vec4(mix(vec3(0.1, 0.1, 0.2), vec3(0.4, 0.3, 0.1), f), 1.0); }`,
        side: THREE.DoubleSide
      })))
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
      if (this.stars) { this.stars.rotation.y += 0.0002; this.stars.rotation.x += 0.0001 }
      
      this.labelSprites.forEach(s => {
        // 使用 getWorldPosition 获取经过 mapGroup 旋转后的真实世界坐标
        const dist = this.camera.position.distanceTo(s.getWorldPosition(new THREE.Vector3()))
        const sFactor = dist / 25000
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
.map-container { width: 100%; height: 100vh; background: #010409; overflow: hidden; position: relative; }
.loading {font-size: 28px; position: absolute; top: 50%; left: 50%; transform: translate(-50%, -50%); color: #00ffff; font-family: "Microsoft YaHei"; letter-spacing: 2px; }
</style>