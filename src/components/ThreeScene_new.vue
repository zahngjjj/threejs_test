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
    '/chejian1213.glb',
    (gltf) => {
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
const handleMachineClick = (clickedObject) => {
  
 const obj = clickedObject
  
  console.log('进来了',obj)
  console.log('点击对象名称:', obj.name)
  
  // 只修改当前点击对象的颜色，不递归处理子对象
  if (obj && obj.material) {
    // 如果对象有材质，直接修改颜色
    if (obj.material.color) {
      obj.material.color.setHex(0x00ff00) // 设置为绿色
      console.log(`对象 ${obj.name} 的颜色已改为绿色`)
    }
    // 如果是材质数组，遍历所有材质
    else if (Array.isArray(obj.material)) {
      obj.material.forEach((material, index) => {
        if (material.color) {
          material.color.setHex(0x00ff00) // 设置为绿色
          console.log(`对象 ${obj.name} 的材质${index} 颜色已改为绿色`)
        }
      })
    }
  } else {
    console.log(`对象 ${obj.name} 没有材质或材质不支持颜色修改`)
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
</style>
