<template>
  <div ref="container" class="content"></div>
</template>

<script setup>
import { onMounted, ref , onBeforeUnmount} from 'vue'
import * as THREE from 'three'
import { OrbitControls } from 'three/examples/jsm/controls/OrbitControls.js'
import { GLTFLoader } from 'three/examples/jsm/loaders/GLTFLoader.js'

const container = ref(null)

// 固定的起始点和终点坐标
const startPoint = { x: 0, y: 0, z: 0 }
const endPoint = { x: 10, y: 0, z: 0 }

// 车辆运动相关变量
let carObject = null
let isCarMoving = false
let carAnimationId = null
let currentTarget = 'end' // 'end' 或 'start'
let moveProgress = 0 // 运动进度 0-1
const moveSpeed = 0.02 // 运动速度

// Three.js 相关变量
let scene, camera, renderer, controls, raycaster, mouse

// 初始化场景
const initScene = () => {
  scene = new THREE.Scene()
  scene.background = new THREE.Color(0x0a1530)
}

// 初始化相机
const initCamera = () => {
  camera = new THREE.PerspectiveCamera(
    45,
    container.value.clientWidth / container.value.clientHeight,
    0.1,
    1000
  )
  camera.position.set(0, 10, 20)
}

// 初始化渲染器
const initRenderer = () => {
  renderer = new THREE.WebGLRenderer({ antialias: true })
  renderer.setSize(container.value.clientWidth, container.value.clientHeight)
  container.value.appendChild(renderer.domElement)
}

// 初始化灯光
const initLights = () => {
  const ambientLight = new THREE.AmbientLight(0xffffff, 0.6)
  scene.add(ambientLight)
  const dirLight = new THREE.DirectionalLight(0xffffff, 1)
  dirLight.position.set(5, 10, 7.5)
  scene.add(dirLight)
}

// 初始化控制器
const initControls = () => {
  controls = new OrbitControls(camera, renderer.domElement)
}

// 加载模型
const loadModel = () => {
  const loader = new GLTFLoader()
  loader.load(
    '/cangku8.glb',
    (gltf) => {
      console.log(gltf, 'model loaded')
      scene.add(gltf.scene)
    },
    (progress) => {
      console.log('Loading progress:', (progress.loaded / progress.total * 100) + '%')
    },
    (error) => {
      console.error('Error loading model:', error)
    }
  )
}

// 初始化射线检测
const initRaycaster = () => {
  raycaster = new THREE.Raycaster()
  mouse = new THREE.Vector2()
}

// 处理车辆点击事件
const handleCarClick = (clickedObject) => {
  carObject = clickedObject
  
  // 设置车辆初始位置为起始点
  if (!isCarMoving) {
    carObject.position.set(startPoint.x, startPoint.y, startPoint.z)
    moveProgress = 0
    currentTarget = 'end'
  }
  
  // 开始或停止运动
  if (!isCarMoving) {
    isCarMoving = true
    animateCar()
    console.log(`车辆开始从起始点(${startPoint.x}, ${startPoint.y}, ${startPoint.z})向终点(${endPoint.x}, ${endPoint.y}, ${endPoint.z})移动`)
  } else {
    isCarMoving = false
    if (carAnimationId) {
      cancelAnimationFrame(carAnimationId)
      carAnimationId = null
    }
    console.log('车辆停止运动')
  }
}

// 点击事件处理函数
const handleClick = (event) => {
  // 计算鼠标在画布中的归一化坐标
  const rect = event.target.getBoundingClientRect()
  mouse.x = ((event.clientX - rect.left) / rect.width) * 2 - 1
  mouse.y = -((event.clientY - rect.top) / rect.height) * 2 + 1

  // 更新射线
  raycaster.setFromCamera(mouse, camera)

  // 计算射线和模型相交的点
  const intersects = raycaster.intersectObjects(scene.children, true)

  if (intersects.length > 0) {
    console.log('点击到的对象：', intersects[0].object)
    console.log('对象名称：', intersects[0].object.name)
    console.log('材质信息：', intersects[0].object.material)
    
    const clickedObject = intersects[0].object
    
    // 检查对象名称是否为 "car"
    if (clickedObject.name === "car") {
      handleCarClick(clickedObject)
    }
  }
}

// 窗口大小调整处理
const handleResize = () => {
  camera.aspect = container.value.clientWidth / container.value.clientHeight
  camera.updateProjectionMatrix()
  renderer.setSize(container.value.clientWidth, container.value.clientHeight)
}

// 渲染循环
const animate = () => {
  requestAnimationFrame(animate)
  controls.update()
  renderer.render(scene, camera)
}

// 车辆坐标运动函数
const animateCar = () => {
  if (!carObject || !isCarMoving) return
  
  // 确定当前的起点和终点
  let fromPoint, toPoint
  if (currentTarget === 'end') {
    fromPoint = startPoint
    toPoint = endPoint
  } else {
    fromPoint = endPoint
    toPoint = startPoint
  }
  
  // 更新运动进度
  moveProgress += moveSpeed
  
  // 使用线性插值计算当前位置
  const t = Math.min(moveProgress, 1) // 确保不超过1
  
  // 线性插值公式: current = from + (to - from) * t
  carObject.position.x = fromPoint.x + (toPoint.x - fromPoint.x) * t
  carObject.position.y = fromPoint.y + (toPoint.y - fromPoint.y) * t
  carObject.position.z = fromPoint.z + (toPoint.z - fromPoint.z) * t
  
  // 检查是否到达目标点
  if (moveProgress >= 1) {
    // 切换目标点
    currentTarget = currentTarget === 'end' ? 'start' : 'end'
    moveProgress = 0 // 重置进度
    
    console.log(`车辆到达${currentTarget === 'end' ? '起始' : '终点'}点，开始向${currentTarget === 'end' ? '终点' : '起始点'}移动`)
  }
  
  carAnimationId = requestAnimationFrame(animateCar)
}

onMounted(() => {
  // 初始化Three.js组件
  initScene()
  initCamera()
  initRenderer()
  initLights()
  initControls()
  initRaycaster()
  
  // 加载模型
  loadModel()
  
  // 添加事件监听器
  window.addEventListener('resize', handleResize)
  renderer.domElement.addEventListener('click', handleClick)
  
  // 开始渲染循环
  animate()
})

// 在组件卸载时移除事件监听器和清理动画
onBeforeUnmount(() => {
  // 移除事件监听器
  if (renderer && renderer.domElement) {
    renderer.domElement.removeEventListener('click', handleClick)
  }
  window.removeEventListener('resize', handleResize)
  
  // 清理车辆动画
  if (carAnimationId) {
    cancelAnimationFrame(carAnimationId)
  }
  isCarMoving = false
})
</script>

<style scoped>
.content {
  width: 100vw;
  height: 100vh;
}
</style>
