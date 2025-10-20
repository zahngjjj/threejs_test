<template>
  <div ref="container" class="content"></div>
</template>

<script setup>
import { onMounted, ref , onBeforeUnmount} from 'vue'
import * as THREE from 'three'
import { OrbitControls } from 'three/examples/jsm/controls/OrbitControls.js'
import { GLTFLoader } from 'three/examples/jsm/loaders/GLTFLoader.js'

const container = ref(null)

// 固定的路径点坐标数组
// 注意：Three.js坐标系 - X:左右, Y:上下, Z:前后
const routePoints = [
  { x: 0, y: 1.51, z: -17 },    // 起始点 - 使用模型的实际地面高度
  { x: 10, y: 1.51, z: -17 },    // 途经点1
  { x: 15, y: 1.51, z: -17 },    // 途经点1
  { x: 20, y: 1.51, z: -17 },    // 途经点1
  { x: 25, y: 1.51, z: -17 },    // 途经点1
  { x: 30, y: 1.51, z: -17 },  // 途经点1
  { x: 32, y: 1.51, z: -17 },  // 途经点1
  { x: 32, y: 1.51, z: -20 },  // 途经点2
  { x: 32, y: 1.51, z: -23 },  // 途经点2
  { x: 32, y: 1.51, z: -26 },  // 途经点2
  { x: 32, y: 1.51, z: -29 },  // 途经点2
  { x: 30, y: 1.51, z: -29 },  // 途经点3
  { x: 25, y: 1.51, z: -29 },  // 途经点3
  { x: 20, y: 1.51, z: -29 },  // 途经点3
  { x: 15, y: 1.51, z: -29 },  // 途经点3
  { x: 10, y: 1.51, z: -29 },  // 途经点3
  { x: 0, y: 1.51, z: -29 },  // 途经点4
  { x: 0, y: 1.51, z: -26 },  // 途经点4
  { x: 0, y: 1.51, z: -23 },  // 途经点4
  { x: 0, y: 1.51, z: -23 },  // 途经点4
  { x: 0, y: 1.51, z: -17 },  // 途经点4
  { x: 0, y: 1.51, z: -17 }   // 终点
]

// 车辆运动相关变量
let carObject = null
let isCarMoving = false
let carAnimationId = null
let currentPointIndex = 0 // 当前目标点索引
let moveProgress = 0 // 运动进度 0-1
const moveSpeed = 0.02 // 运动速度
let isReversing = false // 是否正在返回

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
    '/cangku20.glb',
    (gltf) => {
      console.log(gltf, 'model loaded')
        const car_scene = gltf.scene;
        scene.add(car_scene);
     
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
  
  // 调试信息：显示车辆当前位置
  console.log('点击前车辆位置:', carObject.position)
  console.log('车辆对象信息:', carObject)
  
  if (!isCarMoving) {
    // 开始运动 - 将车辆移动到起始点
    console.log('设置车辆到起始点:', routePoints[0])
    carObject.position.copy(routePoints[0])
    console.log('设置后车辆位置:', carObject.position)
    
    currentPointIndex = 0
    moveProgress = 0
    isReversing = false
    isCarMoving = true
    console.log('车辆开始沿路径运动')
    console.log('路径点:', routePoints)
    animateCar()
  } else {
    // 停止运动
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

// 车辆多点路径运动函数 - 只运动一次
const animateCar = () => {
  if (!carObject || !isCarMoving) return
  
  // 获取当前起点和终点
  const fromPointIndex = currentPointIndex
  const toPointIndex = currentPointIndex + 1
  
  // 检查是否已经到达最后一个点
  if (toPointIndex >= routePoints.length) {
    console.log('路径运动完成，停止运动')
    isCarMoving = false
    if (carAnimationId) {
      cancelAnimationFrame(carAnimationId)
      carAnimationId = null
    }
    return
  }
  
  const fromPoint = routePoints[fromPointIndex]
  const toPoint = routePoints[toPointIndex]
  
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
    moveProgress = 0 // 重置进度
    currentPointIndex++ // 移动到下一个点
    console.log(`到达途经点${currentPointIndex}，继续前进`)
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
