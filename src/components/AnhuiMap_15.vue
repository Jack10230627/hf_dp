<template>
  <div class="page-wrapper">
    <CyberpunkBackground class="hud-layer" />
    <div ref="container" class="map-container">
      <div v-if="loading" class="loading">正在加载安徽水利高清沙盘...</div>
    </div>

    <!-- 顶部标题栏 -->
    <header class="dashboard-header">
      <h1 class="main-title">数字孪生安徽指挥大屏</h1>
      <div class="title-decoration">
        <span class="line left"></span>
        <span class="glow-dot"></span>
        <span class="line right"></span>
      </div>
    </header>
    <!-- 底部导航栏 -->
    <footer class="dashboard-footer">
      <BottomNav />
    </footer>
  </div>
</template>

<script>
import * as THREE from "three"
import { OrbitControls } from "three/examples/jsm/controls/OrbitControls"
import { EffectComposer } from "three/examples/jsm/postprocessing/EffectComposer"
import { RenderPass } from "three/examples/jsm/postprocessing/RenderPass"
import { UnrealBloomPass } from "three/examples/jsm/postprocessing/UnrealBloomPass"
import { fromUrl } from "geotiff"
import BottomNav from './BottomNav.vue'
import CyberpunkBackground from './CyberpunkBackground.vue'

export default {
  name: "AnhuiSpaceMapFixed",
  components: {
    CyberpunkBackground,
    BottomNav
  },
  data() {
    return {
      loading: true
    }
  },
  created() {
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
    this.lakeMaterial = null
    this.lakeMaskData = null
    this.lakeMaskSize = 1024

    this.config = {
      width: 20000,
      height: 20000,
      maxH: 900,
      baseY: -400,
      hExp: 1.4,
      segments: 512,
      borderWidth: 30,
      riverBaseWidth: 8,
      lakeDepth: 150
    }
    this.labelConfig = {
      fontSize: 48,
      color: "#00ddff",
      color2: "#00ddff",
      strokeColor: "#003344",
      strokeWidth: 6,
      bgColor: "rgba(0, 30, 50, 0.7)",
      padding: 12,
      scale: 5,
      heightOffset: 200
    }
  },
  mounted() {
    this.$nextTick(() => {
      this.initThree()
      this.animate()
      window.addEventListener("resize", this.onResize)
    })
  },
  beforeDestroy() {
    window.removeEventListener("resize", this.onResize)
  },
  methods: {
    initThree() {
      const container = this.$refs.container
      if (!container) return

      this.scene = new THREE.Scene()

      this.camera = new THREE.PerspectiveCamera(35, container.clientWidth / container.clientHeight, 1, 1000000)
      this.camera.position.set(0, 25000, 30000)

      this.renderer = new THREE.WebGLRenderer({
        antialias: true,
        alpha: true,
        logarithmicDepthBuffer: true
      })
      this.renderer.setClearColor(0x000000, 0)
      this.renderer.setSize(container.clientWidth, container.clientHeight)
      this.renderer.setPixelRatio(Math.min(window.devicePixelRatio, 2))
      this.renderer.toneMapping = THREE.ACESFilmicToneMapping
      this.renderer.toneMappingExposure = 1.3
      container.appendChild(this.renderer.domElement)

      this.controls = new OrbitControls(this.camera, this.renderer.domElement)
      this.controls.enableDamping = true

      this.mapGroup = new THREE.Group()
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
        0.35, 0.5, 0.92
      )
      this.composer.addPass(bloom)

      this.loadData()
    },

    initSpaceBackground() {
      const starGeometry = new THREE.BufferGeometry()
      const starMaterial = new THREE.PointsMaterial({
        color: 0xffffff, size: 15, transparent: true, opacity: 0.8, sizeAttenuation: true
      })
      const starVertices = []
      for (let i = 0; i < 8000; i++) {
        const r = 200000 + Math.random() * 300000
        const theta = Math.random() * Math.PI * 2
        const phi = Math.acos(2 * Math.random() - 1)
        starVertices.push(
          r * Math.sin(phi) * Math.cos(theta),
          r * Math.sin(phi) * Math.sin(theta),
          r * Math.cos(phi)
        )
      }
      starGeometry.setAttribute('position', new THREE.Float32BufferAttribute(starVertices, 3))
      this.stars = new THREE.Points(starGeometry, starMaterial)
      this.scene.add(this.stars)
    },

    async loadData() {
      try {
        const [geoRes, tiffRes, riverRes, lakeRes] = await Promise.all([
          fetch("/anhui_border.geojson").then(res => res.json()),
          fromUrl("/anhui_height_float.tif"),
          fetch("/anhui_river.geojson").then(res => res.json()),
          fetch("/anhui_lake.geojson").then(res => res.json())
        ])
        const coords = this.extractCoords(geoRes)
        this.calculateBounds(coords)
        const image = await tiffRes.getImage()
        this.demData = await image.readRasters()
        this.demW = image.getWidth()
        this.demH = image.getHeight()
        let demMax = 0
        for (let i = 0; i < this.demData[0].length; i++) {
          const v = this.demData[0][i]
          if (v < 10000) demMax = Math.max(demMax, v)
        }

        // this.createLakeMask(lakeRes)

        this.buildMap(coords, demMax)
        this.createRivers(riverRes, demMax)
        // this.createLakes(lakeRes, demMax)
        this.loading = false
      } catch (err) {
        console.error("加载失败:", err)
      }
    },

    calculateBounds(coords) {
      const lons = coords.map(c => c[0])
      const lats = coords.map(c => c[1])
      this.bounds = {
        minLon: Math.min(...lons),
        maxLon: Math.max(...lons),
        minLat: Math.min(...lats),
        maxLat: Math.max(...lats)
      }
    },

    // ✨ 关键修复：边界保持 0（不沉），中心 1（全沉），用内描边挤压
    createLakeMask(lakeJson) {
      const size = this.lakeMaskSize
      const canvas = document.createElement("canvas")
      canvas.width = canvas.height = size
      const ctx = canvas.getContext("2d")

      // 黑色背景
      ctx.fillStyle = "black"
      ctx.fillRect(0, 0, size, size)

      // 1. 湖泊填白
      ctx.fillStyle = "white"
      lakeJson.features.forEach(feature => {
        const polygons = feature.geometry.type === "Polygon"
          ? [feature.geometry.coordinates]
          : feature.geometry.coordinates
        polygons.forEach(polygon => {
          const ring = polygon[0]
          if (!ring || ring.length < 3) return
          ctx.beginPath()
          ring.forEach((c, i) => {
            const u = (c[0] - this.bounds.minLon) / (this.bounds.maxLon - this.bounds.minLon)
            const v = (c[1] - this.bounds.minLat) / (this.bounds.maxLat - this.bounds.minLat)
            const x = u * size
            const y = (1 - v) * size
            if (i === 0) ctx.moveTo(x, y)
            else ctx.lineTo(x, y)
          })
          ctx.closePath()
          ctx.fill()
        })
      })

      // 2. ✨ 边界画黑色描边，挤压内部白色（让边界=0，不下沉）
      ctx.strokeStyle = "black"
      ctx.lineWidth = 6
      ctx.lineJoin = "round"
      lakeJson.features.forEach(feature => {
        const polygons = feature.geometry.type === "Polygon"
          ? [feature.geometry.coordinates]
          : feature.geometry.coordinates
        polygons.forEach(polygon => {
          const ring = polygon[0]
          if (!ring || ring.length < 3) return
          ctx.beginPath()
          ring.forEach((c, i) => {
            const u = (c[0] - this.bounds.minLon) / (this.bounds.maxLon - this.bounds.minLon)
            const v = (c[1] - this.bounds.minLat) / (this.bounds.maxLat - this.bounds.minLat)
            const x = u * size
            const y = (1 - v) * size
            if (i === 0) ctx.moveTo(x, y)
            else ctx.lineTo(x, y)
          })
          ctx.closePath()
          ctx.stroke()
        })
      })

      // 3. 轻微模糊让过渡自然
      ctx.filter = "blur(4px)"
      ctx.drawImage(canvas, 0, 0)
      ctx.filter = "none"

      this.lakeMaskData = ctx.getImageData(0, 0, size, size).data
    },

    getH(u, v, demMax) {
      const cx = u * (this.demW - 1)
      const cy = (1 - v) * (this.demH - 1)
      const data = this.demData[0]
      const getV = (x, y) => {
        x = Math.max(0, Math.min(this.demW - 1, x))
        y = Math.max(0, Math.min(this.demH - 1, y))
        const v = data[Math.floor(y) * this.demW + Math.floor(x)]
        return (isNaN(v) || v < 0) ? 0 : v
      }
      let h = Math.pow(getV(cx, cy) / demMax, this.config.hExp) * this.config.maxH

      // ✨ 应用湖泊凹陷
      if (this.lakeMaskData) {
        const lx = Math.floor(u * (this.lakeMaskSize - 1))
        const ly = Math.floor((1 - v) * (this.lakeMaskSize - 1))
        const idx = (ly * this.lakeMaskSize + lx) * 4
        const mask = this.lakeMaskData[idx] / 255
        h -= mask * this.config.lakeDepth
      }
      return h
    },

    getOriginalH(u, v, demMax) {
      const cx = u * (this.demW - 1)
      const cy = (1 - v) * (this.demH - 1)
      const data = this.demData[0]
      const getV = (x, y) => {
        x = Math.max(0, Math.min(this.demW - 1, x))
        y = Math.max(0, Math.min(this.demH - 1, y))
        const v = data[Math.floor(y) * this.demW + Math.floor(x)]
        return (isNaN(v) || v < 0) ? 0 : v
      }
      return Math.pow(getV(cx, cy) / demMax, this.config.hExp) * this.config.maxH
    },

    buildMap(coords, demMax) {
      const topGeom = new THREE.PlaneGeometry(this.config.width, this.config.height, this.config.segments, this.config.segments)
      topGeom.rotateX(-Math.PI / 2)
      const pos = topGeom.attributes.position
      const uv = topGeom.attributes.uv
      for (let i = 0; i < pos.count; i++) {
        pos.setY(i, this.getH(uv.getX(i), uv.getY(i), demMax))
      }
      topGeom.computeVertexNormals()

      const topMat = new THREE.ShaderMaterial({
        uniforms: {
          uMask: { value: this.createMask(coords) },
          uMaxH: { value: this.config.maxH }
        },
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
        const radius = name === "淮河"||name === "长江" ? this.config.riverBaseWidth * 5 : this.config.riverBaseWidth * 1.5

        lines.forEach(line => {
          const pts = line.map(c => {
            const u = (c[0] - this.bounds.minLon) / (this.bounds.maxLon - this.bounds.minLon)
            const v = (c[1] - this.bounds.minLat) / (this.bounds.maxLat - this.bounds.minLat)
            if (u < 0 || u > 1 || v < 0 || v > 1) return null
            return new THREE.Vector3((u - 0.5) * this.config.width, this.getOriginalH(u, v, demMax) + 15, -(v - 0.5) * this.config.height)
          }).filter(p => p !== null)
          if (pts.length < 2) return
          allSegmentPoints.push(pts)
          const tubeGeom = new THREE.TubeGeometry(new THREE.CatmullRomCurve3(pts, false, 'catmullrom', 0.2), pts.length * 2, radius, 6, false)
          const mat = new THREE.ShaderMaterial({
            uniforms: {
              uTime: { value: 0 },
              uColor: { value: name === "淮河"||name === "长江" ? new THREE.Color(0x00ffff) : new THREE.Color(0x0088ff) }
            },
            vertexShader: `varying vec2 vUv; void main(){ vUv = uv; gl_Position = projectionMatrix * modelViewMatrix * vec4(position, 1.0); }`,
            fragmentShader: `uniform float uTime; uniform vec3 uColor; varying vec2 vUv; void main(){ float f = fract(vUv.x * 3.0 - uTime * 0.8); float glow = smoothstep(0.4, 1.0, f); gl_FragColor = vec4(mix(uColor * 0.5, uColor * 2.0, glow), 0.8); }`,
            transparent: true,
            blending: THREE.AdditiveBlending,
            depthWrite: false
          })
          this.riverMaterials.push(mat)
          this.mapGroup.add(new THREE.Mesh(tubeGeom, mat))
        })

        if (allSegmentPoints.length > 0 && name !== "未命名" && !labeledRivers.has(name)) {
          const bestPos = this.findBestLabelPosition(allSegmentPoints)
          if (bestPos) {
            this.createRiverLabel(name, bestPos, "rivers")
            labeledRivers.add(name)
          }
        }
      })
    },

    findBestLabelPosition(allSegmentPoints) {
      let longest = null
      let maxL = 0
      allSegmentPoints.forEach(pts => {
        let l = 0
        for (let i = 1; i < pts.length; i++) l += pts[i].distanceTo(pts[i - 1])
        if (l > maxL) {
          maxL = l
          longest = pts
        }
      })
      if (!longest) return null
      const halfL = maxL / 2
      let acc = 0
      for (let i = 1; i < longest.length; i++) {
        const sl = longest[i].distanceTo(longest[i - 1])
        if (acc + sl >= halfL) return new THREE.Vector3().lerpVectors(longest[i - 1], longest[i], (halfL - acc) / sl)
        acc += sl
      }
      return longest[Math.floor(longest.length / 2)]
    },

    createRiverLabel(text, position, type) {
      const cfg = this.labelConfig
      const canvas = document.createElement("canvas")
      const ctx = canvas.getContext("2d")
      ctx.font = `bold ${cfg.fontSize}px "Microsoft YaHei", sans-serif`
      const tw = ctx.measureText(text).width
      const th = cfg.fontSize * 1.4
      canvas.width = tw + cfg.padding * 2
      canvas.height = th + cfg.padding * 2
      ctx.font = `bold ${cfg.fontSize}px "Microsoft YaHei", sans-serif`
      ctx.textAlign = "center"
      ctx.textBaseline = "middle"
      ctx.fillStyle = cfg.bgColor
      ctx.beginPath()
      ctx.roundRect(0, 0, canvas.width, canvas.height, 10)
      ctx.fill()
      ctx.strokeStyle = cfg.strokeColor
      ctx.lineWidth = 4
      ctx.stroke()
      ctx.fillStyle = type == "rivers" ? cfg.color : cfg.color2
      ctx.fillText(text, canvas.width / 2, canvas.height / 2)

      const spriteMat = new THREE.SpriteMaterial({
        map: new THREE.CanvasTexture(canvas),
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
      const canvas = document.createElement("canvas")
      canvas.width = canvas.height = 2048
      const ctx = canvas.getContext("2d")
      ctx.fillStyle = "black"
      ctx.fillRect(0, 0, 2048, 2048)
      ctx.fillStyle = "white"
      ctx.beginPath()
      coords.forEach((c, i) => {
        const u = (c[0] - this.bounds.minLon) / (this.bounds.maxLon - this.bounds.minLon)
        const v = (c[1] - this.bounds.minLat) / (this.bounds.maxLat - this.bounds.minLat)
        if (i === 0) ctx.moveTo(u * 2048, (1 - v) * 2048)
        else ctx.lineTo(u * 2048, (1 - v) * 2048)
      })
      ctx.fill()
      return new THREE.CanvasTexture(canvas)
    },

    createWalls(coords, demMax, baseY) {
      const vertices = []
      for (let i = 0; i < coords.length; i++) {
        const u = (coords[i][0] - this.bounds.minLon) / (this.bounds.maxLon - this.bounds.minLon)
        const v = (coords[i][1] - this.bounds.minLat) / (this.bounds.maxLat - this.bounds.minLat)
        vertices.push(
          (u - 0.5) * this.config.width, this.getH(u, v, demMax), -(v - 0.5) * this.config.height,
          (u - 0.5) * this.config.width, baseY, -(v - 0.5) * this.config.height
        )
      }
      const geom = new THREE.BufferGeometry()
      const indices = []
      for (let i = 0; i < (vertices.length / 6) - 1; i++) {
        indices.push(i * 2, i * 2 + 1, (i + 1) * 2, (i + 1) * 2, i * 2 + 1, (i + 1) * 2 + 1)
      }
      geom.setAttribute("position", new THREE.Float32BufferAttribute(vertices, 3))
      geom.setIndex(indices)
      geom.computeVertexNormals()
      this.mapGroup.add(new THREE.Mesh(geom, new THREE.ShaderMaterial({
        uniforms: {
          uBaseY: { value: baseY },
          uMaxH: { value: this.config.maxH }
        },
        vertexShader: `varying float vY; void main(){ vY = position.y; gl_Position = projectionMatrix * modelViewMatrix * vec4(position, 1.0); }`,
        fragmentShader: `uniform float uBaseY; uniform float uMaxH; varying float vY; void main(){ float f = (vY - uBaseY) / (uMaxH - uBaseY); gl_FragColor = vec4(mix(vec3(0.1, 0.1, 0.2), vec3(0.4, 0.3, 0.1), f), 1.0); }`,
        side: THREE.DoubleSide
      })))
    },

    extractCoords(json) {
      const feat = json.features[0].geometry
      return (feat.type === "Polygon" ? feat.coordinates[0] : feat.coordinates.reduce((a, b) => a[0].length > b[0].length ? a : b)[0]).filter(c => c && !isNaN(c[0]))
    },

    createLakes(lakeJson, demMax) {
      const labeledLakes = new Set()

      this.lakeMaterial = new THREE.ShaderMaterial({
        uniforms: {
          uTime: { value: 0 },
          uDeepColor: { value: new THREE.Color(0x0d4080) },
          uShallowColor: { value: new THREE.Color(0x4ab8ff) },
          uHighlight: { value: new THREE.Color(0x9fe8ff) },
          uEdgeColor: { value: new THREE.Color(0xb0f0ff) },
          uCameraPos: { value: new THREE.Vector3() }
        },
        vertexShader: `
      attribute float aEdgeDist;
      varying vec2 vUv;
      varying vec3 vWorldPos;
      varying vec3 vNormal;
      varying float vEdgeDist;
      void main() {
        vUv = uv;
        vEdgeDist = aEdgeDist;
        vec4 wp = modelMatrix * vec4(position, 1.0);
        vWorldPos = wp.xyz;
        vNormal = normalize(normalMatrix * normal);
        gl_Position = projectionMatrix * viewMatrix * wp;
      }
    `,
        fragmentShader: `
      uniform float uTime;
      uniform vec3 uDeepColor, uShallowColor, uHighlight, uEdgeColor, uCameraPos;
      varying vec2 vUv;
      varying vec3 vWorldPos;
      varying vec3 vNormal;
      varying float vEdgeDist;

      vec2 hash2(vec2 p){
        p = vec2(dot(p,vec2(127.1,311.7)), dot(p,vec2(269.5,183.3)));
        return -1.0 + 2.0*fract(sin(p)*43758.5453123);
      }
      float gnoise(vec2 p){
        vec2 i = floor(p), f = fract(p);
        vec2 u = f*f*(3.0-2.0*f);
        return mix(mix(dot(hash2(i), f),
                       dot(hash2(i+vec2(1,0)), f-vec2(1,0)), u.x),
                   mix(dot(hash2(i+vec2(0,1)), f-vec2(0,1)),
                       dot(hash2(i+vec2(1,1)), f-vec2(1,1)), u.x), u.y);
      }
      float fbm(vec2 p){
        float v = 0.0, a = 0.5;
        for(int i = 0; i < 5; i++){ v += a*gnoise(p); p *= 2.0; a *= 0.5; }
        return v;
      }

      void main() {
        vec2 flow1 = vUv * 12.0 + vec2(uTime * 0.05, uTime * 0.03);
        vec2 flow2 = vUv * 25.0 - vec2(uTime * 0.04, uTime * 0.06);
        float w1 = fbm(flow1);
        float w2 = fbm(flow2);
        float waves = w1 * 0.6 + w2 * 0.4;

        vec3 col = mix(uDeepColor, uShallowColor, smoothstep(-0.3, 0.5, waves));
        col *= 1.25;

        float d = distance(vUv, vec2(0.5));
        float ripple = sin(d * 80.0 - uTime * 2.5) * 0.5 + 0.5;
        ripple *= smoothstep(0.5, 0.0, d);
        col += uHighlight * ripple * 0.04;

        vec3 viewDir = normalize(uCameraPos - vWorldPos);
        float fres = pow(1.0 - max(dot(vNormal, viewDir), 0.0), 3.0);
        col += uHighlight * fres * 0.25;

        float sparkle = smoothstep(0.7, 0.82, w2) * smoothstep(0.65, 0.88, w1);
        col += uHighlight * sparkle * 0.6;

        float band = sin(vUv.y * 60.0 + waves * 4.0 + uTime * 0.6);
        band = smoothstep(0.9, 1.0, band);
        col += uHighlight * band * 0.1;

        float edgeGlow = 1.0 - smoothstep(0.0, 0.06, vEdgeDist);
        float edgePulse = 0.7 + 0.3 * sin(uTime * 2.0);
        col += uEdgeColor * edgeGlow * edgePulse * 1.1;

        float innerRing = smoothstep(0.05, 0.1, vEdgeDist) - smoothstep(0.1, 0.16, vEdgeDist);
        col += uEdgeColor * innerRing * 0.3;

        col = min(col, vec3(1.0, 1.1, 1.2));

        gl_FragColor = vec4(col, 0.95);
      }
    `,
        transparent: true,
        side: THREE.DoubleSide,
        depthWrite: false,
        blending: THREE.NormalBlending
      })

      lakeJson.features.forEach(feature => {
        const name = feature.properties.motorboat || feature.properties.NAME || "湖泊"
        const polygons = feature.geometry.type === "Polygon"
          ? [feature.geometry.coordinates]
          : feature.geometry.coordinates

        polygons.forEach(polygon => {
          const ring = polygon[0]
          if (!ring || ring.length < 3) return

          const shape = new THREE.Shape()
          const ptsXZ = []
          const ringHeights = []

          ring.forEach((coord, i) => {
            const u = (coord[0] - this.bounds.minLon) / (this.bounds.maxLon - this.bounds.minLon)
            const v = (coord[1] - this.bounds.minLat) / (this.bounds.maxLat - this.bounds.minLat)
            const x = (u - 0.5) * this.config.width
            const z = -(v - 0.5) * this.config.height
            if (i === 0) shape.moveTo(x, z); else shape.lineTo(x, z)
            ptsXZ.push({ x, z })
            ringHeights.push(this.getOriginalH(u, v, demMax))
          })

          const avgShoreH = ringHeights.reduce((a, b) => a + b, 0) / ringHeights.length
          const waterLevel = avgShoreH - 80

          const buildGeom = () => {
            const g = new THREE.ShapeGeometry(shape, 32)
            g.computeBoundingBox()
            const { min, max } = g.boundingBox
            const sx = max.x - min.x, sy = max.y - min.y
            const uvAttr = g.attributes.uv
            const posAttr = g.attributes.position

            const edgeDist = new Float32Array(posAttr.count)
            let maxDist = 0
            for (let i = 0; i < posAttr.count; i++) {
              const px = posAttr.getX(i)
              const py = posAttr.getY(i)
              let minD = Infinity
              for (let j = 0; j < ptsXZ.length; j++) {
                const dx = px - ptsXZ[j].x
                const dz = py - ptsXZ[j].z
                const dd = dx * dx + dz * dz
                if (dd < minD) minD = dd
              }
              minD = Math.sqrt(minD)
              edgeDist[i] = minD
              if (minD > maxDist) maxDist = minD
            }
            for (let i = 0; i < edgeDist.length; i++) {
              edgeDist[i] = maxDist > 0 ? edgeDist[i] / maxDist : 0
            }
            g.setAttribute('aEdgeDist', new THREE.BufferAttribute(edgeDist, 1))

            for (let i = 0; i < uvAttr.count; i++) {
              const px = posAttr.getX(i)
              const py = posAttr.getY(i)
              uvAttr.setXY(i, (px - min.x) / sx, (py - min.y) / sy)
            }
            return g
          }

          const waterMesh = new THREE.Mesh(buildGeom(), this.lakeMaterial)
          waterMesh.rotateX(Math.PI / 2)
          waterMesh.position.y = waterLevel
          waterMesh.renderOrder = 10
          this.mapGroup.add(waterMesh)

          // ✨ 贴地发光边界（每段细分 8 次，确保严丝合缝贴地）
          const denseEdgePts = []
          for (let i = 0; i < ring.length; i++) {
            const c0 = ring[i]
            const c1 = ring[(i + 1) % ring.length]
            const SUB = 8
            for (let s = 0; s < SUB; s++) {
              const t = s / SUB
              const lon = c0[0] + (c1[0] - c0[0]) * t
              const lat = c0[1] + (c1[1] - c0[1]) * t
              const u = (lon - this.bounds.minLon) / (this.bounds.maxLon - this.bounds.minLon)
              const v = (lat - this.bounds.minLat) / (this.bounds.maxLat - this.bounds.minLat)
              const x = (u - 0.5) * this.config.width
              const z = -(v - 0.5) * this.config.height
              // 用 getH（含凹陷）取真实当前地形高度
              const y = this.getH(u, v, demMax) + 8
              denseEdgePts.push(new THREE.Vector3(x, y, z))
            }
          }

          if (denseEdgePts.length > 2) {
            const curve = new THREE.CatmullRomCurve3(denseEdgePts, true)
            const edgeGeom = new THREE.TubeGeometry(curve, denseEdgePts.length * 2, 6, 8, true)
            const edgeMat = new THREE.MeshBasicMaterial({
              color: 0x88e8ff,
              transparent: true,
              opacity: 0.95,
              depthWrite: false,
              blending: THREE.AdditiveBlending
            })
            const edge = new THREE.Mesh(edgeGeom, edgeMat)
            edge.renderOrder = 11
            //this.mapGroup.add(edge)
          }

          // 质心 + 标签
          let area = 0, cx = 0, cz = 0
          for (let i = 0; i < ptsXZ.length; i++) {
            const p0 = ptsXZ[i], p1 = ptsXZ[(i + 1) % ptsXZ.length]
            const f = p0.x * p1.z - p1.x * p0.z
            area += f; cx += (p0.x + p1.x) * f; cz += (p0.z + p1.z) * f
          }
          area *= 0.5
          const centroidX = cx / (6 * area)
          const centroidZ = cz / (6 * area)

          if (name && name !== "湖泊" && !labeledLakes.has(name)) {
            this.createRiverLabel(name, new THREE.Vector3(centroidX, avgShoreH + 100, centroidZ), "lakers")
            labeledLakes.add(name)
          }
        })
      })
    },
    animate() {
      requestAnimationFrame(() => this.animate())
      this.controls.update()
      if (this.borderMat) this.borderMat.uniforms.uTime.value += 0.005
      if (this.lakeMaterial) {
        this.lakeMaterial.uniforms.uTime.value += 0.015
        this.lakeMaterial.uniforms.uCameraPos.value.copy(this.camera.position)
      }
      this.riverMaterials.forEach(m => m.uniforms.uTime.value += 0.02)
      if (this.stars) {
        this.stars.rotation.y += 0.0002
        this.stars.rotation.x += 0.0001
      }

      this.labelSprites.forEach(s => {
        const dist = this.camera.position.distanceTo(s.getWorldPosition(new THREE.Vector3()))
        const sFactor = dist / 25000
        const baseS = this.labelConfig.scale * 100
        s.scale.set(baseS * sFactor * (s.material.map.image.width / s.material.map.image.height), baseS * sFactor, 1)
      })
      this.composer.render()
    },

    onResize() {
      const container = this.$refs.container
      if (!container) return
      const w = container.clientWidth
      const h = container.clientHeight
      this.camera.aspect = w / h
      this.camera.updateProjectionMatrix()
      this.renderer.setSize(w, h)
      this.composer.setSize(w, h)
    }
  }
}
</script>

<style scoped>
.page-wrapper {
  position: relative;
  width: 100%;
  height: 100vh;
  overflow: hidden;
  background: #010409;
}

.map-container {
  position: absolute;
  inset: 0;
  z-index: 2;
}

.hud-layer {
  position: absolute !important;
  inset: 0 !important;
  width: 100% !important;
  height: 100% !important;
  z-index: 1 !important;
  pointer-events: none !important;
  opacity: 0.8;
}

.loading {
  font-size: 28px;
  position: absolute;
  top: 50%;
  left: 50%;
  transform: translate(-50%, -50%);
  color: #00ffff;
  z-index: 100;
}
</style>