<template>
  <div ref="container" class="map-container"></div>
</template>

<script>
import * as THREE from "three"
import { OrbitControls } from "three/examples/jsm/controls/OrbitControls"
import { fromUrl } from "geotiff"

export default {
  name: "AnhuiMap",

  mounted() {
    this.initThree()
    this.animate()
    window.addEventListener("resize", this.onResize)
  },

  beforeUnmount() {
    window.removeEventListener("resize", this.onResize)
  },

  methods: {

    async initThree() {

      const container = this.$refs.container

      this.scene = new THREE.Scene()
      this.scene.background = new THREE.Color(0x04121c)

      this.camera = new THREE.PerspectiveCamera(
        50,
        container.clientWidth / container.clientHeight,
        1,
        60000
      );

      this.camera.position.set(6000, 3500, 9000);
      this.camera.lookAt(0, 700, 0);



      this.renderer = new THREE.WebGLRenderer({ antialias: true })
      this.renderer.setSize(container.clientWidth, container.clientHeight)
      this.renderer.setPixelRatio(window.devicePixelRatio)
      container.appendChild(this.renderer.domElement)

      this.controls = new OrbitControls(this.camera, this.renderer.domElement)
      this.controls.enableDamping = true

      // ✅ 光源
      const dirLight = new THREE.DirectionalLight(0xffffff, 1.2)
      dirLight.position.set(5000, 8000, 5000)
      this.scene.add(dirLight)

      const ambient = new THREE.AmbientLight(0xffffff, 0.4)
      this.scene.add(ambient)

      const hemi = new THREE.HemisphereLight(0xffffff, 0x222222, 0.6);
      this.scene.add(hemi);
      this.scene.fog = new THREE.Fog(0x04121c, 8000, 20000);
      
      await this.createTerrain()
      this.createGoldBase()
      this.createBorder()
    },

    async createTerrain() {

      const width = 20000
      const height = 20000
      const segments = 1023
      const maxHeight = 550
      const baseHeight = 700

      const geometry = new THREE.PlaneGeometry(width, height, segments, segments)
      geometry.rotateX(-Math.PI / 2)

      const material = new THREE.ShaderMaterial({
        uniforms: {
          uMaxHeight: { value: maxHeight }
        },
        polygonOffset: true,
        polygonOffsetFactor: -1,
        polygonOffsetUnits: -1,
        vertexShader: `
          varying float vHeight;
          varying vec3 vNormal;

          void main(){
            vHeight = position.y;
            vNormal = normalize(normalMatrix * normal);
            gl_Position = projectionMatrix * modelViewMatrix * vec4(position,1.0);
          }
        `,
        fragmentShader: `
          uniform float uMaxHeight;
          varying float vHeight;
          varying vec3 vNormal;

          void main(){

            float h = clamp(vHeight / uMaxHeight, 0.0, 1.0);
            h = pow(h, 0.6);

          vec3 deepBlue = vec3(0.02, 0.18, 0.45);
vec3 blueCyan = vec3(0.05, 0.65, 0.85);
vec3 greenMid = vec3(0.35, 0.85, 0.50);
vec3 brownHigh = vec3(0.85, 0.65, 0.25);

            vec3 color;

            if(h < 0.30){
                color = mix(deepBlue, blueCyan, h / 0.30);
            } else if(h < 0.60){
                color = mix(blueCyan, greenMid, (h - 0.30) / 0.30);
            } else {
                color = mix(greenMid, brownHigh, (h - 0.60) / 0.40);
            }

            float slope = 1.0 - abs(vNormal.y);
            float ridge = pow(slope, 1.5);
            color += ridge * 0.25;

            vec3 lightDir = normalize(vec3(0.35,1.0,0.25));
            float light = dot(normalize(vNormal), lightDir);
            light = clamp(light,0.2,1.0);

            color *= light;
            color *= 0.95;

            gl_FragColor = vec4(color,1.0);
          }
        `
      })

      const mesh = new THREE.Mesh(geometry, material)
      mesh.position.y = baseHeight + 20   // ✅ 抬高避免冲突
      this.scene.add(mesh)

      // ✅ 读取 Float DEM
      const tiff = await fromUrl("/anhui_height_float.tif")
      const image = await tiff.getImage()
      const raster = await image.readRasters()

      const demData = raster[0]
      const demWidth = image.getWidth()
      const demHeight = image.getHeight()

      const vertices = geometry.attributes.position
      const uvs = geometry.attributes.uv

      let demMax = -Infinity
      for (let i = 0; i < demData.length; i++) {
        if (demData[i] > demMax) demMax = demData[i]
      }

      for (let i = 0; i < vertices.count; i++) {

        const u = uvs.getX(i)
        const v = uvs.getY(i)

        const x = Math.floor(u * demWidth)
        const y = Math.floor((1 - v) * demHeight)

        const index = y * demWidth + x

        const heightValue = Math.pow(demData[index] / demMax, 0.9)
        vertices.setY(i, heightValue * maxHeight)
      }

      vertices.needsUpdate = true
      geometry.computeVertexNormals()
    },

    createGoldBase() {

      fetch("/anhui_border.geojson")
        .then(res => res.json())
        .then(dataGeo => {

          const geometryData = dataGeo.features[0].geometry
          let rings = []

          if (geometryData.type === "Polygon") {
            rings = geometryData.coordinates
          } else {
            geometryData.coordinates.forEach(p => rings.push(...p))
          }

          let outerRing = rings.reduce((a, b) => b.length > a.length ? b : a)

          const minLon = 114.87625
          const maxLon = 119.64514
          const minLat = 29.39542
          const maxLat = 34.65236

          const width = 20000
          const height = 20000

          const shape = new THREE.Shape()

          outerRing.forEach((coord, index) => {

            const lon = coord[0]
            const lat = coord[1]

            const u = (lon - minLon) / (maxLon - minLon)
            const v = (lat - minLat) / (maxLat - minLat)

            const x = (u - 0.5) * width
            const z = -(v - 0.5) * height

            if (index === 0) shape.moveTo(x, z)
            else shape.lineTo(x, z)
          })

          const geometry = new THREE.ExtrudeGeometry(shape, {
            depth: 700,
            bevelEnabled: false
          })

          geometry.rotateX(-Math.PI / 2)

          const material = new THREE.MeshStandardMaterial({
            color: 0xcfa75a,
            metalness: 0.9,
            roughness: 0.15,
            envMapIntensity: 1.2
          });
          const mesh = new THREE.Mesh(geometry, material)
          mesh.position.y = 0
          this.scene.add(mesh)
        })
    },

    createBorder() {

      fetch("/anhui_border.geojson")
        .then(res => res.json())
        .then(dataGeo => {

          const geometryData = dataGeo.features[0].geometry
          let rings = []

          if (geometryData.type === "Polygon") {
            rings = geometryData.coordinates
          } else {
            geometryData.coordinates.forEach(p => rings.push(...p))
          }

          let outerRing = rings.reduce((a, b) => b.length > a.length ? b : a)

          const minLon = 114.87625
          const maxLon = 119.64514
          const minLat = 29.39542
          const maxLat = 34.65236

          const width = 20000
          const height = 20000

          const points = []

          outerRing.forEach(coord => {

            const lon = coord[0]
            const lat = coord[1]

            const u = (lon - minLon) / (maxLon - minLon)
            const v = (lat - minLat) / (maxLat - minLat)

            const x = (u - 0.5) * width
            const z = -(v - 0.5) * height

            points.push(new THREE.Vector3(x, 720, z))
          })

          const geometry = new THREE.BufferGeometry().setFromPoints(points)

          const mat = new THREE.LineBasicMaterial({
            color: 0xfff2c0,
            depthTest: false
          })

          const line = new THREE.LineLoop(geometry, mat)
          this.scene.add(line)
        })
    },

    animate() {
      requestAnimationFrame(() => this.animate())
      this.controls.update()
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

<style>
.map-container {
  width: 100%;
  height: 100vh;
  background: #04121c;
}
</style>