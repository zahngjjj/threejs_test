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
  { x: 0, y: 2.68, z: -17 },    // 起始点,顶点1 - 使用模型的实际地面高度
  { x: 10, y: 2.68, z: -17 },    // 途经点1
  { x: 15, y: 2.68, z: -17 },    // 途经点1
  { x: 20, y: 2.68, z: -17 },    // 途经点1
  { x: 25, y: 2.68, z: -17 },    // 途经点1
  { x: 30, y: 2.68, z: -17 },  // 途经点1
  { x: 32, y: 2.68, z: -17 },  // 顶点2
  { x: 32, y: 2.68, z: -20 },  // 途经点2
  { x: 32, y: 2.68, z: -23 },  // 途经点2
  { x: 32, y: 2.68, z: -26 },  // 途经点2
  { x: 32, y: 2.68, z: -29 },  // 顶点3
  { x: 30, y: 2.68, z: -29 },  // 途经点3
  { x: 25, y: 2.68, z: -29 },  // 途经点3
  { x: 20, y: 2.68, z: -29 },  // 途经点3
  { x: 15, y: 2.68, z: -29 },  // 途经点3
  { x: 10, y: 2.68, z: -29 },  // 途经点3
  { x: 0, y: 2.68, z: -29 },  // 顶点4
  { x: 0, y: 2.68, z: -26 },  // 途经点4
  { x: 0, y: 2.68, z: -23 },  // 途经点4
  { x: 0, y: 2.68, z: -20 },  // 途经点4
  { x: 0, y: 2.68, z: -17 }   // 终点,起始点,顶点1
]

// 车辆运动相关变量
let carObject = null
let isCarMoving = false
let carAnimationId = null
let currentPointIndex = 0 // 当前目标点索引
let moveProgress = 0 // 运动进度 0-1
const moveSpeed = 0.02 // 运动速度
let isReversing = false // 是否正在返回
let isContinuousMovement = true // 是否循环运动，true为循环运动，false为单次运动

// Three.js 相关变量
let scene, camera, renderer, controls, raycaster, mouse

// 初始化场景
const initScene = () => {
  scene = new THREE.Scene()
  scene.background = new THREE.Color(0x0a1530)
  
  // 添加坐标轴辅助器,调试用
  // const axesHelper = new THREE.AxesHelper(20) // 轴长度为20
  // scene.add(axesHelper)
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
    '/chache12.glb',
    (gltf) => {
      console.log(gltf, 'model loaded')
      const car_scene = gltf.scene;
      // 缩放整个场景到0.8倍
      car_scene.scale.set(0.3, 0.3, 0.3);
      // 将模型绕Y轴旋转30度（π/6弧度）,180度换算成弧度制是π
      car_scene.rotation.x = Math.PI / 12;
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
    const clickedObject = intersects[0].object
    // 向上遍历父级对象，寻找叉车的根对象
    let targetObject = clickedObject
    while (targetObject) {    
      // 检查是否是叉车相关的对象（根据层级结构）
      if (targetObject.name === "car" || 
          targetObject.name === "forklift_truck.fbx" ||
          targetObject.name === "car.001") {
        handleCarClick(targetObject)
        break
      }
      
      // 继续向上查找父级对象
      targetObject = targetObject.parent
      // 如果到达场景根节点，停止查找
      if (targetObject === scene) {
        break
      }
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

// 车辆多点路径运动函数 - 循环运动，带方向控制
const animateCar = () => {
  if (!carObject || !isCarMoving) return
  
    // 检查是否已经到达最后一个点，如果是则重新开始
  if (currentPointIndex >= routePoints.length) {
    console.log('完成一轮路径运动，重新开始循环')
    currentPointIndex = 0 // 重置到起始点
    moveProgress = 0 // 重置进度
  }
  // 获取当前起点和终点
  const fromPointIndex = currentPointIndex
  const toPointIndex = currentPointIndex + 1
  
  // 如果下一个点超出范围，根据isContinuousMovement参数决定行为
  if (toPointIndex >= routePoints.length) {
    if (isContinuousMovement) {
      // 循环运动模式：重新开始
      console.log('完成一轮路径运动，重新开始循环')
      currentPointIndex = 0 // 重置到起始点
      moveProgress = 0 // 重置进度
      carAnimationId = requestAnimationFrame(animateCar) // 继续动画
      return
    } else {
      // 单次运动模式：停止运动
      console.log('路径运动完成，停止运动')
      isCarMoving = false
      if (carAnimationId) {
        cancelAnimationFrame(carAnimationId)
        carAnimationId = null
      }
      return
    }
  }


  
  const fromPoint = routePoints[fromPointIndex]
  const toPoint = routePoints[toPointIndex]
  
  // 计算运动方向并设置车辆旋转
  if (moveProgress === 0) { // 只在开始新路段时计算旋转
    const deltaX = toPoint.x - fromPoint.x
    const deltaZ = toPoint.z - fromPoint.z
    
    // 根据运动方向设置车辆旋转角度
    let targetRotation = 0
    
    if (Math.abs(deltaX) > Math.abs(deltaZ)) {
      // 主要是X方向运动
      if (deltaX > 0) {
        targetRotation = Math.PI / 2 // 向右运动，车头向右（90度）
      } else {
        targetRotation = -Math.PI / 2  // 向左运动，车头向左（-90度）
      }
          // 设置车辆旋转
     carObject.rotation.z = -targetRotation
    } else {
      // 主要是Z方向运动
      if (deltaZ > 0) {
        targetRotation = Math.PI // 向前运动，车头向前（0度）
      } else {
        targetRotation = 0 // 向后运动，车头向后（180度）
      }
            // 设置车辆旋转
      carObject.rotation.z = targetRotation
    }
    

    console.log(`路段${fromPointIndex}->${toPointIndex}: 方向(${deltaX.toFixed(1)}, ${deltaZ.toFixed(1)}), 旋转角度: ${(targetRotation * 180 / Math.PI).toFixed(0)}度`)
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
