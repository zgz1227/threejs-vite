<template>
  <div ref="container" class="canvas-container"></div>
</template>

<script setup lang="ts">
import { ref, onMounted, onUnmounted } from 'vue'
import * as THREE from 'three'
import { OrbitControls } from 'three/examples/jsm/controls/OrbitControls'
import { STLLoader } from 'three/examples/jsm/loaders/STLLoader'
import { GUI } from 'dat.gui'

const container = ref<HTMLElement | null>(null)
const gui = ref<GUI | null>(null)

// 次表面散射着色器
const subsurfaceScatteringShader = {
  uniforms: {
    color: { value: new THREE.Color(0xffe4c4) },
    aoMap: { value: null },
    veinMap: { value: null },
    lightPosition: { value: new THREE.Vector3() },
    scatterColor: { value: new THREE.Color(0xff9999) },
    scatterIntensity: { value: 0.3 },
  },
  vertexShader: `
    varying vec3 vNormal;
    varying vec3 vPosition;
    varying vec2 vUv;
    void main() {
      vNormal = normalize(normalMatrix * normal);
      vPosition = (modelViewMatrix * vec4(position, 1.0)).xyz;
      vUv = uv;
      gl_Position = projectionMatrix * modelViewMatrix * vec4(position, 1.0);
    }
  `,
  fragmentShader: `
    uniform vec3 color;
    uniform sampler2D aoMap;
    uniform sampler2D veinMap;
    uniform vec3 lightPosition;
    uniform vec3 scatterColor;
    uniform float scatterIntensity;

    varying vec3 vNormal;
    varying vec3 vPosition;
    varying vec2 vUv;

    void main() {
      // 基础颜色
      vec3 baseColor = color;

      // 环境遮挡
      float ao = texture2D(aoMap, vUv).r;

      // 血管纹理
      vec3 veins = texture2D(veinMap, vUv).rgb;

      // 光照计算
      vec3 lightDir = normalize(lightPosition - vPosition);
      float diffuse = max(dot(vNormal, lightDir), 0.0);

      // 次表面散射
      float thickness = 1.0 - diffuse;
      vec3 sss = scatterColor * scatterIntensity * thickness;

      // 合成颜色
      vec3 finalColor = (baseColor + sss) * ao * (1.0 - veins.r * 0.3);

      gl_FragColor = vec4(finalColor, 1.0);
    }
  `,
}

// 初始化场景
const scene = new THREE.Scene()
const camera = new THREE.PerspectiveCamera(75, window.innerWidth / window.innerHeight, 0.1, 1000)
const renderer = new THREE.WebGLRenderer({ antialias: true })
let controls: OrbitControls

// 材质配置
const textureLoader = new THREE.TextureLoader()
const materials = {
  upper: null as THREE.Material | null,
  lower: null as THREE.Material | null,
}

// 初始化场景
const initScene = async () => {
  scene.background = new THREE.Color(0x000000)

  // 加载纹理
  const [aoMap, veinMap] = await Promise.all([
    /*    textureLoader.loadAsync('/textures/demo.jpg'),
    textureLoader.loadAsync('/textures/veins_map.png')*/
  ])

  // 创建自定义材质
  materials.upper = new THREE.ShaderMaterial({
    ...subsurfaceScatteringShader,
    uniforms: {
      ...subsurfaceScatteringShader.uniforms,
      color: { value: new THREE.Color(0xffe4c4) },
      aoMap: { value: aoMap },
      veinMap: { value: veinMap },
    },
  })

  materials.lower = new THREE.ShaderMaterial({
    ...subsurfaceScatteringShader,
    uniforms: {
      ...subsurfaceScatteringShader.uniforms,
      color: { value: new THREE.Color(0xffdab9) },
      aoMap: { value: aoMap },
      veinMap: { value: veinMap },
    },
  })

  // 光源配置
  const ambientLight = new THREE.AmbientLight(0xffffff, 0.6)
  const keyLight = new THREE.DirectionalLight(0xffeedd, 1.2)
  keyLight.position.set(150, 200, 100)
  scene.add(ambientLight, keyLight)

  // 初始化渲染器
  renderer.setSize(window.innerWidth, window.innerHeight)
  renderer.setPixelRatio(window.devicePixelRatio)
  camera.position.set(0, 100, 200)
}

// 加载模型
const loadModels = async () => {
  const loader = new STLLoader()

  // 加载上下颌
  const [upper, lower] = await Promise.all([
    loader.loadAsync('/models/Tooth_Upperlaw.stl'),
    loader.loadAsync('/models/Tooth_Lowerlaw.stl'),
  ])

  const upperMesh = new THREE.Mesh(upper, materials.upper)
  const lowerMesh = new THREE.Mesh(lower, materials.lower)

  scene.add(upperMesh, lowerMesh)
}

// 初始化GUI
const initGUI = () => {
  gui.value = new GUI()
  const params = {
    scatterIntensity: 0.3,
    lightX: 150,
    lightY: 200,
    lightZ: 100,
  }

  gui.value.add(params, 'scatterIntensity', 0, 1).onChange((v) => {
    materials.upper!.uniforms.scatterIntensity.value = v
    materials.lower!.uniforms.scatterIntensity.value = v
  })

  const lightFolder = gui.value.addFolder('Light Position')
  lightFolder.add(params, 'lightX', -500, 500).onChange((v) => {
    scene.children[2].position.x = v
  })
  lightFolder.add(params, 'lightY', 0, 500).onChange((v) => {
    scene.children[2].position.y = v
  })
  lightFolder.add(params, 'lightZ', -500, 500).onChange((v) => {
    scene.children[2].position.z = v
  })
}

// 动画循环
const animate = () => {
  requestAnimationFrame(animate)
  controls.update()
  renderer.render(scene, camera)
}

onMounted(async () => {
  if (!container.value) return

  await initScene()
  container.value.appendChild(renderer.domElement)

  controls = new OrbitControls(camera, renderer.domElement)
  controls.enableDamping = true

  await loadModels()
  initGUI()
  animate()
  window.addEventListener('resize', () => {
    camera.aspect = window.innerWidth / window.innerHeight
    camera.updateProjectionMatrix()
    renderer.setSize(window.innerWidth, window.innerHeight)
  })
})

onUnmounted(() => {
  gui.value?.destroy()
  window.removeEventListener('resize')
  controls.dispose()
})
</script>

<style>
.canvas-container {
  width: 100vw;
  height: 100vh;
}
</style>
