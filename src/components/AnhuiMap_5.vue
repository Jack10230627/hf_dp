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
        maxH: 750,
        baseY: -400,
        hExp: 1.7,
        segments: 512,
        wallStep: 6,
        sink: 50
      },
      riverMaterials: []  // 河流材质引用（用于动画）
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
      this.scene.add(new THREE.AmbientLight(0x4488bb, 0.3))

      const sun = new THREE.DirectionalLight(0xfff0d0, 2.2)
      sun.position.set(10000, 18000, 8000)
      this.scene.add(sun)

      const fill = new THREE.DirectionalLight(0x66aaff, 0.9)
      fill.position.set(-10000, 6000, -5000)
      this.scene.add(fill)

      const rim = new THREE.DirectionalLight(0xffc060, 0.6)
      rim.position.set(0, 3000, -12000)
      this.scene.add(rim)

      // Bloom
      this.composer = new EffectComposer(this.renderer)
      this.composer.addPass(new RenderPass(this.scene, this.camera))
      const bloom = new UnrealBloomPass(
        new THREE.Vector2(container.clientWidth, container.clientHeight),
        0.8,
        0.7,
        0.7
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
        positions[i * 3]     = radius * Math.sin(phi) * Math.cos(theta)
        positions[i * 3 + 1] = radius * Math.cos(phi) * 0.3 + 20000
        positions[i * 3 + 2] = radius * Math.sin(phi) * Math.sin(theta)
        sizes[i] = Math.random() * 80 + 20
      }

      starGeom.setAttribute("position", new THREE.Float32BufferAttribute(positions, 3))
      starGeom.setAttribute("size", new THREE.Float32BufferAttribute(sizes, 1))

      const starMat = new THREE.ShaderMaterial({
        uniforms: {
          uColor: { value: new THREE.Color(0xaaccff) }
        },
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

      const stars = new THREE.Points(starGeom, starMat)
      this.scene.add(stars)
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

        this.buildMap(coords, demData, demW, demH, demMax)
        
        // 加载河流
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
      const u =
        (lon - this.bounds.minLon) /
        (this.bounds.maxLon - this.bounds.minLon)
      const v =
        (lat - this.bounds.minLat) /
        (this.bounds.maxLat - this.bounds.minLat)
      return {
        x: (u - 0.5) * this.config.width,
        z: -(v - 0.5) * this.config.height,
        u,
        v
      }
    },

    buildMap(coords, demData, demW, demH, demMax) {
      const { width, height, maxH, baseY, hExp, segments, wallStep, sink } = this.config

      // 3x3 高斯采样
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
        if (avg <= 0) return -sink
        return Math.pow(avg / demMax, hExp) * maxH - sink
      }

      this.getH = getH  // 保存给河流用

      // ========== 1. 顶面地形 ==========
      const topGeom = new THREE.PlaneGeometry(width, height, segments, segments)
      topGeom.rotateX(-Math.PI / 2)

      const posAttr = topGeom.attributes.position
      const uvAttr = topGeom.attributes.uv
      for (let i = 0; i < posAttr.count; i++) {
        posAttr.setY(i, getH(uvAttr.getX(i), uvAttr.getY(i)))
      }

      // 拉普拉斯平滑（2 次迭代）
      const segs = segments + 1
      for (let iter = 0; iter < 2; iter++) {
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
          uMaxH: { value: maxH },
          uSink: { value: sink }
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
          uniform float uSink;
          varying vec2 vUv;
          varying float vH;
          varying vec3 vNormal;
          varying vec3 vWorldPos;

          void main(){
            if(texture2D(uMask, vUv).r < 0.5) discard;
            float h = clamp((vH + uSink) / uMaxH, 0.0, 1.0);

            vec3 c0 = vec3(0.043, 0.125, 0.275);
            vec3 c1 = vec3(0.071, 0.192, 0.384);
            vec3 c2 = vec3(0.000, 0.706, 0.847);
            vec3 c3 = vec3(0.180, 0.545, 0.341);
            vec3 c4 = vec3(0.298, 0.686, 0.314);
            vec3 c5 = vec3(0.741, 0.718, 0.420);
            vec3 c6 = vec3(0.831, 0.627, 0.090);
            vec3 c7 = vec3(0.545, 0.271, 0.075);
            vec3 c8 = vec3(0.427, 0.298, 0.255);

            vec3 color;
            if(h < 0.08)      color = mix(c0, c1, h / 0.08);
            else if(h < 0.15) color = mix(c1, c2, (h - 0.08) / 0.07);
            else if(h < 0.30) color = mix(c2, c3, (h - 0.15) / 0.15);
            else if(h < 0.50) color = mix(c3, c4, (h - 0.30) / 0.20);
            else if(h < 0.65) color = mix(c4, c5, (h - 0.50) / 0.15);
            else if(h < 0.80) color = mix(c5, c6, (h - 0.65) / 0.15);
            else if(h < 0.92) color = mix(c6, c7, (h - 0.80) / 0.12);
            else              color = mix(c7, c8, (h - 0.92) / 0.08);

            vec3 lightDir = normalize(vec3(0.6, 1.0, 0.4));
            float diff = max(dot(vNormal, lightDir), 0.0);
            vec3 fillDir = normalize(vec3(-0.5, 0.5, -0.5));
            float diffFill = max(dot(vNormal, fillDir), 0.0) * 0.3;
            float lighting = diff * 0.85 + diffFill + 0.35;
            color *= lighting;

            if(h < 0.25){
              float glow = (1.0 - h / 0.25) * 0.15;
              color += vec3(0.0, 0.5, 0.7) * glow;
            }

            gl_FragColor = vec4(color, 1.0);
          }
        `
      })
      this.scene.add(new THREE.Mesh(topGeom, topMat))

      // ========== 2. 侧墙（共享顶点光滑版） ==========
      const wallTopY = 0
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
        wallVertices.push(p.x, wallTopY, p.z)
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
          uTopY: { value: wallTopY },
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
          uniform float uTopY;
          uniform float uBotY;
          varying vec3 vNormal;
          varying float vY;
          varying vec3 vWorldPos;

          void main(){
            float t = (vY - uBotY) / (uTopY - uBotY);
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
            
            float topGlow = smoothstep(0.90, 1.0, t) * 0.8;
            color += vec3(1.0, 0.85, 0.4) * topGlow;

            gl_FragColor = vec4(color, 1.0);
          }
        `,
        side: THREE.DoubleSide
      })

      this.scene.add(new THREE.Mesh(wallGeom, wallMat))

      // ========== 3. 底面 ==========
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

      // ========== 4. 金色边界 ==========
      this.createGoldenBorder(coords)
    },

    createGoldenBorder(coords) {
      const points = []
      const borderY = 3
      coords.forEach(c => {
        const p = this.getPos(c[0], c[1])
        points.push(new THREE.Vector3(p.x, borderY, p.z))
      })
      if (points.length > 0) points.push(points[0].clone())

      const curve = new THREE.CatmullRomCurve3(points, true, 'catmullrom', 0.1)
      const tubeGeom = new THREE.TubeGeometry(curve, points.length * 2, 22, 8, true)
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

    // ========== 河流加载 ==========
    async loadRivers() {
      try {
        const res = await fetch("/anhui_river.geojson")
        const json = await res.json()

        const rivers = []
        json.features.forEach(feature => {
          const geom = feature.geometry
          if (!geom) return
          if (geom.type === "LineString") {
            rivers.push(geom.coordinates)
          } else if (geom.type === "MultiLineString") {
            geom.coordinates.forEach(line => rivers.push(line))
          }
        })

        console.log("河流数量:", rivers.length)
        this.createRivers(rivers)
      } catch (err) {
        console.error("加载河流失败:", err)
      }
    },

     createRivers(rivers) {
      const { getH } = this

      rivers.forEach(coords => {
        if (coords.length < 2) return

        // 转换坐标并过滤边界外的点
        const points = []
        coords.forEach(c => {
          const p = this.getPos(c[0], c[1])
          if (p.u < 0 || p.u > 1 || p.v < 0 || p.v > 1) return
          const h = getH(p.u, p.v)
          points.push(new THREE.Vector3(p.x, h + 10, p.z))
        })

        if (points.length < 2) return

        // 平滑曲线
        const curve = new THREE.CatmullRomCurve3(points, false, 'catmullrom', 0.3)

        // 河流管道
        const tubeGeom = new THREE.TubeGeometry(
          curve,
          points.length * 3,
          10,    // 河流粗细
          6,
          false
        )

        // 流光 Shader 材质
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
              // 沿管道长度方向流动
              float flow = fract(vUv.x * 4.0 - uTime * 0.8);
              float highlight = smoothstep(0.7, 1.0, flow);
              
              vec3 color = uColor + vec3(1.0, 1.0, 1.0) * highlight * 0.8;
              
              // 中心更亮
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
      
      // 更新河流流光动画
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