<template>
  <div class="page-wrapper">
    <CyberpunkBackground class="hud-layer" />
    <div ref="container" class="map-container">
      <div v-if="loading" class="loading">正在加载安徽水利高清沙盘...</div>

      <div class="region-switcher">
        <button
          :class="{ active: currentRegion === 'all' }"
          @click="switchRegion('all')"
        >
          全部
        </button>
        <button
          :class="{ active: currentRegion === 'anhui' }"
          @click="switchRegion('anhui')"
        >
          安徽
        </button>
        <button
          :class="{ active: currentRegion === 'huaihe' }"
          @click="switchRegion('huaihe')"
        >
          淮河流域
        </button>
      </div>

      <div v-if="selectedRiverName" class="river-info">
        <div class="river-info-title">已选中河流</div>
        <div class="river-info-name">{{ selectedRiverName }}</div>
        <div class="river-info-desc">
          点击空白处可取消；再次点击其他河流可切换。
        </div>
      </div>
    </div>

    <header class="dashboard-header">
      <h1 class="main-title">数字孪生安徽指挥大屏</h1>
      <div class="title-decoration">
        <span class="line left"></span>
        <span class="glow-dot"></span>
        <span class="line right"></span>
      </div>
    </header>

    <footer class="dashboard-footer">
      <BottomNav />
    </footer>
  </div>
</template>

<script>
import * as THREE from "three";
import { OrbitControls } from "three/examples/jsm/controls/OrbitControls";
import { EffectComposer } from "three/examples/jsm/postprocessing/EffectComposer";
import { RenderPass } from "three/examples/jsm/postprocessing/RenderPass";
import { UnrealBloomPass } from "three/examples/jsm/postprocessing/UnrealBloomPass";
import { fromUrl, fromArrayBuffer } from "geotiff";
import BottomNav from "./BottomNav.vue";
import CyberpunkBackground from "./CyberpunkBackground.vue";

export default {
  name: "AnhuiHuaiheMap",
  components: { CyberpunkBackground, BottomNav },
  data() {
    return {
      loading: true,
      selectedRiverName: "",
      currentRegion: "anhui",
    };
  },
  created() {
    this.scene = null;
    this.camera = null;
    this.renderer = null;
    this.composer = null;
    this.controls = null;
    this.mapGroup = null;
    this.stars = null;
    this.riverMaterials = [];

    this.labelSprites = [];
    this.borderMat = null;
    this.topMat = null;

    this.riverMeshes = [];
    this.selectedRiverMat = null;
    this.raycaster = null;
    this.pointer = new THREE.Vector2();
    this._riverBaseColorMap = new Map();
    this.pointerDown = { x: 0, y: 0 };
    this.pointerMoved = false;

    this.config = {
      width: 20000,
      height: 20000,
      maxH: 900, // ✨ 回退到旧版
      baseY: -400, // ✨ 回退到旧版
      hExp: 1.4, // ✨ 回退到旧版
      segments: 512,
      borderWidth: 30,
      riverBaseWidth: 8,
    };

    // ✨ 区域配置：中心 + 摄像机视角
    this.regionConfig = {
      all: {
        // 全部视图（包含整个并集）
        target: { x: 0, y: 0, z: 0 }, // 看向地图中心
        cameraOffset: { x: 0, y: 25000, z: 30000 }, // 摄像机相对目标位置
      },
      anhui: {
        // 安徽中心：约 117.3°E, 31.8°N
        centerLonLat: [117.3, 31.8],
        cameraDistance: 18000, // 摄像机距离
        cameraHeight: 15000, // 摄像机高度
        cameraAngle: 0, // 旋转角度（0 = 正南方向看）
      },
      huaihe: {
        // 淮河流域中心：约 116.5°E, 33.5°N
        centerLonLat: [116.5, 33.5],
        cameraDistance: 22000,
        cameraHeight: 18000,
        cameraAngle: 0,
      },
    };

    // ✨ 摄像机动画相关
    this.cameraAnim = {
      active: false,
      startTime: 0,
      duration: 1500, // 动画时长 ms
      fromPos: new THREE.Vector3(),
      toPos: new THREE.Vector3(),
      fromTarget: new THREE.Vector3(),
      toTarget: new THREE.Vector3(),
    };

    // 默认区域
    this.defaultRegion = "anhui";
    this.labelConfig = {
      fontSize: 48,
      color: "#00ddff",
      color2: "#00ddff",
      strokeColor: "#003344",
      strokeWidth: 6,
      bgColor: "rgba(0, 30, 50, 0.7)",
      padding: 12,
      scale: 5,
      heightOffset: 200,
    };
  },
  mounted() {
    this.$nextTick(() => {
      this.initThree();
      this.animate();
      window.addEventListener("resize", this.onResize);
    });
  },
  beforeDestroy() {
    window.removeEventListener("resize", this.onResize);
    const dom = this.renderer?.domElement;
    if (dom) {
      if (this._onPointerDown)
        dom.removeEventListener("pointerdown", this._onPointerDown);
      if (this._onPointerMove)
        dom.removeEventListener("pointermove", this._onPointerMove);
      if (this._onPointerUp)
        dom.removeEventListener("pointerup", this._onPointerUp);
    }
  },
  methods: {
    initThree() {
      const container = this.$refs.container;
      if (!container) return;

      this.scene = new THREE.Scene();
      this.camera = new THREE.PerspectiveCamera(
        35,
        container.clientWidth / container.clientHeight,
        1,
        1000000,
      );
      this.camera.position.set(0, 25000, 30000);

      this.renderer = new THREE.WebGLRenderer({
        antialias: true,
        alpha: true,
        logarithmicDepthBuffer: true,
      });
      this.renderer.setClearColor(0x000000, 0);
      this.renderer.setSize(container.clientWidth, container.clientHeight);
      this.renderer.setPixelRatio(Math.min(window.devicePixelRatio, 2));
      this.renderer.toneMapping = THREE.ACESFilmicToneMapping;
      this.renderer.toneMappingExposure = 1.3;
      container.appendChild(this.renderer.domElement);

      this.controls = new OrbitControls(this.camera, this.renderer.domElement);
      this.controls.enableDamping = true;

      this.mapGroup = new THREE.Group();
      this.scene.add(this.mapGroup);

      this.scene.add(new THREE.AmbientLight(0x4466aa, 0.7));
      const sun = new THREE.DirectionalLight(0xffffff, 2.2);
      sun.position.set(10000, 20000, 10000);
      this.scene.add(sun);

      this.initSpaceBackground();

      this.raycaster = new THREE.Raycaster();
      this._onPointerDown = (e) => {
        if (e.button === 2) return;
        this.pointerMoved = false;
        this.pointerDown.x = e.clientX;
        this.pointerDown.y = e.clientY;
      };
      this._onPointerMove = (e) => {
        const dx = e.clientX - this.pointerDown.x;
        const dy = e.clientY - this.pointerDown.y;
        if (Math.sqrt(dx * dx + dy * dy) > 6) this.pointerMoved = true;
      };
      this._onPointerUp = (e) => {
        if (this.pointerMoved) return;
        this.handleRiverClick(e);
      };
      this.renderer.domElement.addEventListener(
        "pointerdown",
        this._onPointerDown,
      );
      this.renderer.domElement.addEventListener(
        "pointermove",
        this._onPointerMove,
      );
      this.renderer.domElement.addEventListener("pointerup", this._onPointerUp);

      this.composer = new EffectComposer(this.renderer);
      this.composer.addPass(new RenderPass(this.scene, this.camera));
      const bloom = new UnrealBloomPass(
        new THREE.Vector2(container.clientWidth, container.clientHeight),
        0.25,
        0.4,
        0.9,
      );
      this.composer.addPass(bloom);

      this.loadData();
    },

    initSpaceBackground() {
      const starGeometry = new THREE.BufferGeometry();
      const starMaterial = new THREE.PointsMaterial({
        color: 0xffffff,
        size: 15,
        transparent: true,
        opacity: 0.8,
        sizeAttenuation: true,
      });
      const starVertices = [];
      for (let i = 0; i < 8000; i++) {
        const r = 200000 + Math.random() * 300000;
        const theta = Math.random() * Math.PI * 2;
        const phi = Math.acos(2 * Math.random() - 1);
        starVertices.push(
          r * Math.sin(phi) * Math.cos(theta),
          r * Math.sin(phi) * Math.sin(theta),
          r * Math.cos(phi),
        );
      }
      starGeometry.setAttribute(
        "position",
        new THREE.Float32BufferAttribute(starVertices, 3),
      );
      this.stars = new THREE.Points(starGeometry, starMaterial);
      this.scene.add(this.stars);
    },

    async loadData() {
      try {
        const [anhuiRes, huaiheRes, unionRes, tiffRes, riverRes] =
          await Promise.all([
            fetch("/anhui_border.geojson").then((r) => r.json()),
            fetch("/huaihe_border.geojson").then((r) => r.json()),
            fetch("/union_border.geojson").then((r) => r.json()),
            fetch("/union_web.tif")
              .then((r) => r.arrayBuffer())
              .then((buf) => fromArrayBuffer(buf)),
            fetch("/union_river_filtered_byName.geojson").then((r) => r.json()),
          ]);

        this.anhuiGeoJson = anhuiRes;
        this.huaiheGeoJson = huaiheRes;
        this.unionGeoJson = unionRes;

        const unionCoords = this.extractCoords(unionRes);
        this.calculateBounds(unionCoords);

        const lonRange = this.bounds.maxLon - this.bounds.minLon;
        const latRange = this.bounds.maxLat - this.bounds.minLat;
        const aspect = lonRange / latRange;
        this.config.width = 20000 * aspect;
        this.config.height = 20000;

        const scaleRatio = this.config.width / 20000;
        this.config.borderWidth *= scaleRatio;
        this.config.riverBaseWidth *= scaleRatio;
        this.labelConfig.heightOffset *= scaleRatio;

        console.log("📐 并集 bounds:", this.bounds);
        console.log("📐 aspect:", aspect, "width:", this.config.width);

        const image = await tiffRes.getImage();
        const rasters = await image.readRasters();

        this.demData =
          Array.isArray(rasters) && rasters[0]?.length ? rasters : [rasters];
        this.demW = image.getWidth();
        this.demH = image.getHeight();

        const demBbox = image.getBoundingBox();
        this.demBounds = {
          minLon: demBbox[0],
          minLat: demBbox[1],
          maxLon: demBbox[2],
          maxLat: demBbox[3],
        };

        let demMax = 0;
        let validCount = 0;
        const arr = this.demData[0];
        for (let i = 0; i < arr.length; i++) {
          const v = arr[i];
          if (!isNaN(v) && v > 0 && v < 10000) {
            if (v > demMax) demMax = v;
            validCount++;
          }
        }
        console.log(
          "📐 demMax:",
          demMax,
          "有效像素:",
          validCount,
          "/",
          arr.length,
        );

        if (demMax === 0) {
          console.warn("⚠️ demMax 为 0，使用默认值 1500");
          demMax = 1500;
        }

        this.buildMap(unionCoords, demMax);
        this.createRivers(riverRes, demMax);
        // ✨ 默认切到安徽（不带动画，瞬间到位）
        this.$nextTick(() => {
          this.switchRegion(this.defaultRegion, false);
        });

        this.loading = false;
      } catch (err) {
        console.error("加载失败:", err);
      }
    },
    // 经纬度转世界坐标
    lonLatToWorld(lon, lat) {
      const u =
        (lon - this.bounds.minLon) / (this.bounds.maxLon - this.bounds.minLon);
      const v =
        (lat - this.bounds.minLat) / (this.bounds.maxLat - this.bounds.minLat);
      return new THREE.Vector3(
        (u - 0.5) * this.config.width,
        0,
        -(v - 0.5) * this.config.height,
      );
    },
    calculateBounds(coords) {
      const lons = coords.map((c) => c[0]);
      const lats = coords.map((c) => c[1]);
      this.bounds = {
        minLon: Math.min(...lons),
        maxLon: Math.max(...lons),
        minLat: Math.min(...lats),
        maxLat: Math.max(...lats),
      };
    },

    getH(u, v, demMax) {
      const lon =
        this.bounds.minLon + u * (this.bounds.maxLon - this.bounds.minLon);
      const lat =
        this.bounds.minLat + v * (this.bounds.maxLat - this.bounds.minLat);

      const demU =
        (lon - this.demBounds.minLon) /
        (this.demBounds.maxLon - this.demBounds.minLon);
      const demV =
        (lat - this.demBounds.minLat) /
        (this.demBounds.maxLat - this.demBounds.minLat);

      if (demU < 0 || demU > 1 || demV < 0 || demV > 1) return 0;

      const cx = demU * (this.demW - 1);
      const cy = (1 - demV) * (this.demH - 1);
      const data = this.demData[0];
      const getV = (x, y) => {
        x = Math.max(0, Math.min(this.demW - 1, x));
        y = Math.max(0, Math.min(this.demH - 1, y));
        const vv = data[Math.floor(y) * this.demW + Math.floor(x)];
        return isNaN(vv) || vv < 0 ? 0 : vv;
      };
      return (
        Math.pow(getV(cx, cy) / demMax, this.config.hExp) * this.config.maxH
      );
    },

    getOriginalH(u, v, demMax) {
      return this.getH(u, v, demMax);
    },

    // ✨ 回退到旧版 shader（你之前效果好的版本）+ 加区域明暗
    buildMap(coords, demMax) {
      const topGeom = new THREE.PlaneGeometry(
        this.config.width,
        this.config.height,
        this.config.segments,
        this.config.segments,
      );
      topGeom.rotateX(-Math.PI / 2);
      const pos = topGeom.attributes.position;
      const uv = topGeom.attributes.uv;
      for (let i = 0; i < pos.count; i++) {
        pos.setY(i, this.getH(uv.getX(i), uv.getY(i), demMax));
      }
      topGeom.computeVertexNormals();

      this.topMat = new THREE.ShaderMaterial({
        uniforms: {
          uUnionMask: { value: this.createMaskFromGeoJson(this.unionGeoJson) },
          uAnhuiMask: { value: this.createMaskFromGeoJson(this.anhuiGeoJson) },
          uHuaiheMask: {
            value: this.createMaskFromGeoJson(this.huaiheGeoJson),
          },
          uActiveRegion: { value: 0 },
          uDarkness: { value: 0.22 },
          uMaxH: { value: this.config.maxH },
        },
        vertexShader: `
          varying vec2 vUv;
          varying float vH;
          varying vec3 vNormal;
          void main(){
            vUv = uv;
            vH = position.y;
            vNormal = normalize(normalMatrix * normal);
            gl_Position = projectionMatrix * modelViewMatrix * vec4(position, 1.0);
          }
        `,
        fragmentShader: `
          uniform sampler2D uUnionMask;
          uniform sampler2D uAnhuiMask;
          uniform sampler2D uHuaiheMask;
          uniform int uActiveRegion;
          uniform float uDarkness;
          uniform float uMaxH;
          varying vec2 vUv;
          varying float vH;
          varying vec3 vNormal;
          
          void main(){
            if(texture2D(uUnionMask, vUv).r < 0.5) discard;
            
            // ✨ 旧版 3 段渐变（你觉得好看的版本）
            float h = clamp(vH / uMaxH, 0.0, 1.0);
            vec3 c1 = vec3(0.01, 0.1, 0.25);
            vec3 c2 = vec3(0.1, 0.4, 0.35);
            vec3 c3 = vec3(0.5, 0.45, 0.25);
            vec3 color = h < 0.3 ? mix(c1, c2, h/0.3) : mix(c2, c3, (h-0.3)/0.7);
            
            float diff = max(dot(vNormal, normalize(vec3(0.5, 1.0, 0.5))), 0.0);
            vec3 litColor = color * (diff * 0.9 + 0.35);
            
            // 区域明暗
            float inAnhui  = smoothstep(0.35, 0.65, texture2D(uAnhuiMask, vUv).r);
            float inHuaihe = smoothstep(0.35, 0.65, texture2D(uHuaiheMask, vUv).r);
            
            float brightness = 1.0;
            if (uActiveRegion == 0) {
              brightness = mix(uDarkness, 1.0, inAnhui);
            } else if (uActiveRegion == 1) {
              brightness = mix(uDarkness, 1.0, inHuaihe);
            }
            
            gl_FragColor = vec4(litColor * brightness, 1.0);
          }
        `,
      });
      this.mapGroup.add(new THREE.Mesh(topGeom, this.topMat));
      this.createGoldenBorder(coords, demMax);
      this.createWalls(coords, demMax, this.config.baseY);
    },

    createMaskFromGeoJson(geoJson, size = 2048) {
      const canvas = document.createElement("canvas");
      canvas.width = canvas.height = size;
      const ctx = canvas.getContext("2d");
      ctx.fillStyle = "black";
      ctx.fillRect(0, 0, size, size);
      ctx.fillStyle = "white";

      const drawRing = (ring) => {
        ctx.beginPath();
        ring.forEach((c, i) => {
          const u =
            (c[0] - this.bounds.minLon) /
            (this.bounds.maxLon - this.bounds.minLon);
          const v =
            (c[1] - this.bounds.minLat) /
            (this.bounds.maxLat - this.bounds.minLat);
          const x = u * size;
          const y = (1 - v) * size;
          if (i === 0) ctx.moveTo(x, y);
          else ctx.lineTo(x, y);
        });
        ctx.closePath();
        ctx.fill();
      };

      geoJson.features.forEach((f) => {
        const geom = f.geometry;
        if (geom.type === "Polygon") {
          drawRing(geom.coordinates[0]);
        } else if (geom.type === "MultiPolygon") {
          geom.coordinates.forEach((poly) => drawRing(poly[0]));
        }
      });

      const tex = new THREE.CanvasTexture(canvas);
      tex.minFilter = THREE.LinearFilter;
      tex.magFilter = THREE.LinearFilter;
      return tex;
    },

    switchRegion(name, animate = true) {
      const map = { anhui: 0, huaihe: 1, all: 2 };
      this.currentRegion = name;
      const regionId = map[name];

      // 更新地形和河流的高亮
      if (this.topMat) {
        this.topMat.uniforms.uActiveRegion.value = regionId;
      }
      this.riverMaterials.forEach((mat) => {
        if (mat.uniforms.uActiveRegion) {
          mat.uniforms.uActiveRegion.value = regionId;
        }
      });

      // ✨ 计算目标位置和摄像机位置
      const cfg = this.regionConfig[name];
      let targetPos, cameraPos;

      if (name === "all") {
        targetPos = new THREE.Vector3(cfg.target.x, cfg.target.y, cfg.target.z);
        cameraPos = new THREE.Vector3(
          cfg.cameraOffset.x,
          cfg.cameraOffset.y,
          cfg.cameraOffset.z,
        );
      } else {
        // 用经纬度算中心
        targetPos = this.lonLatToWorld(
          cfg.centerLonLat[0],
          cfg.centerLonLat[1],
        );
        // 摄像机在目标正上方偏后（角度回正）
        cameraPos = new THREE.Vector3(
          targetPos.x + Math.sin(cfg.cameraAngle) * cfg.cameraDistance,
          cfg.cameraHeight,
          targetPos.z + Math.cos(cfg.cameraAngle) * cfg.cameraDistance,
        );
      }

      // ✨ 触发动画或瞬间定位
      if (animate) {
        this.startCameraAnimation(cameraPos, targetPos);
      } else {
        this.camera.position.copy(cameraPos);
        this.controls.target.copy(targetPos);
        this.controls.update();
      }
    },
    startCameraAnimation(toPos, toTarget) {
      this.cameraAnim.active = true;
      this.cameraAnim.startTime = performance.now();
      this.cameraAnim.fromPos.copy(this.camera.position);
      this.cameraAnim.toPos.copy(toPos);
      this.cameraAnim.fromTarget.copy(this.controls.target);
      this.cameraAnim.toTarget.copy(toTarget);
    },

    updateCameraAnimation() {
      if (!this.cameraAnim.active) return;

      const now = performance.now();
      const elapsed = now - this.cameraAnim.startTime;
      let t = Math.min(elapsed / this.cameraAnim.duration, 1.0);

      // 缓动函数：easeInOutCubic
      t = t < 0.5 ? 4 * t * t * t : 1 - Math.pow(-2 * t + 2, 3) / 2;

      // 插值
      this.camera.position.lerpVectors(
        this.cameraAnim.fromPos,
        this.cameraAnim.toPos,
        t,
      );
      this.controls.target.lerpVectors(
        this.cameraAnim.fromTarget,
        this.cameraAnim.toTarget,
        t,
      );
      this.controls.update();

      if (elapsed >= this.cameraAnim.duration) {
        this.cameraAnim.active = false;
      }
    },

    createMask(coords) {
      return this.createMaskFromGeoJson({
        features: [{ geometry: { type: "Polygon", coordinates: [coords] } }],
      });
    },

    handleRiverClick(event) {
      if (!this.raycaster || !this.camera || !this.renderer) return;
      if (!this.riverMeshes || this.riverMeshes.length === 0) return;
      const rect = this.renderer.domElement.getBoundingClientRect();
      const x = ((event.clientX - rect.left) / rect.width) * 2 - 1;
      const y = -(((event.clientY - rect.top) / rect.height) * 2 - 1);
      this.pointer.set(x, y);
      this.raycaster.setFromCamera(this.pointer, this.camera);
      const intersects = this.raycaster.intersectObjects(
        this.riverMeshes,
        false,
      );
      if (!intersects || intersects.length === 0) {
        this.clearRiverSelection();
        return;
      }
      const mesh = intersects[0].object;
      if (!mesh || !mesh.userData || !mesh.userData.riverName) return;
      this.selectRiverByMesh(mesh);
    },

    clearRiverSelection() {
      for (const mesh of this.riverMeshes) {
        const base = this._riverBaseColorMap.get(mesh.uuid);
        if (
          base &&
          mesh.material &&
          mesh.material.uniforms &&
          mesh.material.uniforms.uColor
        ) {
          mesh.material.uniforms.uColor.value.copy(base);
        }
      }
      this.selectedRiverMat = null;
      this.selectedRiverName = "";
    },

    selectRiverByMesh(mesh) {
      const riverName = mesh.userData.riverName || "";
      if (!riverName) return;
      if (
        this.selectedRiverName === riverName &&
        this.selectedRiverMat === mesh.material
      )
        return;
      for (const m of this.riverMeshes) {
        const base = this._riverBaseColorMap.get(m.uuid);
        if (
          base &&
          m.material &&
          m.material.uniforms &&
          m.material.uniforms.uColor
        ) {
          m.material.uniforms.uColor.value.copy(base);
        }
      }
      if (
        mesh.material &&
        mesh.material.uniforms &&
        mesh.material.uniforms.uColor
      ) {
        const base = this._riverBaseColorMap.get(mesh.uuid);
        if (base) {
          const highlight = base.clone().lerp(new THREE.Color(0xffffff), 0.6);
          mesh.material.uniforms.uColor.value.copy(highlight);
        }
      }
      this.selectedRiverMat = mesh.material;
      this.selectedRiverName = riverName;
    },

    createRivers(riverJson, demMax) {
      const labeledRivers = new Set();
      riverJson.features.forEach((f) => {
        const name = f.properties.name || "未命名";
        const lines =
          f.geometry.type === "LineString"
            ? [f.geometry.coordinates]
            : f.geometry.coordinates;
        const allSegmentPoints = [];

        const isMajor = name === "淮河" || name === "长江";
        const radius = isMajor
          ? this.config.riverBaseWidth * 4
          : this.config.riverBaseWidth * 1.8;

        lines.forEach((line) => {
          const pts = line
            .map((c) => {
              const u =
                (c[0] - this.bounds.minLon) /
                (this.bounds.maxLon - this.bounds.minLon);
              const v =
                (c[1] - this.bounds.minLat) /
                (this.bounds.maxLat - this.bounds.minLat);
              if (u < 0 || u > 1 || v < 0 || v > 1) return null;
              return new THREE.Vector3(
                (u - 0.5) * this.config.width,
                this.getOriginalH(u, v, demMax) + 20,
                -(v - 0.5) * this.config.height,
              );
            })
            .filter((p) => p !== null);

          if (pts.length < 2) return;
          allSegmentPoints.push(pts);

          const tubeGeom = new THREE.TubeGeometry(
            new THREE.CatmullRomCurve3(pts, false, "catmullrom", 0.2),
            pts.length * 2,
            radius,
            8,
            false,
          );

          const baseColor = isMajor
            ? new THREE.Color(0x00ffff)
            : new THREE.Color(0x4dc8ff);

          const mat = new THREE.ShaderMaterial({
            uniforms: {
              uTime: { value: 0 },
              uColor: { value: baseColor.clone() },
              uAnhuiMask: { value: this.topMat.uniforms.uAnhuiMask.value },
              uHuaiheMask: { value: this.topMat.uniforms.uHuaiheMask.value },
              uActiveRegion: {
                value: this.topMat.uniforms.uActiveRegion.value,
              },
              uMapBounds: {
                value: new THREE.Vector4(
                  -this.config.width / 2,
                  this.config.width / 2,
                  -this.config.height / 2,
                  this.config.height / 2,
                ),
              },
            },
            vertexShader: `
    varying vec2 vUv;
    varying vec3 vWorldPos;
    void main(){
      vUv = uv;
      vec4 wp = modelMatrix * vec4(position, 1.0);
      vWorldPos = wp.xyz;
      gl_Position = projectionMatrix * modelViewMatrix * vec4(position, 1.0);
    }
  `,
            fragmentShader: `
    uniform float uTime;
    uniform vec3 uColor;
    uniform sampler2D uAnhuiMask;
    uniform sampler2D uHuaiheMask;
    uniform int uActiveRegion;
    uniform vec4 uMapBounds;
    varying vec2 vUv;
    varying vec3 vWorldPos;
    
    void main(){
      float f1 = fract(vUv.x * 3.0 - uTime * 1.2);
      float f2 = fract(vUv.x * 6.0 - uTime * 0.6);
      float glow = smoothstep(0.3, 1.0, f1) * 0.7 + smoothstep(0.5, 1.0, f2) * 0.3;
      
      // ✨ 提亮：从 0.5/1.4 提到 1.0/2.0
      vec3 col = mix(uColor * 1.0, uColor * 2.0, glow);
      
      float maskU = (vWorldPos.x - uMapBounds.x) / (uMapBounds.y - uMapBounds.x);
      float maskV = 1.0 - (vWorldPos.z - uMapBounds.z) / (uMapBounds.w - uMapBounds.z);
      vec2 maskUV = vec2(maskU, maskV);
      
      float inAnhui  = smoothstep(0.35, 0.65, texture2D(uAnhuiMask, maskUV).r);
      float inHuaihe = smoothstep(0.35, 0.65, texture2D(uHuaiheMask, maskUV).r);
      
      float inRegion = 1.0;
      if (uActiveRegion == 0) {
        inRegion = inAnhui;
      } else if (uActiveRegion == 1) {
        inRegion = inHuaihe;
      }
      
      float brightness = mix(0.15, 1.0, inRegion);
      float alpha = mix(0.2, 1.0, inRegion);
      
      gl_FragColor = vec4(col * brightness, alpha);
    }
  `,
            transparent: true,
            blending: THREE.NormalBlending, // ✨ 保持 Normal（不会爆炸）
            depthWrite: false,
          });
          this.riverMaterials.push(mat);

          const mesh = new THREE.Mesh(tubeGeom, mat);
          mesh.userData = { riverName: name };
          this.riverMeshes.push(mesh);
          this._riverBaseColorMap.set(mesh.uuid, baseColor.clone());
          this.mapGroup.add(mesh);
        });

        if (
          allSegmentPoints.length > 0 &&
          name !== "未命名" &&
          !labeledRivers.has(name)
        ) {
          const bestPos = this.findBestLabelPosition(allSegmentPoints);
          if (bestPos) {
            this.createRiverLabel(name, bestPos, "rivers");
            labeledRivers.add(name);
          }
        }
      });
    },

    findBestLabelPosition(allSegmentPoints) {
      let longest = null;
      let maxL = 0;
      allSegmentPoints.forEach((pts) => {
        let l = 0;
        for (let i = 1; i < pts.length; i++) l += pts[i].distanceTo(pts[i - 1]);
        if (l > maxL) {
          maxL = l;
          longest = pts;
        }
      });
      if (!longest) return null;
      const halfL = maxL / 2;
      let acc = 0;
      for (let i = 1; i < longest.length; i++) {
        const sl = longest[i].distanceTo(longest[i - 1]);
        if (acc + sl >= halfL)
          return new THREE.Vector3().lerpVectors(
            longest[i - 1],
            longest[i],
            (halfL - acc) / sl,
          );
        acc += sl;
      }
      return longest[Math.floor(longest.length / 2)];
    },

    createRiverLabel(text, position, type) {
      const cfg = this.labelConfig;
      const canvas = document.createElement("canvas");
      const ctx = canvas.getContext("2d");
      ctx.font = `bold ${cfg.fontSize}px "Microsoft YaHei", sans-serif`;
      const tw = ctx.measureText(text).width;
      const th = cfg.fontSize * 1.4;
      canvas.width = tw + cfg.padding * 2;
      canvas.height = th + cfg.padding * 2;
      ctx.font = `bold ${cfg.fontSize}px "Microsoft YaHei", sans-serif`;
      ctx.textAlign = "center";
      ctx.textBaseline = "middle";
      ctx.fillStyle = cfg.bgColor;
      ctx.beginPath();
      ctx.roundRect(0, 0, canvas.width, canvas.height, 10);
      ctx.fill();
      ctx.strokeStyle = cfg.strokeColor;
      ctx.lineWidth = 4;
      ctx.stroke();
      ctx.fillStyle = type == "rivers" ? cfg.color : cfg.color2;
      ctx.fillText(text, canvas.width / 2, canvas.height / 2);

      const spriteMat = new THREE.SpriteMaterial({
        map: new THREE.CanvasTexture(canvas),
        transparent: true,
        toneMapped: false,
        depthTest: false,
      });
      const sprite = new THREE.Sprite(spriteMat);
      const aspect = canvas.width / canvas.height;
      sprite.scale.set(cfg.scale * aspect * 100, cfg.scale * 100, 1);
      sprite.position.copy(position);
      sprite.position.y += cfg.heightOffset;
      sprite.renderOrder = 999;

      // 在 createRiverLabel 函数最后改成
      sprite.userData = {
        worldPos: position.clone(),
        type: type,
      };
      this.mapGroup.add(sprite);

      this.labelSprites.push(sprite);
    },

    createGoldenBorder(coords, demMax) {
      const points = coords.map((c) => {
        const u =
          (c[0] - this.bounds.minLon) /
          (this.bounds.maxLon - this.bounds.minLon);
        const v =
          (c[1] - this.bounds.minLat) /
          (this.bounds.maxLat - this.bounds.minLat);
        return new THREE.Vector3(
          (u - 0.5) * this.config.width,
          this.getH(u, v, demMax) + 25,
          -(v - 0.5) * this.config.height,
        );
      });
      points.push(points[0].clone());
      const tubeGeom = new THREE.TubeGeometry(
        new THREE.CatmullRomCurve3(points),
        points.length,
        this.config.borderWidth,
        8,
        true,
      );
      this.borderMat = new THREE.ShaderMaterial({
        uniforms: { uTime: { value: 0 } },
        vertexShader: `varying vec2 vUv; void main(){ vUv = uv; gl_Position = projectionMatrix * modelViewMatrix * vec4(position, 1.0); }`,
        fragmentShader: `uniform float uTime; varying vec2 vUv; void main(){ float flow = fract(vUv.x * 5.0 - uTime * 0.5); float spark = pow(smoothstep(0.6, 1.0, flow), 4.0); gl_FragColor = vec4(mix(vec3(0.6, 0.4, 0.1), vec3(1.0, 0.9, 0.5), spark) * 1.5, 1.0); }`,
      });
      this.mapGroup.add(new THREE.Mesh(tubeGeom, this.borderMat));
    },

    createWalls(coords, demMax, baseY) {
      const vertices = [];
      for (let i = 0; i < coords.length; i++) {
        const u =
          (coords[i][0] - this.bounds.minLon) /
          (this.bounds.maxLon - this.bounds.minLon);
        const v =
          (coords[i][1] - this.bounds.minLat) /
          (this.bounds.maxLat - this.bounds.minLat);
        vertices.push(
          (u - 0.5) * this.config.width,
          this.getH(u, v, demMax),
          -(v - 0.5) * this.config.height,
          (u - 0.5) * this.config.width,
          baseY,
          -(v - 0.5) * this.config.height,
        );
      }
      const geom = new THREE.BufferGeometry();
      const indices = [];
      for (let i = 0; i < vertices.length / 6 - 1; i++) {
        indices.push(
          i * 2,
          i * 2 + 1,
          (i + 1) * 2,
          (i + 1) * 2,
          i * 2 + 1,
          (i + 1) * 2 + 1,
        );
      }
      geom.setAttribute(
        "position",
        new THREE.Float32BufferAttribute(vertices, 3),
      );
      geom.setIndex(indices);
      geom.computeVertexNormals();
      this.mapGroup.add(
        new THREE.Mesh(
          geom,
          new THREE.ShaderMaterial({
            uniforms: {
              uBaseY: { value: baseY },
              uMaxH: { value: this.config.maxH },
            },
            vertexShader: `varying float vY; void main(){ vY = position.y; gl_Position = projectionMatrix * modelViewMatrix * vec4(position, 1.0); }`,
            fragmentShader: `uniform float uBaseY; uniform float uMaxH; varying float vY; void main(){ float f = (vY - uBaseY) / (uMaxH - uBaseY); gl_FragColor = vec4(mix(vec3(0.1, 0.1, 0.2), vec3(0.4, 0.3, 0.1), f), 1.0); }`,
            side: THREE.DoubleSide,
          }),
        ),
      );
    },

    extractCoords(json) {
      const feat = json.features[0].geometry;
      return (
        feat.type === "Polygon"
          ? feat.coordinates[0]
          : feat.coordinates.reduce((a, b) =>
              a[0].length > b[0].length ? a : b,
            )[0]
      ).filter((c) => c && !isNaN(c[0]));
    },

    animate() {
      requestAnimationFrame(() => this.animate());

      // ✨ 更新摄像机动画
      this.updateCameraAnimation();

      this.controls.update();
      if (this.borderMat) this.borderMat.uniforms.uTime.value += 0.005;

      // ✨ 内层河流流光
      this.riverMaterials.forEach((m) => {
        const speed = this.selectedRiverMat === m ? 0.03 : 0.01;
        m.uniforms.uTime.value += speed;
      });

      if (this.stars) {
        this.stars.rotation.y += 0.0002;
        this.stars.rotation.x += 0.0001;
      }

      this.labelSprites.forEach((s) => {
        const dist = this.camera.position.distanceTo(
          s.getWorldPosition(new THREE.Vector3()),
        );
        const sFactor = dist / 25000;
        const baseS = this.labelConfig.scale * 100;
        s.scale.set(
          baseS *
            sFactor *
            (s.material.map.image.width / s.material.map.image.height),
          baseS * sFactor,
          1,
        );
      });
      this.composer.render();
    },

    onResize() {
      const container = this.$refs.container;
      if (!container) return;
      const w = container.clientWidth;
      const h = container.clientHeight;
      this.camera.aspect = w / h;
      this.camera.updateProjectionMatrix();
      this.renderer.setSize(w, h);
      this.composer.setSize(w, h);
    },
  },
};
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

.region-switcher {
  position: absolute;
  top: 100px;
  left: 24px;
  display: flex;
  flex-direction: column;
  gap: 10px;
  z-index: 50;
}
.region-switcher button {
  padding: 10px 24px;
  background: rgba(0, 24, 40, 0.6);
  border: 1px solid rgba(0, 221, 255, 0.4);
  color: #bff7ff;
  border-radius: 8px;
  cursor: pointer;
  backdrop-filter: blur(8px);
  font-size: 14px;
  font-weight: 500;
  transition: all 0.25s;
}
.region-switcher button:hover {
  background: rgba(0, 221, 255, 0.2);
  transform: translateX(4px);
}
.region-switcher button.active {
  background: rgba(0, 221, 255, 0.4);
  color: #fff;
  box-shadow: 0 0 16px rgba(0, 221, 255, 0.5);
  border-color: #00ddff;
}

.river-info {
  position: absolute;
  right: 24px;
  bottom: 92px;
  width: 360px;
  padding: 14px 16px;
  border-radius: 12px;
  background: rgba(0, 24, 40, 0.55);
  border: 1px solid rgba(0, 221, 255, 0.35);
  box-shadow: 0 0 24px rgba(0, 221, 255, 0.15);
  backdrop-filter: blur(6px);
  color: #bff7ff;
  z-index: 50;
}

.river-info-title {
  font-size: 14px;
  opacity: 0.9;
  margin-bottom: 6px;
}

.river-info-name {
  font-size: 22px;
  font-weight: 800;
  color: #00ddff;
  margin-bottom: 6px;
  text-shadow: 0 0 14px rgba(0, 221, 255, 0.35);
}

.river-info-desc {
  font-size: 13px;
  opacity: 0.85;
  line-height: 1.4;
}
</style>
