<script setup lang="ts">
import { ref, onMounted, onUnmounted, watch } from 'vue';
import * as THREE from 'three';
import { OrbitControls } from 'three/examples/jsm/controls/OrbitControls.js';

// 定义网格信息接口
interface GridInfo {
  cols: number;
  rows: number;
  xMin: number;
  xMax: number;
  yMin: number;
  yMax: number;
  nullValue: number;
  rotation: number;
  xStep: number;
  yStep: number;
  data: number[][];
}

// 场景相关变量
const scene = new THREE.Scene();
scene.background = new THREE.Color(0x000000);

const camera = new THREE.PerspectiveCamera(75, window.innerWidth / window.innerHeight, 0.1, 10000);
camera.position.set(0, 0, 500);

let renderer: THREE.WebGLRenderer;
let controls: any; // OrbitControls 类型
let currentMesh: THREE.Mesh | null = null;

// 响应式状态
const heightScaleFactor = ref(0.2);
const viewMode = ref('3d');
const statsInfo = ref('');
const infoText = ref('ZMAP 文件 3D 可视化');

// 高度标签
const maxHeightLabel = ref('最高: 0');
const height80Label = ref('80%: 0');
const height60Label = ref('60%: 0');
const height40Label = ref('40%: 0');
const height20Label = ref('20%: 0');
const minHeightLabel = ref('最低: 0');

// 初始化 Three.js 场景
function initThreeJS() {
  // 创建渲染器
  renderer = new THREE.WebGLRenderer({ antialias: true });
  renderer.setSize(window.innerWidth, window.innerHeight);
  document.getElementById('threeContainer')?.appendChild(renderer.domElement);
  
  // 创建控制器
  controls = new OrbitControls(camera, renderer.domElement);
  controls.enableDamping = true;
  controls.dampingFactor = 0.05;
  
  // 添加光源
  const ambientLight = new THREE.AmbientLight(0xffffff, 0.7);
  scene.add(ambientLight);
  
  const directionalLight = new THREE.DirectionalLight(0xffffff, 0.8);
  directionalLight.position.set(1, 1, 1);
  scene.add(directionalLight);
  
  // 开始动画循环
  animate();
  
  // 窗口大小调整处理
  window.addEventListener('resize', onWindowResize);
}

// 窗口大小调整处理函数
function onWindowResize() {
  camera.aspect = window.innerWidth / window.innerHeight;
  camera.updateProjectionMatrix();
  renderer.setSize(window.innerWidth, window.innerHeight);
}

// 动画循环
function animate() {
  requestAnimationFrame(animate);
  controls.update();
  
  // 动态调整等高线宽度：根据相机与模型中心的距离，按比例更新 uniform 值
  if (currentMesh) {
    const distance = camera.position.distanceTo(currentMesh.position);
    const refDistance = (currentMesh.userData.refDistance as number) || 1;
    const baseContourWidth = (currentMesh.userData.baseContourWidth as number) || 0.05;
    (currentMesh.material as THREE.ShaderMaterial).uniforms.contourWidth.value = 
      baseContourWidth * (distance / refDistance);
  }
  
  renderer.render(scene, camera);
}

// ZMAP 文件解析函数
function parseZMAPFile(content: string): GridInfo {
  const lines = content.split('\n');
  const cleanLines = lines.map(line => line.trim());
  
  let headerEndIndex: number | null = null;
  for (let i = 0; i < cleanLines.length; i++) {
    if (cleanLines[i] === '@') {
      headerEndIndex = i;
      break;
    }
  }
  if (headerEndIndex === null) {
    throw new Error("未找到 header 结束标记 '@'");
  }
  
  let gridInfoLine = '';
  for (let i = 0; i < headerEndIndex; i++) {
    if ((cleanLines[i] as string).startsWith('@Grid')) {
      gridInfoLine = cleanLines[i + 2];
      break;
    }
  }
  if (!gridInfoLine) {
    throw new Error("未找到网格信息行");
  }
  
  const gridInfoParts = gridInfoLine.split(',').map(s => s.trim()).filter(s => s !== '');
  if (gridInfoParts.length < 6) {
    throw new Error("网格信息不足，请检查文件格式！");
  }
  
  const gridInfo: GridInfo = {
    cols: parseInt(gridInfoParts[0]),
    rows: parseInt(gridInfoParts[1]),
    xMin: parseFloat(gridInfoParts[2]),
    xMax: parseFloat(gridInfoParts[3]),
    yMin: parseFloat(gridInfoParts[4]),
    yMax: parseFloat(gridInfoParts[5]),
    nullValue: 0.1e31,
    rotation: 0,
    xStep: 0,
    yStep: 0,
    data: []
  };
  
  gridInfo.xStep = (gridInfo.xMax - gridInfo.xMin) / (gridInfo.cols - 1);
  gridInfo.yStep = (gridInfo.yMax - gridInfo.yMin) / (gridInfo.rows - 1);
  
  const dataValues: number[] = [];
  for (let i = headerEndIndex + 1; i < cleanLines.length; i++) {
    const line = cleanLines[i];
    if (line === '') continue;
    const parts = line.split(/\s+/).filter(part => part !== '');
    for (const part of parts) {
      try {
        const value = parseFloat(part);
        dataValues.push(value);
      } catch (e) {
        console.warn("无法转换数据:", part);
      }
    }
  }
  
  const expectedCount = gridInfo.rows * gridInfo.cols;
  const actualCount = dataValues.length;
  console.log(`预期数据数量: ${expectedCount}, 实际数据数量: ${actualCount}`);
  if (actualCount !== expectedCount) {
    throw new Error(`数据数量与预期不符，预期 ${expectedCount}，实际 ${actualCount}`);
  }
  
  for (let r = 0; r < gridInfo.rows; r++) {
    gridInfo.data[r] = [];
    for (let c = 0; c < gridInfo.cols; c++) {
      const index = r * gridInfo.cols + c;
      gridInfo.data[r][c] = dataValues[index];
    }
  }
  
  return gridInfo;
}

// 创建3D地形网格
function createTerrainMesh(gridInfo: GridInfo): THREE.Mesh {
  const width = gridInfo.xMax - gridInfo.xMin;
  const height = gridInfo.yMax - gridInfo.yMin;
  
  const geometry = new THREE.PlaneGeometry(
    width, 
    height, 
    gridInfo.cols - 1, 
    gridInfo.rows - 1
  );
  
  const vertices = geometry.attributes.position.array;
  let minHeight = Infinity;
  let maxHeight = -Infinity;
  
  for (let r = 0; r < gridInfo.rows; r++) {
    for (let c = 0; c < gridInfo.cols; c++) {
      const value = gridInfo.data[r][c];
      if (value !== null && !isNaN(value) && Math.abs(value) < gridInfo.nullValue) {
        minHeight = Math.min(minHeight, value);
        maxHeight = Math.max(maxHeight, value);
      }
    }
  }
  
  statsInfo.value = `
    网格大小: ${gridInfo.rows} × ${gridInfo.cols}<br>
    X范围: ${gridInfo.xMin.toFixed(2)} - ${gridInfo.xMax.toFixed(2)}<br>
    Y范围: ${gridInfo.yMin.toFixed(2)} - ${gridInfo.yMax.toFixed(2)}<br>
    高度范围: ${minHeight.toFixed(2)} - ${maxHeight.toFixed(2)}<br>
    网格间距: ${gridInfo.xStep.toFixed(2)} × ${gridInfo.yStep.toFixed(2)}
  `;
  
  // 根据高度范围计算高度缩放，使高度变化更明显
  const heightScale = Math.max(width, height) / (maxHeight - minHeight) * heightScaleFactor.value;
  const baseHeight = 10;
  
  for (let i = 0, j = 0; i < vertices.length; i += 3, j++) {
    const col = j % (gridInfo.cols);
    const row = Math.floor(j / (gridInfo.cols));
    const invertedRow = gridInfo.rows - 1 - row;
    
    if (invertedRow >= 0 && invertedRow < gridInfo.rows && col >= 0 && col < gridInfo.cols) {
      const value = gridInfo.data[invertedRow][col];
      if (value !== null && !isNaN(value) && Math.abs(value) < gridInfo.nullValue) {
        vertices[i + 2] = (value - minHeight) * heightScale + baseHeight;
      } else {
        vertices[i + 2] = baseHeight;
      }
    } else {
      vertices[i + 2] = baseHeight;
    }
  }
  
  geometry.attributes.position.needsUpdate = true;
  geometry.computeVertexNormals();
  
  // 等高线参数
  const contourInterval = (maxHeight - minHeight) / 20;
  // 设置一个基础的等高线宽度，后续根据相机距离动态调整
  const baseContourWidth = 0.4;
  
  const material = new THREE.ShaderMaterial({
    uniforms: {
      contourInterval: { value: contourInterval },
      contourWidth: { value: baseContourWidth },
      minHeight: { value: minHeight },
      maxHeight: { value: maxHeight },
      heightScale: { value: heightScale },
      baseHeight: { value: baseHeight }
    },
    vertexShader: `
      varying float vHeight;
      uniform float minHeight;
      uniform float heightScale;
      uniform float baseHeight;
      
      void main() {
        vHeight = (position.z - baseHeight) / heightScale + minHeight;
        gl_Position = projectionMatrix * modelViewMatrix * vec4(position, 1.0);
      }
    `,
    fragmentShader: `
      uniform float contourInterval;
      uniform float contourWidth;
      uniform float minHeight;
      uniform float maxHeight;
      varying float vHeight;
      
      void main() {
        float normalizedHeight = (vHeight - minHeight) / (maxHeight - minHeight);
        float modHeight = mod(vHeight, contourInterval);
        
        if(modHeight < contourWidth || modHeight > contourInterval - contourWidth) {
          gl_FragColor = vec4(0.0, 0.0, 0.0, 1.0);
        } else {
          vec3 color;
          if (normalizedHeight < 0.2) {
            color = vec3(0.0, normalizedHeight * 2.5, 0.5 + normalizedHeight * 2.5);
          } else if (normalizedHeight < 0.4) {
            color = vec3(0.0, 0.5 + (normalizedHeight - 0.2) * 2.5, 1.0 - (normalizedHeight - 0.2) * 2.5);
          } else if (normalizedHeight < 0.6) {
            color = vec3((normalizedHeight - 0.4) * 5.0, 1.0, 0.0);
          } else if (normalizedHeight < 0.8) {
            color = vec3(1.0, 1.0 - (normalizedHeight - 0.6) * 2.5, 0.0);
          } else {
            color = vec3(1.0, 0.5 + (normalizedHeight - 0.8) * 2.5, (normalizedHeight - 0.8) * 5.0);
          }
          gl_FragColor = vec4(color, 1.0);
        }
      }
    `,
    side: THREE.DoubleSide
  });
  
  const mesh = new THREE.Mesh(geometry, material);
  mesh.rotation.x = -Math.PI / 2;
  mesh.position.set(0, 0, 0);
  
  // 将基础等高线宽度保存到 userData 中，后续动态调整时作为参考
  mesh.userData.baseContourWidth = baseContourWidth;
  
  return mesh;
}

// 处理ZMAP数据
function processZMAPData(content: string) {
  try {
    const gridInfo = parseZMAPFile(content);
    
    if (currentMesh) {
      scene.remove(currentMesh as unknown as THREE.Object3D);
      currentMesh.geometry.dispose();
      (currentMesh.material as THREE.Material).dispose();
    }
    
    // 创建新的地形网格
    currentMesh = createTerrainMesh(gridInfo);
    scene.add(currentMesh as unknown as THREE.Object3D);
    
    const width = gridInfo.xMax - gridInfo.xMin;
    const height = gridInfo.yMax - gridInfo.yMin;
    const maxDim = Math.max(width, height);
    
    // 保存一个参考距离到当前网格，后续用于动态调整等高线宽度
    currentMesh.userData.refDistance = maxDim;
    
    // 根据视图模式设置相机位置
    if (viewMode.value === 'top') {
      camera.position.set(0, 0, maxDim * 1.5);
      camera.lookAt(0, 0, 0);
    } else {
      camera.position.set(maxDim * 0.8, maxDim * 0.8, maxDim * 0.8);
      camera.lookAt(0, 0, 0);
    }
    camera.near = 0.1;
    camera.far = maxDim * 10;
    camera.updateProjectionMatrix();
    
    controls.target.set(0, 0, 0);
    controls.update();
    
    infoText.value = '已加载: ' + (gridInfo.rows * gridInfo.cols) + ' 个数据点';
    
    // 更新颜色卡标签
    let minHeight = Infinity;
    let maxHeight = -Infinity;
    
    for (let r = 0; r < gridInfo.rows; r++) {
      for (let c = 0; c < gridInfo.cols; c++) {
        const value = gridInfo.data[r][c];
        if (value !== null && !isNaN(value) && Math.abs(value) < gridInfo.nullValue) {
          minHeight = Math.min(minHeight, value);
          maxHeight = Math.max(maxHeight, value);
        }
      }
    }
    
    const heightRange = maxHeight - minHeight;
    maxHeightLabel.value = `最高: ${maxHeight.toFixed(1)}`;
    height80Label.value = `80%: ${(minHeight + heightRange * 0.8).toFixed(1)}`;
    height60Label.value = `60%: ${(minHeight + heightRange * 0.6).toFixed(1)}`;
    height40Label.value = `40%: ${(minHeight + heightRange * 0.4).toFixed(1)}`;
    height20Label.value = `20%: ${(minHeight + heightRange * 0.2).toFixed(1)}`;
    minHeightLabel.value = `最低: ${minHeight.toFixed(1)}`;
    
  } catch (error: any) {
    console.error('处理ZMAP数据失败:', error);
    alert('处理ZMAP数据失败: ' + error.message);
  }
}

// 加载默认数据
function loadDefaultData() {
  fetch('zmapdemo1.txt')
    .then(response => response.text())
    .then(content => {
      processZMAPData(content);
    })
    .catch(error => {
      console.error('加载默认数据失败:', error);
      alert('加载默认数据失败，请检查控制台获取详细信息。');
    });
}

// 处理文件上传
function handleFileUpload(event: Event) {
  const input = event.target as HTMLInputElement;
  const file = input.files?.[0];
  if (!file) return;
  
  const reader = new FileReader();
  reader.onload = function(e) {
    const content = e.target?.result as string;
    processZMAPData(content);
  };
  reader.readAsText(file);
}

// 监听高度缩放变化
watch(heightScaleFactor, () => {
  const fileInput = document.getElementById('zmapFile') as HTMLInputElement;
  if (currentMesh) {
    if (fileInput.files?.[0]) {
      const file = fileInput.files[0];
      const reader = new FileReader();
      reader.onload = function(e) {
        const content = e.target?.result as string;
        processZMAPData(content);
      };
      reader.readAsText(file);
    } else {
      loadDefaultData();
    }
  }
});

// 监听视图模式变化
watch(viewMode, () => {
  if (!currentMesh) return;
  
  const width = (currentMesh.geometry as THREE.PlaneGeometry).parameters.width || 1000;
  const height = (currentMesh.geometry as THREE.PlaneGeometry).parameters.height || 1000;
  const maxDim = Math.max(width, height);
  
  if (viewMode.value === 'top') {
    camera.position.set(0, 0, maxDim * 1.5);
    camera.lookAt(0, 0, 0);
  } else {
    camera.position.set(maxDim * 0.8, maxDim * 0.8, maxDim * 0.8);
    camera.lookAt(0, 0, 0);
  }
  controls.update();
});

// 组件挂载时初始化
onMounted(() => {
  initThreeJS();
});

// 组件卸载时清理
onUnmounted(() => {
  window.removeEventListener('resize', onWindowResize);
  if (renderer) {
    renderer.dispose();
  }
  if (currentMesh) {
    scene.remove(currentMesh as unknown as THREE.Object3D);
    currentMesh.geometry.dispose();
    (currentMesh.material as THREE.Material).dispose();
  }
});
</script>

<template>
  <header>
    <div class="wrapper">
      <div id="info">{{ infoText }}</div>
    </div>
  </header>

  <main>
    <!-- Three.js 容器 -->
    <div id="threeContainer"></div>
    
    <!-- 文件输入控件 -->
    <div id="fileInput">
      <input type="file" id="zmapFile" accept=".txt,.grd,.asc" @change="handleFileUpload">
      <button id="loadDefault" @click="loadDefaultData">加载默认数据</button>
    </div>
    
    <!-- 控制面板 -->
    <div id="controls">
      <div>
        <label for="heightScale">高度缩放: </label>
        <input type="range" id="heightScale" min="0.05" max="1" step="0.05" v-model="heightScaleFactor">
        <span id="heightScaleValue">{{ heightScaleFactor.toFixed(2) }}</span>
      </div>
      <div>
        <label for="viewMode">视图模式: </label>
        <select id="viewMode" v-model="viewMode">
          <option value="3d">3D 视图</option>
          <option value="top">俯视图</option>
        </select>
      </div>
    </div>
    
    <!-- 统计信息 -->
    <div id="stats" v-html="statsInfo"></div>
    
    <!-- 颜色图例 -->
    <div id="colorLegend"></div>
    <div id="colorLabels">
      <div id="maxHeightLabel">{{ maxHeightLabel }}</div>
      <div id="height80Label">{{ height80Label }}</div>
      <div id="height60Label">{{ height60Label }}</div>
      <div id="height40Label">{{ height40Label }}</div>
      <div id="height20Label">{{ height20Label }}</div>
      <div id="minHeightLabel">{{ minHeightLabel }}</div>
    </div>
  </main>
</template>

<style scoped>
/* 基础样式 */
body {
  margin: 0;
  overflow: hidden;
  font-family: Arial, sans-serif;
}

header {
  line-height: 1.5;
}

main {
  position: relative;
  width: 100%;
  height: calc(100vh - 60px);
}

/* Three.js 容器 */
#threeContainer {
  width: 100%;
  height: 100%;
}

/* 信息面板 */
#info {
  position: absolute;
  top: 10px;
  left: 10px;
  color: white;
  background-color: rgba(0, 0, 0, 0.5);
  padding: 10px;
  border-radius: 5px;
  pointer-events: none;
  z-index: 10;
}

/* 文件输入控件 */
#fileInput {
  position: absolute;
  top: 10px;
  right: 10px;
  padding: 10px;
  background-color: rgba(255, 255, 255, 0.8);
  border-radius: 5px;
  z-index: 10;
}

/* 统计信息 */
#stats {
  position: absolute;
  bottom: 10px;
  left: 10px;
  color: white;
  background-color: rgba(0, 0, 0, 0.5);
  padding: 10px;
  border-radius: 5px;
  z-index: 10;
}

/* 控制面板 */
#controls {
  position: absolute;
  top: 60px;
  right: 10px;
  padding: 10px;
  background-color: rgba(255, 255, 255, 0.8);
  border-radius: 5px;
  z-index: 10;
}

/* 颜色图例 */
#colorLegend {
  position: absolute;
  left: 10px;
  top: 50%;
  transform: translateY(-50%);
  width: 30px;
  height: 300px;
  background: linear-gradient(to top, 
    rgb(0, 0, 128),     /* 0% - 深蓝色 */
    rgb(0, 128, 255),   /* 20% - 浅蓝色 */
    rgb(0, 255, 128),   /* 40% - 青绿色 */
    rgb(128, 255, 0),   /* 60% - 黄绿色 */
    rgb(255, 128, 0),   /* 80% - 橙色 */
    rgb(255, 255, 128)  /* 100% - 浅黄色 */
  );
  border-radius: 5px;
  border: 1px solid white;
  z-index: 10;
}

/* 颜色标签 */
#colorLabels {
  position: absolute;
  left: 45px;
  top: 50%;
  transform: translateY(-50%);
  height: 300px;
  color: white;
  font-size: 12px;
  display: flex;
  flex-direction: column;
  justify-content: space-between;
  background-color: rgba(0, 0, 0, 0.5);
  padding: 5px;
  border-radius: 5px;
  z-index: 10;
}

/* 响应式布局 */
@media (min-width: 1024px) {
  header {
    display: flex;
    place-items: center;
    padding-right: calc(var(--section-gap) / 2);
  }

  header .wrapper {
    display: flex;
    place-items: flex-start;
    flex-wrap: wrap;
  }
}
</style>
