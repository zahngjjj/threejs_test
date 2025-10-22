<template>
  <div ref="container" class="content"></div>
  
  <!-- 机器详情弹框 -->
  <div v-if="showMachineDialog" class="machine-dialog-overlay" @click="closeMachineDialog">
    <div class="machine-dialog" @click.stop>
      <div class="dialog-header">
        <h3>机器详情</h3>
        <button class="close-btn" @click="closeMachineDialog">×</button>
      </div>
      <div class="dialog-content">
        <div class="machine-info">
          <h4>{{ selectedMachine.name }}</h4>
          <div class="info-row">
            <span class="label">机器编号:</span>
            <span class="value">{{ selectedMachine.id }}</span>
          </div>
          <div class="info-row">
            <span class="label">机器类型:</span>
            <span class="value">{{ selectedMachine.type }}</span>
          </div>
          <div class="info-row">
            <span class="label">运行状态:</span>
            <span class="value" :class="selectedMachine.status === '运行中' ? 'status-running' : 'status-stopped'">
              {{ selectedMachine.status }}
            </span>
          </div>
          <div class="info-row">
            <span class="label">温度:</span>
            <span class="value">{{ selectedMachine.temperature }}°C</span>
          </div>
          <div class="info-row">
            <span class="label">运行时长:</span>
            <span class="value">{{ selectedMachine.runtime }}</span>
          </div>
          <div class="info-row">
            <span class="label">最后维护:</span>
            <span class="value">{{ selectedMachine.lastMaintenance }}</span>
          </div>
        </div>
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

// 弹框相关的响应式变量
const showMachineDialog = ref(false)
const selectedMachine = ref({
  name: '',
  id: '',
  type: '',
  status: '',
  temperature: '',
  runtime: '',
  lastMaintenance: ''
})

// Three.js 相关变量
let scene, camera, renderer, controls, raycaster, mouse, mixer, clock

// 初始化场景
const initScene = () => {
  scene = new THREE.Scene()
  scene.background = new THREE.Color(0x0a1530)
  
  // 初始化时钟
  clock = new THREE.Clock()
  
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
  const ambientLight = new THREE.AmbientLight(0xffffff, 1)
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
    '/chache_donghua.glb',
    (gltf) => {
      const car_scene = gltf.scene;
      // 缩放整个场景到0.8倍
      car_scene.scale.set(0.3, 0.3, 0.3);
      // 将模型绕Y轴旋转30度（π/6弧度）,180度换算成弧度制是π
      car_scene.rotation.x = Math.PI / 12;
      scene.add(car_scene);
      mixer = new THREE.AnimationMixer(gltf.scene);
      const action = mixer.clipAction(gltf.animations[0]);
      action.play();
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


const handleGoodsClick = (obj) => {
  console.log('点击了货物对象:', obj.name, obj)
  
  // 定义一个函数来克隆材质并修改颜色
  const cloneAndChangeMaterialColor = (material, objName, materialIndex = '') => {
    if (material.color) {
      // 克隆材质以避免影响其他对象
      const clonedMaterial = material.clone()
      clonedMaterial.color.setHex(0x00ff00) // 设置为绿色
      console.log(`对象 ${objName} 的材质${materialIndex} 颜色已改为绿色`)
      return clonedMaterial
    }
    return null
  }
  
  // 定义一个函数来处理对象的材质
  const processObjectMaterial = (targetObj, objName) => {
    if (targetObj && targetObj.material) {
      // 如果是单个材质
      if (!Array.isArray(targetObj.material)) {
        const newMaterial = cloneAndChangeMaterialColor(targetObj.material, objName)
        if (newMaterial) {
          targetObj.material = newMaterial
          return true
        }
      }
      // 如果是材质数组，遍历所有材质
      else {
        let changed = false
        const newMaterials = targetObj.material.map((material, index) => {
          const newMaterial = cloneAndChangeMaterialColor(material, objName, `[${index}]`)
          if (newMaterial) {
            changed = true
            return newMaterial
          }
          return material
        })
        if (changed) {
          targetObj.material = newMaterials
        }
        return changed
      }
    }
    return false
  }
  
  let colorChanged = false
  
  // 情况1：对象本身有材质
  if (obj && obj.material) {
    colorChanged = processObjectMaterial(obj, obj.name)
  }
  // 情况2：材质在子对象中
  else if (obj && obj.children && obj.children.length > 0) {
    // 遍历所有子对象，寻找有材质的子对象
    obj.children.forEach((child, index) => {
      if (processObjectMaterial(child, `${obj.name}.children[${index}]`)) {
        colorChanged = true
      }
    })
  }
  
  if (!colorChanged) {
    console.log(`对象 ${obj.name} 没有找到可修改颜色的材质`)
  }
}
// 处理机器点击事件
const handleMachineClick = (clickedObject) => {
  const obj = clickedObject
  
  console.log('点击了机器:', obj.name)
  
  // 模拟机器详情数据
  const machineData = {
    name: `机器设备 ${obj.name}`,
    id: `ID-${Math.random().toString(36).substr(2, 8).toUpperCase()}`,
    type: '生产设备',
    status: Math.random() > 0.5 ? '运行中' : '停机',
    temperature: `${(Math.random() * 30 + 40).toFixed(1)}`,
    runtime: `${Math.floor(Math.random() * 24)}小时${Math.floor(Math.random() * 60)}分钟`,
    lastMaintenance: new Date(Date.now() - Math.random() * 30 * 24 * 60 * 60 * 1000).toLocaleDateString()
  }
  
  // 设置选中的机器信息
  selectedMachine.value = machineData
  
  // 显示弹框
  // showMachineDialog.value = true
  
  // 将点击的对象颜色改为绿色（保留原有功能）
  console.log(obj,'obj')
  console.log(obj.material,'material')
  handleGoodsClick(obj)
}

// 关闭机器详情弹框
const closeMachineDialog = () => {
  showMachineDialog.value = false
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
      if (targetObject.name.indexOf("3MFMeshParent") !== -1) {
        handleMachineClick(targetObject)
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
  
  // 更新动画混合器
  if (mixer && clock) {
    const delta = clock.getDelta()
    mixer.update(delta)
  }
  
  controls.update()
  renderer.render(scene, camera)
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
  

})
</script>

<style scoped>
.content {
  width: 100vw;
  height: 100vh;
}

/* 弹框样式 */
.machine-dialog-overlay {
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

.machine-dialog {
  background: white;
  border-radius: 8px;
  box-shadow: 0 4px 20px rgba(0, 0, 0, 0.3);
  width: 400px;
  max-width: 90vw;
  max-height: 80vh;
  overflow: hidden;
}

.dialog-header {
  background: #2c3e50;
  color: white;
  padding: 15px 20px;
  display: flex;
  justify-content: space-between;
  align-items: center;
}

.dialog-header h3 {
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

.dialog-content {
  padding: 20px;
}

.machine-info h4 {
  margin: 0 0 15px 0;
  color: #2c3e50;
  font-size: 16px;
  border-bottom: 2px solid #3498db;
  padding-bottom: 8px;
}

.info-row {
  display: flex;
  justify-content: space-between;
  align-items: center;
  padding: 8px 0;
  border-bottom: 1px solid #ecf0f1;
}

.info-row:last-child {
  border-bottom: none;
}

.label {
  font-weight: 600;
  color: #34495e;
  flex: 1;
}

.value {
  flex: 1;
  text-align: right;
  color: #2c3e50;
}

.status-running {
  color: #27ae60;
  font-weight: 600;
}

.status-stopped {
  color: #e74c3c;
  font-weight: 600;
}
</style>
