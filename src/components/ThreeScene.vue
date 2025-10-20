<template>
  <div ref="container" class="content"></div>
  
  <!-- 车辆信息弹框 -->
  <div v-if="showCarModal" class="car-modal" @click="closeCarModal">
    <div class="modal-content" @click.stop>
      <div class="modal-header">
        <h3>车辆信息</h3>
        <button class="close-btn" @click="closeCarModal">&times;</button>
      </div>
      <div class="modal-body">
        <p><strong>车辆编号:</strong> {{ carInfo.id }}</p>
        <p><strong>车辆类型:</strong> {{ carInfo.type }}</p>
        <p><strong>当前状态:</strong> {{ carInfo.status }}</p>
        <p><strong>运行路线:</strong> {{ carInfo.route }}</p>
        <p><strong>位置坐标:</strong> X: {{ carInfo.position.x.toFixed(2) }}, Y: {{ carInfo.position.y.toFixed(2) }}, Z: {{ carInfo.position.z.toFixed(2) }}</p>
        <p><strong>运行时间:</strong> {{ carInfo.runTime }}小时</p>
      </div>
    </div>
  </div>
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

// 车辆弹框相关数据
const showCarModal = ref(false)
const carInfo = ref({
  id: 'CAR-001',
  type: '运输车辆',
  status: '运行中',
  position: { x: 0, y: 0, z: 0 },
  runTime: 8.5,
  route: `起始点(${startPoint.x}, ${startPoint.y}, ${startPoint.z}) ↔ 终点(${endPoint.x}, ${endPoint.y}, ${endPoint.z})`
})

// 车辆运动相关变量
let carObject = null
let isCarMoving = false
let carAnimationId = null
let currentTarget = 'end' // 'end' 或 'start'
let moveProgress = 0 // 运动进度 0-1
const moveSpeed = 0.001 // 运动速度

// 关闭弹框
const closeCarModal = () => {
  showCarModal.value = false
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
  
  // 更新车辆信息中的位置
  carInfo.value.position.x = carObject.position.x
  carInfo.value.position.y = carObject.position.y
  carInfo.value.position.z = carObject.position.z
  
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
  // 初始化场景
  const scene = new THREE.Scene()
  scene.background = new THREE.Color(0x0a1530)
  // scene.background = new THREE.Color(0xffffff)

  // 初始化相机
  const camera = new THREE.PerspectiveCamera(
    45,
    container.value.clientWidth / container.value.clientHeight,
    0.1,
    1000
  )
  camera.position.set(0, 10, 20)

  // 初始化渲染器
  const renderer = new THREE.WebGLRenderer({ antialias: true })
  renderer.setSize(container.value.clientWidth, container.value.clientHeight)
  container.value.appendChild(renderer.domElement)

  // 添加环境光与方向光
  const ambientLight = new THREE.AmbientLight(0xffffff, 0.6)
  scene.add(ambientLight)
  const dirLight = new THREE.DirectionalLight(0xffffff, 1)
  dirLight.position.set(5, 10, 7.5)
  scene.add(dirLight)

  // 控制器
  const controls = new OrbitControls(camera, renderer.domElement)
  // controls.autoRotate = true
  // controls.autoRotateSpeed = 1.5

  // 临时添加一个立方体测试场景
  const geometry = new THREE.BoxGeometry(5, 5, 5)
  const material = new THREE.MeshPhongMaterial({ color: 0x0a1530 })
  const cube = new THREE.Mesh(geometry, material)
  cube.visible = false  // 设置立方体不可见
  scene.add(cube)

  // 注释掉模型加载部分

  const loader = new GLTFLoader()
  loader.load(
    '/cangku8.glb',
    (gltf) => {
      console.log(gltf,'dddddd')
      scene.add(gltf.scene)
    },
    (progress) => {
      console.log('Loading progress:', (progress.loaded / progress.total * 100) + '%')
    },
    (error) => {
      console.error('Error loading model:', error)
    }
  )

  // 删除注释掉的重复代码块
  
  // 自适应窗口大小
  window.addEventListener('resize', () => {
    camera.aspect = container.value.clientWidth / container.value.clientHeight
    camera.updateProjectionMatrix()
    renderer.setSize(container.value.clientWidth, container.value.clientHeight)
  })

  // 渲染循环
  const animate = () => {
    requestAnimationFrame(animate)
    controls.update()
    renderer.render(scene, camera)
  }
  animate()



  const raycaster = new THREE.Raycaster()
  const mouse = new THREE.Vector2()

  // 添加点击事件处理函数
  let index = 0
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
      
      // 获取点击到的对象
      const clickedObject = intersects[0].object
      
      // 检查对象名称是否为 "car"
      if (clickedObject.name === "car") {
        // 保存车辆对象引用
        carObject = clickedObject
        
        // 设置车辆初始位置为起始点
        if (!isCarMoving) {
          carObject.position.set(startPoint.x, startPoint.y, startPoint.z)
          moveProgress = 0
          currentTarget = 'end'
        }
        
        // 更新车辆信息
        carInfo.value.position.x = carObject.position.x
        carInfo.value.position.y = carObject.position.y
        carInfo.value.position.z = carObject.position.z
        
        // 显示弹框
        showCarModal.value = true
        
        // 开始坐标运动
        if (!isCarMoving) {
          isCarMoving = true
          animateCar()
          console.log(`车辆开始从起始点(${startPoint.x}, ${startPoint.y}, ${startPoint.z})向终点(${endPoint.x}, ${endPoint.y}, ${endPoint.z})移动`)
        } else {
          // 如果已经在运动，则停止运动
          isCarMoving = false
          if (carAnimationId) {
            cancelAnimationFrame(carAnimationId)
            carAnimationId = null
          }
          console.log('车辆停止运动')
        }
      }
    }
  }

  renderer.domElement.addEventListener('click', handleClick)
})

  // 在组件卸载时移除事件监听器和清理动画
onBeforeUnmount(() => {
  if (renderer && renderer.domElement) {
    renderer.domElement.removeEventListener('click', handleClick)
  }
  
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

/* 车辆信息弹框样式 */
.car-modal {
  position: fixed;
  top: 0;
  left: 0;
  width: 100%;
  height: 100%;
  background-color: rgba(0, 0, 0, 0.5);
  display: flex;
  justify-content: center;
  align-items: center;
  z-index: 1000;
}

.modal-content {
  background: white;
  border-radius: 8px;
  padding: 0;
  max-width: 400px;
  width: 90%;
  box-shadow: 0 4px 20px rgba(0, 0, 0, 0.3);
  animation: modalFadeIn 0.3s ease-out;
}

@keyframes modalFadeIn {
  from {
    opacity: 0;
    transform: scale(0.9);
  }
  to {
    opacity: 1;
    transform: scale(1);
  }
}

.modal-header {
  background: #2c3e50;
  color: white;
  padding: 15px 20px;
  border-radius: 8px 8px 0 0;
  display: flex;
  justify-content: space-between;
  align-items: center;
}

.modal-header h3 {
  margin: 0;
  font-size: 18px;
}

.close-btn {
  background: none;
  border: none;
  color: white;
  font-size: 24px;
  cursor: pointer;
  padding: 0;
  width: 30px;
  height: 30px;
  display: flex;
  align-items: center;
  justify-content: center;
  border-radius: 50%;
  transition: background-color 0.2s;
}

.close-btn:hover {
  background-color: rgba(255, 255, 255, 0.2);
}

.modal-body {
  padding: 20px;
}

.modal-body p {
  margin: 10px 0;
  font-size: 14px;
  line-height: 1.5;
}

.modal-body strong {
  color: #2c3e50;
  font-weight: 600;
}
</style>
