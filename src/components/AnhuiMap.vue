<template>
  <div class="page-wrapper">
    <CyberpunkBackground class="hud-layer" />
    <div ref="container" class="map-container">
      <div v-if="loading" class="loading">正在加载安徽水利高清沙盘...</div>

      <!-- ✨ 新增：区域切换按钮 -->
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

      <!-- 河流点击交互信息面板 -->
      <div v-if="selectedRiverName" class="river-info">
        <div class="river-info-title">已选中河流</div>
        <div class="river-info-name">{{ selectedRiverName }}</div>
        <div class="river-info-desc">
          点击空白处可取消；再次点击其他河流可切换（可在此处扩展水位/预警/指标联动）。
        </div>
      </div>
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
import * as THREE from "three";
import { OrbitControls } from "three/examples/jsm/controls/OrbitControls";
import { EffectComposer } from "three/examples/jsm/postprocessing/EffectComposer";
import { RenderPass } from "three/examples/jsm/postprocessing/RenderPass";
import { UnrealBloomPass } from "three/examples/jsm/postprocessing/UnrealBloomPass";
import { fromArrayBuffer } from "geotiff";
import BottomNav from "./BottomNav.vue";
import CyberpunkBackground from "./CyberpunkBackground.vue";
import gsap from "gsap";
export default {
  name: "AnhuiSpaceMapFixed",
  components: {
    CyberpunkBackground,
    BottomNav,
  },
  data() {
    return {
      loading: true,
      selectedRiverName: "",
      // ✨ 新增：当前区域
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
    this.lakeMaterial = null;
    this.lakeMaskData = null;
    this.lakeMaskSize = 1024;

    this.riverMeshes = [];
    this.selectedRiverMat = null;
    this.raycaster = null;
    this.pointer = new THREE.Vector2();

    this._riverBaseColorMap = new Map();
    this.pointerDown = { x: 0, y: 0 };
    this.pointerMoved = false;
    this._onPointerDown = null;
    this._onPointerMove = null;
    this._onPointerUp = null;

    this.config = {
      width: 20000,
      height: 20000,
      maxH: 900,
      baseY: -400,
      hExp: 1.4,
      segments: 1024,
      borderWidth: 18,
      riverBaseWidth: 8,
      lakeDepth: 150,
    };

    // ✨ 新增：区域配置
    this.regionConfig = {
      all: {
        target: { x: 0, y: 0, z: 0 },
        cameraOffset: { x: 0, y: 28000, z: 34000 },
      },
      anhui: {
        centerLonLat: [117.3, 31.8],
        cameraDistance: 20000,
        cameraHeight: 20000,
        cameraAngle: 0,
      },
      huaihe: {
        centerLonLat: [116.5, 33.5],
        cameraDistance: 24000,
        cameraHeight: 22000,
        cameraAngle: 0,
      },
    };

    // ✨ 新增：摄像机动画状态
    this.cameraAnim = {
      active: false,
      startTime: 0,
      duration: 1500,
      fromPos: new THREE.Vector3(),
      toPos: new THREE.Vector3(),
      fromTarget: new THREE.Vector3(),
      toTarget: new THREE.Vector3(),
    };
    this.defaultRegion = "anhui";

    this.labelConfig = {
      fontSize: 42,
      color: "#0d3a68",
      color2: "#0d3a68",
      strokeColor: "#ffffff",
      strokeWidth: 6,
      bgColor: "rgba(255, 255, 255, 0.35)",
      padding: 10,
      scale: 5,
      heightOffset: 200,
    };
  },
  mounted() {
    window.addEventListener("message", this.handleIframeMessage);
    this.$nextTick(() => {
      this.initThree();
      this.animate();
      window.addEventListener("resize", this.onResize);
    });
  },
  beforeDestroy() {
    window.removeEventListener("resize", this.onResize);
    window.removeEventListener("message", this.handleIframeMessage);

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
      this.camera.position.set(0, 28000, 28000);

      this.renderer = new THREE.WebGLRenderer({
        antialias: true,
        alpha: true,
        logarithmicDepthBuffer: true,
      });
      this.renderer.setClearColor(0x000000, 0);
      this.renderer.setSize(container.clientWidth, container.clientHeight);
      this.renderer.setPixelRatio(window.devicePixelRatio);
      this.renderer.toneMapping = THREE.NoToneMapping;
      this.renderer.toneMappingExposure = 1.0;
      container.appendChild(this.renderer.domElement);

      this.controls = new OrbitControls(this.camera, this.renderer.domElement);
      this.controls.enableRotate = false;
      this.controls.enablePan = true;
      this.controls.enableDamping = true;
      this.controls.minDistance = 4000;
      this.controls.maxDistance = 60000;

      this.mapGroup = new THREE.Group();
      this.scene.add(this.mapGroup);

      this.scene.add(new THREE.AmbientLight(0xffffff, 0.6));
      const sun = new THREE.DirectionalLight(0xffffff, 0.5);
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
        0.5,
        0.6,
        0.92,
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
        // ✨ 换成项目2的数据源（并集 + 安徽 + 淮河 + 并集 DEM + 并集河流）
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

        // 用并集算 bounds
        const unionCoords = this.extractCoords(unionRes);
        this.calculateBounds(unionCoords);

        // 保持地图纵横比
        const lonRange = this.bounds.maxLon - this.bounds.minLon;
        const latRange = this.bounds.maxLat - this.bounds.minLat;
        const aspect = lonRange / latRange;
        this.config.width = 20000 * aspect;
        this.config.height = 20000;

        const scaleRatio = this.config.width / 20000;
        this.config.borderWidth *= scaleRatio;
        this.config.riverBaseWidth *= scaleRatio;
        this.labelConfig.heightOffset *= scaleRatio;

        // 读取 DEM
        const image = await tiffRes.getImage();
        const rasters = await image.readRasters();
        this.demData =
          Array.isArray(rasters) && rasters[0]?.length ? rasters : [rasters];
        this.demW = image.getWidth();
        this.demH = image.getHeight();

        // DEM 自身的地理范围
        const demBbox = image.getBoundingBox();
        this.demBounds = {
          minLon: demBbox[0],
          minLat: demBbox[1],
          maxLon: demBbox[2],
          maxLat: demBbox[3],
        };

        let demMax = 0;
        const arr = this.demData[0];
        for (let i = 0; i < arr.length; i++) {
          const v = arr[i];
          if (!isNaN(v) && v > 0 && v < 10000) {
            if (v > demMax) demMax = v;
          }
        }
        if (demMax === 0) demMax = 1500;

        this.buildMap(unionCoords, demMax);
        this.createRivers(riverRes, demMax);

        this.loading = false;

        // ✨ 默认切换到安徽（无动画，瞬间到位）
        this.$nextTick(() => {
          this.switchRegion(this.defaultRegion, false);
        });
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

    // ✨ 修改：使用 DEM 独立 bounds 采样（因为 DEM 是并集范围，可能和 bounds 不同）
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

      // ✨ 保留项目1的浅色地形风格 + 加区域高亮控制
      this.topMat = new THREE.ShaderMaterial({
        uniforms: {
          uUnionMask: { value: this.createMaskFromGeoJson(this.unionGeoJson) },
          uAnhuiMask: { value: this.createMaskFromGeoJson(this.anhuiGeoJson) },
          uHuaiheMask: {
            value: this.createMaskFromGeoJson(this.huaiheGeoJson),
          },
          uActiveRegion: { value: 0 }, // 0=anhui, 1=huaihe, 2=all
          uDarkness: { value: 0.35 }, // 非当前区域压暗程度
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
            float unionMask = texture2D(uUnionMask, vUv).r;
            if(unionMask < 0.5) discard;

            // AO 边缘阴影
            float aoRadius = 0.004;
            float aoSum = 0.0;
            for(int i = 0; i < 8; i++){
              float angle = float(i) * 0.7854;
              vec2 offset = vec2(cos(angle), sin(angle)) * aoRadius;
              aoSum += texture2D(uUnionMask, vUv + offset).r;
            }
            float ao = mix(0.55, 1.0, aoSum / 8.0);

            // 项目1的浅色地形渐变
            float h = clamp(vH / uMaxH, 0.0, 1.0);
            vec3 c1 = vec3(0.78, 0.84, 0.72);
            vec3 c2 = vec3(0.60, 0.72, 0.52);
            vec3 c3 = vec3(0.38, 0.52, 0.32);
            vec3 color;
            if(h < 0.25){
              color = mix(c1, c2, h/0.25);
            } else {
              color = mix(c2, c3, (h-0.25)/0.75);
            }

            float diff = max(dot(vNormal, normalize(vec3(0.5, 1.0, 0.5))), 0.0);
            vec3 lit = color * (diff * 0.45 + 0.5) * ao;
            lit = min(lit, vec3(0.9));

            // ✨ 区域高亮/压暗
            float inAnhui  = smoothstep(0.35, 0.65, texture2D(uAnhuiMask, vUv).r);
            float inHuaihe = smoothstep(0.35, 0.65, texture2D(uHuaiheMask, vUv).r);

            float brightness = 1.0;
            if (uActiveRegion == 0) {
              brightness = mix(uDarkness, 1.0, inAnhui);
            } else if (uActiveRegion == 1) {
              brightness = mix(uDarkness, 1.0, inHuaihe);
            }

            gl_FragColor = vec4(lit * brightness, 1.0);
          }
        `,
      });
      this.mapGroup.add(new THREE.Mesh(topGeom, this.topMat));
      this.createGoldenBorder(coords, demMax);
      this.createWalls(coords, demMax, this.config.baseY);
    },

    // ✨ 新增：从任意 GeoJson 生成 mask
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

    // ✨ 新增：区域切换
    switchRegion(name, animate = true) {
      const map = { anhui: 0, huaihe: 1, all: 2 };
      this.currentRegion = name;
      const regionId = map[name];

      // 更新地形高亮
      if (this.topMat) {
        this.topMat.uniforms.uActiveRegion.value = regionId;
      }
      // 更新河流高亮
      this.riverMaterials.forEach((mat) => {
        if (mat.uniforms.uActiveRegion) {
          mat.uniforms.uActiveRegion.value = regionId;
        }
      });

      // 计算摄像机目标
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
        targetPos = this.lonLatToWorld(
          cfg.centerLonLat[0],
          cfg.centerLonLat[1],
        );
        cameraPos = new THREE.Vector3(
          targetPos.x + Math.sin(cfg.cameraAngle) * cfg.cameraDistance,
          cfg.cameraHeight,
          targetPos.z + Math.cos(cfg.cameraAngle) * cfg.cameraDistance,
        );
      }

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
      // easeInOutCubic
      t = t < 0.5 ? 4 * t * t * t : 1 - Math.pow(-2 * t + 2, 3) / 2;

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

    // ========= 河流点击交互：拾取事件处理 =========
 // ========= 河流点击交互：拾取事件处理 =========
handleRiverClick(event) {
  if (!this.raycaster || !this.camera || !this.renderer) return;

  const rect = this.renderer.domElement.getBoundingClientRect();
  const x = ((event.clientX - rect.left) / rect.width) * 2 - 1;
  const y = -(((event.clientY - rect.top) / rect.height) * 2 - 1);
  this.pointer.set(x, y);
  this.raycaster.setFromCamera(this.pointer, this.camera);

  // ✨ 先拾取标签（渲染在最上层，优先级更高）
  if (this.labelSprites && this.labelSprites.length > 0) {
    const labelHits = this.raycaster.intersectObjects(
      this.labelSprites,
      false,
    );
    if (labelHits.length > 0) {
      const sp = labelHits[0].object;
      const name = sp.userData && sp.userData.riverName;
      if (name) {
        // 找到同名 mesh，复用同一选中逻辑
        const mesh = this.riverMeshes.find(
          (m) => m.userData && m.userData.riverName === name,
        );
        if (mesh) {
          this.selectRiverByMesh(mesh);
          return;
        } else {
          // 兜底：只更新面板显示
          this.selectedRiverName = name;
          this.selectedRiverMat = null;
          return;
        }
      }
    }
  }

  // 再拾取河流 tube
  if (!this.riverMeshes || this.riverMeshes.length === 0) {
    this.clearRiverSelection();
    return;
  }
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
        const radius =
          name === "淮河" || name === "长江"
            ? this.config.riverBaseWidth * 7
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
                this.getH(u, v, demMax) + 30,
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
            6,
            false,
          );

          const baseColor =
            name === "淮河" || name === "长江"
              ? new THREE.Color(0x1a5fa8)
              : new THREE.Color(0x3a85c8);

          const mat = new THREE.ShaderMaterial({
            uniforms: {
              uTime: { value: 0 },
              uColor: { value: baseColor.clone() },
              uBaseColor: { value: new THREE.Color(0x2889dd) },
              uFlowColor: { value: new THREE.Color(0xbfe6ff) },
              uHeadColor: { value: new THREE.Color(0x63c2fe) },
              // ✨ 新增：区域高亮相关
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
              uniform vec3 uBaseColor;
              uniform vec3 uFlowColor;
              uniform vec3 uHeadColor;
              uniform sampler2D uAnhuiMask;
              uniform sampler2D uHuaiheMask;
              uniform int uActiveRegion;
              uniform vec4 uMapBounds;
              varying vec2 vUv;
              varying vec3 vWorldPos;

              void main(){
                vec3 baseColor = uBaseColor;
                float baseAlpha = 0.92;

                float f = fract(vUv.x * 1.0 - uTime * 0.4);
                float bandStart = 0.35;
                float bandEnd = 1.0;
                float bandMask = smoothstep(bandStart, bandStart + 0.05, f)
                               * smoothstep(bandEnd, bandEnd - 0.02, f);
                float localT = clamp((f - bandStart) / (bandEnd - bandStart), 0.0, 1.0);
                float colorT = pow(localT, 1.3);
                vec3 flowColor = mix(uFlowColor, uHeadColor, colorT);
                float flowAlpha = pow(localT, 2.0);
                float headBoost = smoothstep(0.8, 1.0, localT);
                flowColor += uHeadColor * headBoost * 0.6;
                flowAlpha = min(flowAlpha + headBoost * 0.3, 1.0);

                float flowIntensity = bandMask * flowAlpha;
                vec3 finalColor = mix(baseColor, flowColor, flowIntensity);
                float finalAlpha = baseAlpha + flowIntensity * 0.08;
                finalAlpha = clamp(finalAlpha, 0.0, 1.0);

                // ✨ 区域高亮控制
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

                float brightness = mix(0.35, 1.0, inRegion);
                float alphaFactor = mix(0.25, 1.0, inRegion);

                gl_FragColor = vec4(finalColor * brightness, finalAlpha * alphaFactor);
              }
            `,
            transparent: true,
            blending: THREE.NormalBlending,
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
  ctx.roundRect(0, 0, canvas.width, canvas.height, 6);
  ctx.fill();

  ctx.shadowColor = "rgba(255, 255, 255, 0.9)";
  ctx.shadowBlur = 8;
  ctx.shadowOffsetX = 0;
  ctx.shadowOffsetY = 0;

  ctx.lineJoin = "round";
  ctx.lineWidth = cfg.strokeWidth;
  ctx.strokeStyle = cfg.strokeColor;
  ctx.strokeText(text, canvas.width / 2, canvas.height / 2);

  ctx.shadowColor = "transparent";
  ctx.shadowBlur = 0;

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

  // ✨ 新增：给标签打上河流名，用于点击拾取
  sprite.userData = {
    riverName: text,
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
          this.getH(u, v, demMax) + 60,
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
        vertexShader: `
          varying vec2 vUv;
          void main(){
            vUv = uv;
            gl_Position = projectionMatrix * modelViewMatrix * vec4(position, 1.0);
          }
        `,
        fragmentShader: `
          uniform float uTime;
          varying vec2 vUv;
          void main(){
            vec3 baseColor = vec3(0.13, 0.83, 0.93);
            vec3 sparkColor = vec3(0.75, 0.98, 1.0);
            float flow = fract(vUv.x * 3.0 - uTime * 0.3);
            float spark = pow(smoothstep(0.75, 1.0, flow), 3.0);
            vec3 color = mix(baseColor, sparkColor, spark * 0.5);
            gl_FragColor = vec4(color, 1.0);
          }
        `,
      });
      this.mapGroup.add(new THREE.Mesh(tubeGeom, this.borderMat));
    },

    createMask(coords) {
      const canvas = document.createElement("canvas");
      canvas.width = canvas.height = 2048;
      const ctx = canvas.getContext("2d");
      ctx.fillStyle = "black";
      ctx.fillRect(0, 0, 2048, 2048);
      ctx.fillStyle = "white";
      ctx.beginPath();
      coords.forEach((c, i) => {
        const u =
          (c[0] - this.bounds.minLon) /
          (this.bounds.maxLon - this.bounds.minLon);
        const v =
          (c[1] - this.bounds.minLat) /
          (this.bounds.maxLat - this.bounds.minLat);
        if (i === 0) ctx.moveTo(u * 2048, (1 - v) * 2048);
        else ctx.lineTo(u * 2048, (1 - v) * 2048);
      });
      ctx.fill();
      return new THREE.CanvasTexture(canvas);
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
            vertexShader: `
              varying float vY;
              void main(){
                vY = position.y;
                gl_Position = projectionMatrix * modelViewMatrix * vec4(position, 1.0);
              }
            `,
            fragmentShader: `
              uniform float uBaseY;
              uniform float uMaxH;
              varying float vY;
              void main(){
                float f = (vY - uBaseY) / (uMaxH - uBaseY);
                vec3 topColor = vec3(0.12, 0.22, 0.52);
                vec3 bottomColor = vec3(0.02, 0.03, 0.08);
                float t = pow(f, 0.7);
                vec3 color = mix(bottomColor, topColor, t);
                gl_FragColor = vec4(color, 1.0);
              }
            `,
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

      // ✨ 摄像机动画更新
      this.updateCameraAnimation();

      if (this.controls && !this.isTransitioning && !this.cameraAnim.active) {
        this.controls.update();
      }

      if (
        this.camera &&
        this.controls &&
        !this.isTransitioning &&
        !this.cameraAnim.active
      ) {
        const dist = this.camera.position.distanceTo(this.controls.target);

        if (dist < 5000) {
          this.isTransitioning = true;

          if (this.controls) {
            this.controls.enabled = false;
          }

          const centerLonLat = this.threePosToLonLat(this.controls.target);

          this.$emit("zoom-in-limit", {
            lonLat: centerLonLat,
            zoom: 14,
          });
        }
      }

      if (this.borderMat) this.borderMat.uniforms.uTime.value += 0.005;
      if (this.lakeMaterial) {
        this.lakeMaterial.uniforms.uTime.value += 0.015;
        this.lakeMaterial.uniforms.uCameraPos.value.copy(this.camera.position);
      }

      this.riverMaterials.forEach((m) => {
        const speed = this.selectedRiverMat === m ? 0.04 : 0.018;
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

    threePosToLonLat(pos) {
      if (!pos || !this.config || !this.bounds) return [117.2, 31.8];
      const u = pos.x / this.config.width + 0.5;
      const v = -pos.z / this.config.height + 0.5;
      const lon =
        u * (this.bounds.maxLon - this.bounds.minLon) + this.bounds.minLon;
      const lat =
        v * (this.bounds.maxLat - this.bounds.minLat) + this.bounds.minLat;
      return [lon, lat];
    },

    lonLatToThreePos(lonLat) {
      const u =
        (lonLat[0] - this.bounds.minLon) /
        (this.bounds.maxLon - this.bounds.minLon);
      const v =
        (lonLat[1] - this.bounds.minLat) /
        (this.bounds.maxLat - this.bounds.minLat);
      return new THREE.Vector3(
        (u - 0.5) * this.config.width,
        0,
        -(v - 0.5) * this.config.height,
      );
    },

    syncViewFrom2D(lonLat) {
      // 回退到当前选中区域视角
      this.switchRegion(this.currentRegion, false);

      setTimeout(() => {
        this.isTransitioning = false;
        if (this.controls) {
          this.controls.enabled = true;
        }
      }, 300);
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

    handleTransitionTrigger() {
      const targetLonLat = this.threePosToLonLat(this.controls.target);
      this.isTransitioned = true;
      const iframeWin = this.$refs.mapIframe.contentWindow;
      iframeWin.postMessage(
        {
          type: "SYNC_MAP",
          lonLat: targetLonLat,
          zoom: 14,
        },
        "http://localhost:5174",
      );
    },

    handleIframeMessage(event) {
      if (event.origin !== "http://localhost:5173") return;

      if (event.data.type === "BACK_TO_3D") {
        this.isTransitioned = false;
        const lonLat = event.data.lonLat;
        this.syncViewFrom2D(lonLat);
      }
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
  background: radial-gradient(
    ellipse at center,
    #0a1929 0%,
    #050b14 60%,
    #02060c 100%
  );
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

/* ✨ 新增：区域切换按钮（沿用项目1的浅色主题） */
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
  background: rgba(255, 255, 255, 0.75);
  border: 1px solid rgba(36, 120, 212, 0.45);
  color: #1a5490;
  border-radius: 8px;
  cursor: pointer;
  backdrop-filter: blur(8px);
  font-size: 14px;
  font-weight: 600;
  transition: all 0.25s;
  box-shadow: 0 2px 12px rgba(36, 120, 212, 0.1);
}
.region-switcher button:hover {
  background: rgba(255, 255, 255, 0.95);
  transform: translateX(4px);
  box-shadow: 0 4px 16px rgba(36, 120, 212, 0.25);
}
.region-switcher button.active {
  background: linear-gradient(135deg, #2478d4, #22d3ee);
  color: #fff;
  border-color: #22d3ee;
  box-shadow: 0 0 20px rgba(34, 211, 238, 0.6);
}

/* ===== 河流点击交互面板 ===== */
.river-info {
  position: absolute;
  right: 24px;
  bottom: 92px;
  width: 360px;
  padding: 14px 16px;
  border-radius: 12px;
  background: rgba(255, 255, 255, 0.85);
  border: 1px solid rgba(36, 120, 212, 0.45);
  box-shadow: 0 4px 20px rgba(36, 120, 212, 0.15);
  backdrop-filter: blur(8px);
  color: #1a5490;
  z-index: 50;
}

.river-info-title {
  font-size: 14px;
  opacity: 0.75;
  margin-bottom: 6px;
  color: #557fa5;
}

.river-info-name {
  font-size: 22px;
  font-weight: 800;
  color: #2478d4;
  margin-bottom: 6px;
  text-shadow: 0 0 12px rgba(255, 255, 255, 0.9);
}
.river-info-desc {
  font-size: 13px;
  opacity: 0.85;
  line-height: 1.4;
}
</style>
