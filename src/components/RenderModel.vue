<script setup lang="ts">
import { ref, onMounted, onUnmounted } from 'vue'
import * as THREE from 'three'
import { OrbitControls } from 'three/examples/jsm/controls/OrbitControls'
import { STLLoader } from 'three/examples/jsm/loaders/STLLoader'

const container = ref<HTMLElement | null>(null)

// 初始化场景基础元素
const scene = new THREE.Scene()
const aspect = window.innerWidth / window.innerHeight
const frustumSize = 100 // 控制模型的缩放大小
const camera = new THREE.OrthographicCamera(
  (-frustumSize * aspect) / 2, // left
  (frustumSize * aspect) / 2, // right
  frustumSize / 2, // top
  -frustumSize / 2, // bottom
  0.1, // near plane
  5000, // far plane
)
const renderer = new THREE.WebGLRenderer({ antialias: true })
let controls: OrbitControls

// 颜色配置
const MATERIAL_CONFIG = {
  teeth: new THREE.MeshPhongMaterial({
    color: 0xffffff,
    specular: 0x888888, // 增强高光反射
    shininess: 300, // 提升光泽度
  }),
  upper: new THREE.MeshPhongMaterial({
    // color: 0xffb8b8,// 浅
    color: 0xef9998,// 适中
    // color: 0xbf6164, // 深
    specular: 0x454545,
    shininess: 150,
    transparent: true,
    // opacity: 0.95,
  }),
  lower: new THREE.MeshPhongMaterial({
    // color: 0xffb8b8,// 浅
    color: 0xef9998,// 适中
    // color: 0xbf6164,// 深
    specular: 0x454545,
    shininess: 150,
    transparent: true,
    // opacity: 0.95,
  }),
}

// 初始化场景
const initScene = () => {
  // 场景背景色
  scene.background = new THREE.Color(0x000000)

  // 初始化渲染器
  renderer.setSize(window.innerWidth, window.innerHeight)
  renderer.setPixelRatio(window.devicePixelRatio)

  // 初始化相机位置
  camera.position.set(0, -180, 0)
  camera.lookAt(0, 0, 0)
  // 添加坐标辅助（调试用）
  const axesHelper = new THREE.AxesHelper(200)
  scene.add(axesHelper)
  scene.background = new THREE.Color(0x000000)

  // 新增光照系统
  const ambientLight = new THREE.AmbientLight(0xffffff, 1.0)
  const keyLight = new THREE.DirectionalLight(0xffffff, 1.2)
  keyLight.position.set(150, -200, 120)

  const fillLight = new THREE.DirectionalLight(0xccddee, 0.8)
  fillLight.position.set(-150, -120, -100)

  const topLight = new THREE.DirectionalLight(0xffffff, 0.6)
  topLight.position.set(0, -250, 0)

  scene.add(ambientLight, keyLight, fillLight, topLight)
}
// 如需更逼真可添加纹理贴图
const loadTextures = async () => {
  const textureLoader = new THREE.TextureLoader()
  const boneTexture = await textureLoader.loadAsync('/textures/demo.png')

  MATERIAL_CONFIG.upper.map = boneTexture
  MATERIAL_CONFIG.lower.map = boneTexture
  MATERIAL_CONFIG.upper.needsUpdate = true
  MATERIAL_CONFIG.lower.needsUpdate = true
}
// 加载STL模型
const loadModels = async () => {
  const loader = new STLLoader()
  const teethMaterial = MATERIAL_CONFIG.teeth // 创建共享材质实例

  try {
    // 加载上下颌
    await Promise.all([
      loader.loadAsync('/models/Tooth_UpperJaw.stl').then((geometry) => {
        const mesh = new THREE.Mesh(geometry, MATERIAL_CONFIG.upper)
        scene.add(mesh)
      }),
      loader.loadAsync('/models/Tooth_LowerJaw.stl').then((geometry) => {
        const mesh = new THREE.Mesh(geometry, MATERIAL_CONFIG.lower)
        scene.add(mesh)
      }),
    ])
    // await loadTextures()
    // 优化牙齿加载循环
/*    for (let i = 3; i <= 30; i++) {
      const geometry = await loader.loadAsync(`/models/Tooth_${i}.stl`)
      const mesh = new THREE.Mesh(geometry, teethMaterial) // 使用共享材质
      scene.add(mesh)
    }*/
  } catch (error) {
    console.error('加载失败:', error) // 查看具体报错文件
  }
}
const loadTeethModels = async () => {
  const loader = new STLLoader()
  const existingTeeth: number[] = []
  const missingTeeth: number[] = []

  // 创建并行加载任务
  const loadTasks = []
  for (let i = 3; i <= 30; i++) {
    loadTasks.push(
      (async (toothNumber: number) => {
        try {
          const geometry = await loader.loadAsync(`/models/Tooth_${toothNumber}.stl`)
          return { success: true, geometry, toothNumber }
        } catch (error) {
          return { success: false, toothNumber }
        }
      })(i)
    )
  }

  // 等待所有加载任务完成
  const results = await Promise.all(loadTasks)

  // 处理结果
  results.forEach(({ success, geometry, toothNumber }) => {
    if (success && geometry) {
      const mesh = new THREE.Mesh(geometry, MATERIAL_CONFIG.teeth)
      scene.add(mesh)
      existingTeeth.push(toothNumber)
    } else {
      missingTeeth.push(toothNumber)
    }
  })

  // 输出加载报告
  console.groupCollapsed('[模型加载报告]')
  console.log('✅ 成功加载牙齿:', existingTeeth.join(', '))
  console.log('❌ 缺失牙齿编号:', missingTeeth.join(', '))
  console.groupEnd()

  // 可视化提示（可选）
  if (missingTeeth.length > 0) {
    const warning = document.createElement('div')
    warning.style.cssText = `
      position: fixed;
      top: 20px;
      right: 20px;
      padding: 15px;
      background: #ffeb3b;
      border-radius: 4px;
      box-shadow: 0 2px 10px rgba(0,0,0,0.1);
    `
    warning.innerHTML = `
      <b>部分牙齿模型缺失：</b><br>
      ${missingTeeth.join(', ')} 号牙齿未找到<br>
      <small>检查 models 目录文件完整性</small>
    `
    document.body.appendChild(warning)

    // 自动移除提示
    setTimeout(() => {
      document.body.removeChild(warning)
    }, 10000)
  }
}

// 动画循环
const animate = () => {
  requestAnimationFrame(animate)
  controls.update()
  renderer.render(scene, camera)
}

// 窗口尺寸变化处理
const onWindowResize = () => {
  camera.aspect = window.innerWidth / window.innerHeight
  camera.updateProjectionMatrix()
  renderer.setSize(window.innerWidth, window.innerHeight)
}

onMounted(async () => {
  if (!container.value) return

  // 初始化场景
  initScene()
  container.value.appendChild(renderer.domElement)

  // 初始化轨道控制器
  controls = new OrbitControls(camera, renderer.domElement)
  controls.enableDamping = true
  controls.dampingFactor = 0.05

  // 加载模型
  await loadModels()
  await loadTeethModels()

  // 启动动画
  animate()
  window.addEventListener('resize', onWindowResize)
})

onUnmounted(() => {
  window.removeEventListener('resize', onWindowResize)
  controls.dispose()
  scene.clear()
})
</script>

<template>
  <div class="render-model">
    <div ref="container" class="canvas-container"></div>
  </div>
</template>

<style scoped>
.canvas-container {
  width: 100vw;
  height: 100vh;
  position: fixed;
  top: 0;
  left: 0;
}
</style>
