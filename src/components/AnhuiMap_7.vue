<template>
  <div ref="container" class="map-container">
    <div v-if="loading" class="loading">正在加载地形与边界...</div>
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
  name: "AnhuiMap",
  data() {
    return {
      loading: true,
      config: {
        width: 20000,
        height: 20000,
        maxH: 850,
        baseY: -600,
        hExp: 1.5,
        segments: 1023,
        wallStep: 6
      },
      riverMaterials: [],
      labelConfig: {
        fontSize: 48,
        color: "#00ddff",
        strokeColor: "#003344",
        strokeWidth: 6,
        bgColor: "rgba(0, 30, 50, 0.7)",
        padding: 12,
        scale: 8,
        heightOffset: 200
      },
      labelSprites: []
    }
  },
  mounted() {
    this.initThree()
    this.animate()
    window.addEventListener("resize", this.onResize)
  },
  beforeDestroy() {
    window.removeEventListener("resize", this.onResize)
  },
  methods: {
    initThree() {
      const container = this.$refs.container

      this.scene = new THREE.Scene()
      this.createGradientBackground()
      this.createStarfield()

      this.camera = new THREE.PerspectiveCamera(
        45,
        container.clientWidth / container.clientHeight,
        1,
        500000
      )
      this.camera.position.set(0, 14000, 24000)

      this.renderer = new THREE.WebGLRenderer({ antialias: true, alpha: true })
      this.renderer.setSize(container.clientWidth, container.clientHeight)
      this.renderer.setPixelRatio(window.devicePixelRatio)
      this.renderer.toneMapping = THREE.ACESFilmicToneMapping
      this.renderer.toneMappingExposure = 1.0
      this.renderer.setClearColor(0x000000, 0)
      container.appendChild(this.renderer.domElement)

      this.controls = new OrbitControls(this.camera, this.renderer.domElement)
      this.controls.enableDamping = true

      // 光照
      this.scene.add(new THREE.AmbientLight(0x3366aa, 0.25))

      const sun = new THREE.DirectionalLight(0xffeecc, 2.8)
      sun.position.set(12000, 18000, 8000)
      this.scene.add(sun)

      const fill = new THREE.DirectionalLight(0x66aaff, 0.5)
      fill.position.set(-10000, 6000, -5000)
      this.scene.add(fill)

      const rim = new THREE.DirectionalLight(0xffc060, 0.5)
      rim.position.set(0, 3000, -12000)
      this.scene.add(rim)

      // Bloom
      this.composer = new EffectComposer(this.renderer)
      this.composer.addPass(new RenderPass(this.scene, this.camera))
      const bloom = new UnrealBloomPass(
        new THREE.Vector2(container.clientWidth, container.clientHeight),
        0.5,
        0.6,
        0.85
      )
      this.composer.addPass(bloom)

      this.loadData()
    },

    createGradientBackground() {
      const canvas = document.createElement("canvas")
      canvas.width = 2
      canvas.height = 512
      const ctx = canvas.getContext("2d")
      const grad = ctx.createLinearGradient(0, 0, 0, 512)
      grad.addColorStop(0, "#0a1530")
      grad.addColorStop(0.5, "#050e1c")
      grad.addColorStop(1, "#020610")
      ctx.fillStyle = grad
      ctx.fillRect(0, 0, 2, 512)
      const tex = new THREE.CanvasTexture(canvas)
      this.scene.background = tex
    },

    createStarfield() {
      const starGeom = new THREE.BufferGeometry()
      const starCount = 800
      const positions = new Float32Array(starCount * 3)
      const sizes = new Float32Array(starCount)

      for (let i = 0; i < starCount; i++) {
        const radius = 80000 + Math.random() * 50000
        const theta = Math.random() * Math.PI * 2
        const phi = Math.acos(2 * Math.random() - 1)
        positions[i * 3] = radius * Math.sin(phi) * Math.cos(theta)
        positions[i * 3 + 1] = radius * Math.cos(phi) * 0.3 + 20000
        positions[i * 3 + 2] = radius * Math.sin(phi) * Math.sin(theta)
        sizes[i] = Math.random() * 80 + 20
      }

      starGeom.setAttribute("position", new THREE.Float32BufferAttribute(positions, 3))
      starGeom.setAttribute("size", new THREE.Float32BufferAttribute(sizes, 1))

      const starMat = new THREE.ShaderMaterial({
        uniforms: { uColor: { value: new THREE.Color(0xaaccff) } },
        vertexShader: `
          attribute float size;
          void main(){
            vec4 mvPosition = modelViewMatrix * vec4(position, 1.0);
            gl_PointSize = size * (300.0 / -mvPosition.z);
            gl_Position = projectionMatrix * mvPosition;
          }
        `,
        fragmentShader: `
          uniform vec3 uColor;
          void main(){
            vec2 c = gl_PointCoord - 0.5;
            float d = length(c);
            if(d > 0.5) discard;
            float alpha = smoothstep(0.5, 0.0, d);
            gl_FragColor = vec4(uColor, alpha * 0.8);
          }
        `,
        transparent: true,
        depthWrite: false
      })

      this.scene.add(new THREE.Points(starGeom, starMat))
    },

    async loadData() {
      try {
        const geoRes = await fetch("/anhui_border.geojson")
        const geojson = await geoRes.json()
        const coords = this.extractCoords(geojson)

        const lons = coords.map(c => c[0])
        const lats = coords.map(c => c[1])
        this.bounds = {
          minLon: Math.min(...lons),
          maxLon: Math.max(...lons),
          minLat: Math.min(...lats),
          maxLat: Math.max(...lats)
        }

        const tiff = await fromUrl("/anhui_height_float.tif")
        const image = await tiff.getImage()
        const raster = await image.readRasters()
        const demData = raster[0]
        const demW = image.getWidth()
        const demH = image.getHeight()

        let demMax = 0
        for (let i = 0; i < demData.length; i++) {
          const v = demData[i]
          if (!isNaN(v) && v < 10000) demMax = Math.max(demMax, v)
        }

        console.log("DEM 分辨率:", demW, "x", demH, "最大高度:", demMax)

        this.buildMap(coords, demData, demW, demH, demMax)
        await this.loadRivers()
        this.loading = false
      } catch (err) {
        console.error(err)
      }
    },

    extractCoords(json) {
      const feat = json.features[0].geometry
      let raw =
        feat.type === "Polygon"
          ? feat.coordinates[0]
          : feat.coordinates.reduce((a, b) =>
              a[0].length > b[0].length ? a : b
            )[0]
      return raw.filter(c => c && !isNaN(c[0]))
    },

    getPos(lon, lat) {
      const u = (lon - this.bounds.minLon) / (this.bounds.maxLon - this.bounds.minLon)
      const v = (lat - this.bounds.minLat) / (this.bounds.maxLat - this.bounds.minLat)
      return {
        x: (u - 0.5) * this.config.width,
        z: -(v - 0.5) * this.config.height,
        u,
        v
      }
    },

    buildMap(coords, demData, demW, demH, demMax) {
      const { width, height, maxH, baseY, hExp, segments, wallStep } = this.config

      // 高度采样（最简单，地形最低 = 0）
      const getH = (u, v) => {
        const cx = u * (demW - 1)
        const cy = (1 - v) * (demH - 1)
        const weights = [[1,2,1],[2,4,2],[1,2,1]]
        let sum = 0, weightSum = 0
        for (let dy = -1; dy <= 1; dy++) {
          for (let dx = -1; dx <= 1; dx++) {
            const tx = Math.max(0, Math.min(demW - 1, Math.floor(cx + dx)))
            const ty = Math.max(0, Math.min(demH - 1, Math.floor(cy + dy)))
            let val = demData[ty * demW + tx]
            if (isNaN(val) || val < 0) val = 0
            const w = weights[dy + 1][dx + 1]
            sum += val * w
            weightSum += w
          }
        }
        const avg = sum / weightSum
        if (avg <= 0) return 0
        return Math.pow(avg / demMax, hExp) * maxH
      }

      this.getH = getH

      // ===== 顶面地形 =====
      const topGeom = new THREE.PlaneGeometry(width, height, segments, segments)
      topGeom.rotateX(-Math.PI / 2)

      const posAttr = topGeom.attributes.position
      const uvAttr = topGeom.attributes.uv
      for (let i = 0; i < posAttr.count; i++) {
        posAttr.setY(i, getH(uvAttr.getX(i), uvAttr.getY(i)))
      }

      // 轻度平滑（1 次）
      const segs = segments + 1
      for (let iter = 0; iter < 1; iter++) {
        const newHeights = new Float32Array(posAttr.count)
        for (let i = 0; i < posAttr.count; i++) {
          const row = Math.floor(i / segs)
          const col = i % segs
          let sum = posAttr.getY(i)
          let count = 1
          if (row > 0) { sum += posAttr.getY(i - segs); count++ }
          if (row < segs - 1) { sum += posAttr.getY(i + segs); count++ }
          if (col > 0) { sum += posAttr.getY(i - 1); count++ }
          if (col < segs - 1) { sum += posAttr.getY(i + 1); count++ }
          newHeights[i] = sum / count
        }
        for (let i = 0; i < posAttr.count; i++) {
          posAttr.setY(i, newHeights[i])
        }
      }

      posAttr.needsUpdate = true
      topGeom.computeVertexNormals()

      // mask
      const maskCanvas = document.createElement("canvas")
      maskCanvas.width = 2048
      maskCanvas.height = 2048
      const ctx = maskCanvas.getContext("2d")
      ctx.fillStyle = "black"
      ctx.fillRect(0, 0, 2048, 2048)
      ctx.fillStyle = "white"
      ctx.beginPath()
      coords.forEach((c, i) => {
        const p = this.getPos(c[0], c[1])
        const x = p.u * 2048
        const y = (1 - p.v) * 2048
        if (i === 0) ctx.moveTo(x, y)
        else ctx.lineTo(x, y)
      })
      ctx.closePath()
      ctx.fill()
      ctx.lineWidth = 10
      ctx.strokeStyle = "white"
      ctx.stroke()

      const maskTex = new THREE.CanvasTexture(maskCanvas)
      maskTex.minFilter = THREE.LinearFilter
      maskTex.magFilter = THREE.LinearFilter

      const topMat = new THREE.ShaderMaterial({
        uniforms: {
          uMask: { value: maskTex },
          uMaxH: { value: maxH }
        },
        vertexShader: `
          varying vec2 vUv;
          varying float vH;
          varying vec3 vNormal;
          varying vec3 vWorldPos;
          void main(){
            vUv = uv;
            vH = position.y;
            vNormal = normalize(normalMatrix * normal);
            vec4 wp = modelMatrix * vec4(position, 1.0);
            vWorldPos = wp.xyz;
            gl_Position = projectionMatrix * viewMatrix * wp;
          }
        `,
        fragmentShader: `
          uniform sampler2D uMask;
          uniform float uMaxH;
          varying vec2 vUv;
          varying float vH;
          varying vec3 vNormal;
          varying vec3 vWorldPos;

          void main(){
            if(texture2D(uMask, vUv).r < 0.5) discard;
            float h = clamp(vH / uMaxH, 0.0, 1.0);
            float hColor = pow(h, 0.6);

            vec3 c0 = vec3(0.04, 0.10, 0.25);
            vec3 c1 = vec3(0.05, 0.18, 0.38);
            vec3 c2 = vec3(0.00, 0.60, 0.80);
            vec3 c3 = vec3(0.15, 0.55, 0.35);
            vec3 c4 = vec3(0.40, 0.70, 0.25);
            vec3 c5 = vec3(0.85, 0.75, 0.30);
            vec3 c6 = vec3(0.85, 0.55, 0.15);
            vec3 c7 = vec3(0.65, 0.30, 0.10);
            vec3 c8 = vec3(0.45, 0.20, 0.10);

            vec3 color;
            if(hColor < 0.10)      color = mix(c0, c1, hColor / 0.10);
            else if(hColor < 0.18) color = mix(c1, c2, (hColor - 0.10) / 0.08);
            else if(hColor < 0.32) color = mix(c2, c3, (hColor - 0.18) / 0.14);
            else if(hColor < 0.50) color = mix(c3, c4, (hColor - 0.32) / 0.18);
            else if(hColor < 0.65) color = mix(c4, c5, (hColor - 0.50) / 0.15);
            else if(hColor < 0.80) color = mix(c5, c6, (hColor - 0.65) / 0.15);
            else if(hColor < 0.92) color = mix(c6, c7, (hColor - 0.80) / 0.12);
            else                   color = mix(c7, c8, (hColor - 0.92) / 0.08);

            vec3 lightDir = normalize(vec3(0.5, 0.85, 0.3));
            float diff = max(dot(vNormal, lightDir), 0.0);
            float mainLight = pow(diff, 0.8) * 1.2;
            
            vec3 fillDir = normalize(vec3(-0.5, 0.3, -0.5));
            float fillLight = max(dot(vNormal, fillDir), 0.0) * 0.2;
            
            float ambient = 0.25;
            float lighting = mainLight + fillLight + ambient;
            color *= lighting;

            if(h > 0.5){
              float ridge = smoothstep(0.5, 1.0, h) * 0.15;
              color += vec3(1.0, 0.8, 0.4) * ridge;
            }

            if(h < 0.18){
              float glow = (1.0 - h / 0.18) * 0.25;
              color += vec3(0.0, 0.5, 0.8) * glow;
            }

            if(diff < 0.3){
              float shadow = (0.3 - diff) / 0.3;
              color *= (1.0 - shadow * 0.4);
            }

            gl_FragColor = vec4(color, 1.0);
          }
        `
      })
      this.scene.add(new THREE.Mesh(topGeom, topMat))

      // ===== 侧墙：顶部跟随地形真实高度 =====
      const wallCoords = []
      for (let i = 0; i < coords.length; i += wallStep) {
        wallCoords.push(coords[i])
      }
      if (wallCoords[wallCoords.length - 1] !== wallCoords[0]) {
        wallCoords.push(wallCoords[0])
      }

      const wallVertices = []
      const wallIndices = []

      for (let i = 0; i < wallCoords.length; i++) {
        const c = wallCoords[i]
        const p = this.getPos(c[0], c[1])
        const terrainH = getH(p.u, p.v)   // 关键：侧壁顶部 = 该点地形高度

        wallVertices.push(p.x, terrainH, p.z)
        wallVertices.push(p.x, baseY, p.z)
      }

      for (let i = 0; i < wallCoords.length - 1; i++) {
        const topA = i * 2
        const botA = i * 2 + 1
        const topB = (i + 1) * 2
        const botB = (i + 1) * 2 + 1
        wallIndices.push(topA, botA, topB)
        wallIndices.push(topB, botA, botB)
      }

      const wallGeom = new THREE.BufferGeometry()
      wallGeom.setAttribute("position", new THREE.Float32BufferAttribute(wallVertices, 3))
      wallGeom.setIndex(wallIndices)
      wallGeom.computeVertexNormals()

      const wallMat = new THREE.ShaderMaterial({
        uniforms: {
          uBotY: { value: baseY }
        },
        vertexShader: `
          varying vec3 vNormal;
          varying float vY;
          varying vec3 vWorldPos;
          void main(){
            vNormal = normalize(normalMatrix * normal);
            vY = position.y;
            vec4 wp = modelMatrix * vec4(position, 1.0);
            vWorldPos = wp.xyz;
            gl_Position = projectionMatrix * viewMatrix * wp;
          }
        `,
        fragmentShader: `
          uniform float uBotY;
          varying vec3 vNormal;
          varying float vY;
          varying vec3 vWorldPos;

          void main(){
            // 估算顶部高度（侧壁通常在 0~200 之间）
            float estimatedTop = 100.0;
            float t = (vY - uBotY) / (estimatedTop - uBotY);
            t = clamp(t, 0.0, 1.0);
            
            vec3 cTop = vec3(0.95, 0.75, 0.35);
            vec3 cMid = vec3(0.55, 0.38, 0.15);
            vec3 cBot = vec3(0.15, 0.10, 0.05);
            
            vec3 color = t > 0.6
              ? mix(cMid, cTop, smoothstep(0.6, 1.0, t))
              : mix(cBot, cMid, smoothstep(0.0, 0.6, t));
            
            vec3 n = gl_FrontFacing ? vNormal : -vNormal;
            vec3 viewDir = normalize(cameraPosition - vWorldPos);
            
            vec3 light1 = normalize(vec3(0.6, 0.9, 0.8));
            float diff1 = max(dot(n, light1), 0.0);
            vec3 light2 = normalize(vec3(-0.6, 0.4, -0.6));
            float diff2 = max(dot(n, light2), 0.0);
            
            vec3 halfDir = normalize(light1 + viewDir);
            float spec = pow(max(dot(n, halfDir), 0.0), 48.0);
            
            float lighting = diff1 * 0.55 + diff2 * 0.30 + 0.45;
            color *= lighting;
            color += vec3(1.0, 0.85, 0.5) * spec * 0.4;

            gl_FragColor = vec4(color, 1.0);
          }
        `,
        side: THREE.DoubleSide
      })

      this.scene.add(new THREE.Mesh(wallGeom, wallMat))

      // ===== 底面 =====
      const shape = new THREE.Shape()
      coords.forEach((c, i) => {
        const p = this.getPos(c[0], c[1])
        if (i === 0) shape.moveTo(p.x, p.z)
        else shape.lineTo(p.x, p.z)
      })
      const botGeom = new THREE.ShapeGeometry(shape)
      botGeom.rotateX(Math.PI / 2)
      const botMat = new THREE.MeshBasicMaterial({ color: 0x0a0805 })
      const botMesh = new THREE.Mesh(botGeom, botMat)
      botMesh.position.y = baseY
      this.scene.add(botMesh)

      // ===== 金色边界（跟随地形高度） =====
      this.createGoldenBorder(coords)
    },

    createGoldenBorder(coords) {
      const { getH } = this
      const points = []

      coords.forEach(c => {
        const p = this.getPos(c[0], c[1])
        const h = getH(p.u, p.v)
        // 金边贴在该点地形顶部稍上方
        points.push(new THREE.Vector3(p.x, h + 3, p.z))
      })
      if (points.length > 0) points.push(points[0].clone())

      const curve = new THREE.CatmullRomCurve3(points, true, 'catmullrom', 0.1)
      const tubeGeom = new THREE.TubeGeometry(curve, points.length * 2, 48, 8, true)
      const tubeMat = new THREE.MeshStandardMaterial({
        color: 0xffc94a,
        metalness: 0.9,
        roughness: 0.2,
        emissive: 0xff8810,
        emissiveIntensity: 0.8,
        toneMapped: false
      })
      this.scene.add(new THREE.Mesh(tubeGeom, tubeMat))

      const coreGeom = new THREE.TubeGeometry(curve, points.length * 2, 8, 6, true)
      const coreMat = new THREE.MeshBasicMaterial({
        color: 0xffffee,
        toneMapped: false
      })
      this.scene.add(new THREE.Mesh(coreGeom, coreMat))
    },

    async loadRivers() {
      try {
        const res = await fetch("/anhui_river.geojson")
        const json = await res.json()

        const riverGroups = {}
        json.features.forEach(feature => {
          const geom = feature.geometry
          const name = feature.properties?.name || "未命名"
          if (!geom) return

          if (!riverGroups[name]) riverGroups[name] = []

          if (geom.type === "LineString") {
            riverGroups[name].push(geom.coordinates)
          } else if (geom.type === "MultiLineString") {
            geom.coordinates.forEach(line => {
              riverGroups[name].push(line)
            })
          }
        })

        console.log("河流分组:", Object.keys(riverGroups))
        this.createRivers(riverGroups)
      } catch (err) {
        console.error("加载河流失败:", err)
      }
    },

    createRivers(riverGroups) {
      const { getH } = this

      Object.keys(riverGroups).forEach(name => {
        const segments = riverGroups[name]
        const allSegmentPoints = []

        segments.forEach(coords => {
          if (coords.length < 2) return

          const points = []
          coords.forEach(c => {
            const p = this.getPos(c[0], c[1])
            if (p.u < 0 || p.u > 1 || p.v < 0 || p.v > 1) return
            const h = getH(p.u, p.v)
            const pt = new THREE.Vector3(p.x, h + 10, p.z)
            points.push(pt)
          })

          if (points.length < 2) return

          allSegmentPoints.push(points)

          const baseRadius = 6
          const radius = name === "淮河" ? baseRadius * 6 : baseRadius * 2

          const curve = new THREE.CatmullRomCurve3(points, false, 'catmullrom', 0.3)
          const tubeGeom = new THREE.TubeGeometry(
            curve,
            points.length * 3,
            radius,
            6,
            false
          )

          const tubeMat = new THREE.ShaderMaterial({
            uniforms: {
              uTime: { value: 0 },
              uColor: { value: new THREE.Color(0x00ddff) }
            },
            vertexShader: `
              varying vec2 vUv;
              void main(){
                vUv = uv;
                gl_Position = projectionMatrix * modelViewMatrix * vec4(position, 1.0);
              }
            `,
            fragmentShader: `
              uniform float uTime;
              uniform vec3 uColor;
              varying vec2 vUv;
              void main(){
                float flow = fract(vUv.x * 4.0 - uTime * 0.8);
                float highlight = smoothstep(0.7, 1.0, flow);
                vec3 color = uColor + vec3(1.0) * highlight * 0.8;
                float center = 1.0 - abs(vUv.y - 0.5) * 2.0;
                float alpha = 0.6 + center * 0.4;
                gl_FragColor = vec4(color, alpha);
              }
            `,
            transparent: true,
            toneMapped: false,
            side: THREE.DoubleSide,
            depthWrite: false
          })

          this.riverMaterials.push(tubeMat)
          this.scene.add(new THREE.Mesh(tubeGeom, tubeMat))
        })

        if (allSegmentPoints.length > 0 && name !== "未命名") {
          const bestPosition = this.findBestLabelPosition(allSegmentPoints)
          if (bestPosition) {
            this.createRiverLabel(name, bestPosition)
          }
        }
      })
    },

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
          return new THREE.Vector3().lerpVectors(
            longestSegment[i - 1],
            longestSegment[i],
            t
          )
        }
        accumulated += segLen
      }

      return longestSegment[Math.floor(longestSegment.length / 2)]
    },

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

    onResize() {
      const container = this.$refs.container
      const w = container.clientWidth
      const h = container.clientHeight
      this.camera.aspect = w / h
      this.camera.updateProjectionMatrix()
      this.renderer.setSize(w, h)
      this.composer.setSize(w, h)
    },

    animate() {
      requestAnimationFrame(() => this.animate())
      if (this.controls) this.controls.update()

      if (this.riverMaterials.length > 0) {
        this.riverMaterials.forEach(mat => {
          mat.uniforms.uTime.value += 0.016
        })
      }

      if (this.composer) this.composer.render()
    }
  }
}
</script>

<style scoped>
.map-container {
  width: 100%;
  height: 100vh;
  position: relative;
  background: #050e1c;
  overflow: hidden;
}
.loading {
  position: absolute;
  top: 50%;
  left: 50%;
  transform: translate(-50%, -50%);
  color: #ffd060;
  font-family: sans-serif;
  font-size: 14px;
}
</style>