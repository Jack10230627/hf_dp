<template>
  <div ref="container" class="map-container"></div>
</template>

<script>
import * as THREE from "three"
import { OrbitControls } from "three/examples/jsm/controls/OrbitControls"
import { fromUrl } from "geotiff"

export default {
  name: "AnhuiSolidMap",

  data() {
    return {
      // 这里的范围需与你的 TIFF 和 GeoJSON 对应
      mapBounds: {
        minLon: 114.87625,
        maxLon: 119.64514,
        minLat: 29.39542,
        maxLat: 34.65236,
        width: 20000,
        height: 20000,
        maxElevation: 600, // 山峰最高抬升
        baseDepth: -800,   // 底座厚度（负值）
      }
    }
  },

  mounted() {
    this.initThree()
    this.animate()
    window.addEventListener("resize", this.onResize)
  },

  methods: {
    async initThree() {
      const container = this.$refs.container
      this.scene = new THREE.Scene()
      this.scene.background = new THREE.Color(0x04121c)

      this.camera = new THREE.PerspectiveCamera(45, container.clientWidth / container.clientHeight, 10, 100000)
      this.camera.position.set(0, 8000, 12000)

      this.renderer = new THREE.WebGLRenderer({ antialias: true })
      this.renderer.setSize(container.clientWidth, container.clientHeight)
      this.renderer.setPixelRatio(window.devicePixelRatio)
      this.renderer.toneMapping = THREE.ACESFilmicToneMapping
      container.appendChild(this.renderer.domElement)

      this.controls = new OrbitControls(this.camera, this.renderer.domElement)
      this.controls.enableDamping = true

      // 光照
      const ambient = new THREE.AmbientLight(0xffffff, 1.5)
      this.scene.add(ambient)
      const dirLight = new THREE.DirectionalLight(0xffffff, 3)
      dirLight.position.set(5000, 10000, 5000)
      this.scene.add(dirLight)

      await this.loadAndCreateMap()
    },

    async loadAndCreateMap() {
      // 1. 读取 DEM 数据
      const tiff = await fromUrl("/anhui_height_float.tif")
      const image = await tiff.getImage()
      const raster = await image.readRasters()
      const demData = raster[0]
      const demW = image.getWidth()
      const demH = image.getHeight()

      // 找出 DEM 最大值用于归一化
      let demMax = 0
      for (let i = 0; i < demData.length; i++) if (demData[i] > demMax) demMax = demData[i]

      // 采样函数：根据经纬度/坐标获取高度
      const getHeightAt = (u, v) => {
        const x = Math.floor(u * (demW - 1))
        const y = Math.floor((1 - v) * (demH - 1))
        const idx = y * demW + x
        const val = demData[idx] || 0
        return Math.pow(val / demMax, 0.9) * this.mapBounds.maxElevation
      }

      // 2. 加载边界 GeoJSON
      const geoRes = await fetch("/anhui_border.geojson")
      const geoData = await geoRes.json()
      const coords = this.getOuterRing(geoData)

      // 3. 创建顶面（地形）
      this.createTerrainTop(getHeightAt)

      // 4. 创建侧面裙部（关键：填充空隙）
      this.createSideWalls(coords, getHeightAt)

      // 5. 创建底盖
      this.createBottomCap(coords)
    },

    // 提取外边界坐标
    getOuterRing(geoData) {
      let rings = []
      const geometry = geoData.features[0].geometry
      if (geometry.type === "Polygon") rings = geometry.coordinates
      else geometry.coordinates.forEach(p => rings.push(...p))
      return rings.reduce((a, b) => b.length > a.length ? b : a)
    },

    // 经纬度转场景坐标
    lonLatToXY(lon, lat) {
      const { minLon, maxLon, minLat, maxLat, width, height } = this.mapBounds
      const u = (lon - minLon) / (maxLon - minLon)
      const v = (lat - minLat) / (maxLat - minLat)
      const x = (u - 0.5) * width
      const z = -(v - 0.5) * height
      return { x, z, u, v }
    },

    createTerrainTop(getHeightAt) {
      const { width, height, maxElevation } = this.mapBounds
      const geometry = new THREE.PlaneGeometry(width, height, 512, 512)
      geometry.rotateX(-Math.PI / 2)

      const vertices = geometry.attributes.position
      const uvs = geometry.attributes.uv

      for (let i = 0; i < vertices.count; i++) {
        const u = uvs.getX(i)
        const v = uvs.getY(i)
        const h = getHeightAt(u, v)
        vertices.setY(i, h)
      }
      geometry.computeVertexNormals()

      const material = new THREE.ShaderMaterial({
        uniforms: { uMaxH: { value: maxElevation } },
        vertexShader: `
          varying float vHeight;
          void main() {
            vHeight = position.y;
            gl_Position = projectionMatrix * modelViewMatrix * vec4(position, 1.0);
          }
        `,
        fragmentShader: `
          uniform float uMaxH;
          varying float vHeight;
          void main() {
            float h = clamp(vHeight / uMaxH, 0.0, 1.0);
            vec3 low = vec3(0.02, 0.15, 0.4);
            vec3 mid = vec3(0.1, 0.6, 0.5);
            vec3 high = vec3(0.8, 0.7, 0.3);
            vec3 color = h < 0.5 ? mix(low, mid, h*2.0) : mix(mid, high, (h-0.5)*2.0);
            gl_FragColor = vec4(color * 1.2, 1.0);
          }
        `,
        side: THREE.DoubleSide
      })

      const mesh = new THREE.Mesh(geometry, material)
      // 使用 Stencil 或简单的 Clipping 剔除边界外，此处为简化逻辑暂不剔除，
      // 如果你的 TIFF 已经裁剪过，边缘就是 0。
      this.scene.add(mesh)
    },

    createSideWalls(coords, getHeightAt) {
      const points = []
      const { baseDepth } = this.mapBounds

      for (let i = 0; i < coords.length - 1; i++) {
        const p1 = this.lonLatToXY(coords[i][0], coords[i][1])
        const p2 = this.lonLatToXY(coords[i+1][0], coords[i+1][1])

        const h1 = getHeightAt(p1.u, p1.v)
        const h2 = getHeightAt(p2.u, p2.v)

        // 侧面由两个三角形组成的矩形面片
        // 顶点顺序：上1, 下1, 上2, 下1, 下2, 上2
        points.push(new THREE.Vector3(p1.x, h1, p1.z))
        points.push(new THREE.Vector3(p1.x, baseDepth, p1.z))
        points.push(new THREE.Vector3(p2.x, h2, p2.z))

        points.push(new THREE.Vector3(p1.x, baseDepth, p1.z))
        points.push(new THREE.Vector3(p2.x, baseDepth, p2.z))
        points.push(new THREE.Vector3(p2.x, h2, p2.z))
      }

      const geometry = new THREE.BufferGeometry().setFromPoints(points)
      geometry.computeVertexNormals()

      const material = new THREE.MeshStandardMaterial({
        color: 0xcfa75a, // 金色侧边
        metalness: 0.8,
        roughness: 0.2,
        side: THREE.DoubleSide
      })

      const wallMesh = new THREE.Mesh(geometry, material)
      this.scene.add(wallMesh)
    },

    createBottomCap(coords) {
      const shape = new THREE.Shape()
      coords.forEach((c, i) => {
        const p = this.lonLatToXY(c[0], c[1])
        if (i === 0) shape.moveTo(p.x, p.z)
        else shape.lineTo(p.x, p.z)
      })

      const geometry = new THREE.ShapeGeometry(shape)
      geometry.rotateX(Math.PI / 2) // 放到水平面
      
      const material = new THREE.MeshStandardMaterial({ color: 0x111111 })
      const mesh = new THREE.Mesh(geometry, material)
      mesh.position.y = this.mapBounds.baseDepth // 置于底部
      this.scene.add(mesh)
      
      // 添加底边框线
      const lineGeom = new THREE.BufferGeometry().setFromPoints(
        coords.map(c => {
          const p = this.lonLatToXY(c[0], c[1])
          return new THREE.Vector3(p.x, this.mapBounds.baseDepth, p.z)
        })
      )
      const line = new THREE.Line(lineGeom, new THREE.LineBasicMaterial({ color: 0xfff2c0 }))
      this.scene.add(line)
    },

    animate() {
      requestAnimationFrame(() => this.animate())
      this.controls && this.controls.update()
      this.renderer.render(this.scene, this.camera)
    },

    onResize() {
      const container = this.$refs.container
      this.camera.aspect = container.clientWidth / container.clientHeight
      this.camera.updateProjectionMatrix()
      this.renderer.setSize(container.clientWidth, container.clientHeight)
    }
  }
}
</script>

<style scoped>
.map-container { width: 100%; height: 100vh; }
</style>